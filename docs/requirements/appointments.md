# Module — Appointments & Scheduling

## Objective
Provide slot-based appointment booking (public + staff) and appointment
lifecycle management, with a concurrency-safe guarantee that a slot can
never be double-booked. Doctor availability/clinic hours/holidays are now
owned by `clinic-config.md` — this module consumes that data rather than
owning it.

## Scope
- Public slot availability lookup and booking (unauthenticated).
- Staff-side booking, reschedule, cancel, check-in, no-show marking,
  consultation start/completion.
- Appointment status lifecycle.

## Dependencies
- `foundation.md`: shared infra.
- `staff-auth.md`: `Staff.id`/role, `requireRole` (RECEPTIONIST/ADMIN
  manage; DOCTOR own-only per the permission matrix).
- `patients.md`: `Patient.id`, patient lookup/search service. Public
  booking that doesn't match an existing patient creates a minimal
  `Patient` record (name + phone) via that module's create path.
- `clinic-config.md`: `DoctorAvailability`, `ClinicHoliday` — source data
  for slot generation. This module does not define or migrate those
  tables.

## Entities / Data Model
```prisma
enum AppointmentStatus {
  BOOKED
  CONFIRMED
  CHECKED_IN
  IN_CONSULTATION
  COMPLETED
  CANCELLED
  NO_SHOW
}

model Appointment {
  id           String            @id @default(uuid())
  patientId    String            @map("patient_id")
  doctorId     String            @map("doctor_id") // Staff.id
  slotStartAt  DateTime          @map("slot_start_at") // UTC
  slotEndAt    DateTime          @map("slot_end_at")
  status       AppointmentStatus @default(CONFIRMED)
  reason       String?
  createdBy    String?           @map("created_by") // Staff.id, null if public self-booked
  cancelledAt  DateTime?         @map("cancelled_at")
  cancelReason String?           @map("cancel_reason")
  createdAt    DateTime          @default(now()) @map("created_at")
  updatedAt    DateTime          @updatedAt @map("updated_at")

  @@unique([doctorId, slotStartAt])
  @@map("appointments")
}
```
The `@@unique([doctorId, slotStartAt])` constraint (as a partial index
excluding `CANCELLED` rows) is the concurrency mechanism — see below.
`status` defaults to `CONFIRMED` — see State Machines for why `BOOKED` is
not the default in the current (unresolved) confirmation-flow design.

## Relationships
`Appointment.patientId` → `Patient.id`. `Appointment.doctorId` →
`Staff.id`. `visits.md`'s `Visit.appointmentId` and `billing.md`'s
`Invoice.appointmentId` reference `Appointment.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/public/availability` | none | Available slots by doctor + date range (reads `clinic-config.md` data) |
| POST | `/api/v1/public/appointments` | none (rate-limited) | Public booking request |
| GET | `/api/v1/public/appointments/lookup` | none (rate-limited) | Look up own booking by phone + booking ref |
| GET | `/api/v1/appointments` | ADMIN/DOCTOR(own)/RECEPTIONIST/COMPOUNDER(read-only,today) | List/filter appointments |
| POST | `/api/v1/appointments` | ADMIN/RECEPTIONIST | Staff-created booking |
| PATCH | `/api/v1/appointments/:id/confirm` | ADMIN/RECEPTIONIST | Mark confirmed (see State Machines) |
| PATCH | `/api/v1/appointments/:id/reschedule` | ADMIN/RECEPTIONIST | Change slot |
| PATCH | `/api/v1/appointments/:id/cancel` | ADMIN/RECEPTIONIST/DOCTOR(own) | Cancel with reason |
| PATCH | `/api/v1/appointments/:id/check-in` | ADMIN/RECEPTIONIST | Mark checked-in |
| PATCH | `/api/v1/appointments/:id/start-consultation` | ADMIN/DOCTOR(own) | Mark in-consultation |
| PATCH | `/api/v1/appointments/:id/no-show` | ADMIN/RECEPTIONIST | Mark no-show |
| PATCH | `/api/v1/appointments/:id/complete` | ADMIN/DOCTOR(own) | Mark completed |

## Business Rules
- A slot is defined by `(doctorId, slotStartAt)`; only one non-cancelled
  appointment may occupy it (see Concurrency).
- Slots are generated from `clinic-config.md`'s `DoctorAvailability` minus
  `ClinicHoliday` dates minus already-booked slots — `GET
  /public/availability` computes this on the fly, it is not a
  pre-materialized table.
- Cancelling an appointment frees its slot immediately for rebooking.
- Rescheduling is implemented as atomically cancelling the old slot and
  creating the new one (same transaction) — never a bare update of
  `slotStartAt` that could collide with a mid-flight booking on the target
  slot.
- Cancellation window: appointments cannot be booked/rescheduled into a
  slot in the past. (A minimum-notice cancellation window, if the clinic
  wants one, is a business config value to confirm during implementation
  — not hardcoded speculatively here.)
- DOCTOR role can only act on appointments where `doctorId` matches their
  own `Staff.id` ("own only" per the permission matrix).
- COMPOUNDER gets read-only access to today's appointments only (to know
  what to prep), no write access.
- See [business-rules.md](../business-rules.md) for the full catalogue of
  scheduling-related rules, including the UNRESOLVED items below.

## State Machines
```
BOOKED → CONFIRMED → CHECKED_IN → IN_CONSULTATION → COMPLETED
CONFIRMED (created directly, staff/public) → CHECKED_IN → IN_CONSULTATION → COMPLETED
BOOKED/CONFIRMED/CHECKED_IN → CANCELLED
BOOKED/CONFIRMED → NO_SHOW
```
No transitions out of `COMPLETED`, `CANCELLED`, or `NO_SHOW`.

**UNRESOLVED (see [business-rules.md](../business-rules.md)):** whether
`BOOKED → CONFIRMED` is a real manual step (e.g. reception calls to
confirm) or purely a future-flow placeholder. Until the clinic decides,
the default behavior is that both public and staff-created appointments
are created directly in `CONFIRMED` (the `confirm` endpoint exists for
forward-compatibility but is a no-op path in practice until this is
resolved) — implementers must not silently invent a confirmation
mechanism (e.g. auto-SMS) beyond what `notifications.md` already defines.

## Security / Privacy / Compliance
- Public endpoints are unauthenticated but must be rate-limited per-IP
  (booking spam / slot-hoarding prevention) and validated strictly (phone
  format, no free-text injection into slot lookup).
- Public booking lookup (`/public/appointments/lookup`) requires both
  phone AND a booking reference — phone alone must not be enough to
  enumerate someone else's appointment.
- Public endpoints never return other patients' data — availability
  responses contain only open slots (times), never other patients' names
  or booking details.
- Appointment list/detail responses for staff show patient name/phone
  (needed operationally) but not clinical history — that stays behind
  `visits.md`'s own authorization.

## Transactions / Concurrency
**This is the module's core hard requirement.** Booking (public or staff)
and reschedule must:
1. Attempt to `INSERT` the `Appointment` row (or, for reschedule, insert
   the new row + soft-cancel the old row) inside a transaction.
2. Rely on the `@@unique([doctorId, slotStartAt])` constraint — on a
   unique-violation (Postgres `23505`), roll back and return `409
   Conflict` with a message indicating the slot was just taken.
3. Never pre-check availability and then insert as two separate
   non-atomic steps as the sole safety mechanism — the pre-check is only
   for UX (showing available slots); the unique constraint is what
   actually prevents the race:
```
Patient A ─┐
           ├── both attempt slotStartAt=10:30, doctorId=D1
Patient B ─┘
Only one INSERT succeeds; the other gets 23505 → 409 to that caller.
```
- Cancelled appointments must not block the unique constraint from
  allowing a new booking in the freed slot — either exclude
  `status = CANCELLED` from the unique index (partial unique index) or
  model cancellation as a delete-equivalent that doesn't collide. Decide
  and document whichever approach is used in `architecture-context.md`
  during implementation (Prisma supports partial indexes via raw SQL in
  the migration).

## Edge Cases
- Two simultaneous requests for the same slot → exactly one `201`, one
  `409`.
- Booking a slot outside `DoctorAvailability` hours or on a
  `ClinicHoliday` → `422`.
- Rescheduling to a slot that gets taken mid-request → `409`, original
  appointment remains unchanged (transaction rolled back).
- Public lookup with correct phone but wrong booking reference → `404`,
  not `403` (don't confirm the booking reference format is "close").

## Acceptance Criteria
- Concurrent booking attempts on the same slot: exactly one succeeds.
- Cancelling then immediately rebooking the same slot succeeds.
- DOCTOR cannot reschedule/cancel another doctor's appointment (`403`).
- Public availability endpoint never exposes another patient's identity.
- Rescheduling is atomic: a failed reschedule leaves the original
  appointment intact.
- Invalid state transitions (e.g. `COMPLETED → CHECKED_IN`) are rejected
  with `422`.

## Testing Requirements
- Integration: fire concurrent booking requests at the same slot (e.g.
  `Promise.all`) and assert exactly one `201` / one `409`.
- Integration: full status-transition matrix (valid/invalid transitions).
- Integration: role-based access sweep per the permission matrix.
- Unit: slot-generation logic (availability minus holidays minus booked).

## Out of Scope
- Multi-location scheduling.
- Recurring appointments.
- Waitlists.
- Ad-hoc doctor unavailability/leave blocking (UNRESOLVED, see
  business-rules.md — do not build speculatively).
- SMS/email confirmation sending (that's `notifications.md` — this module
  only emits the event/data notifications needs).

## Cross-Module Contracts
Published for later modules:
- `Appointment.id`, `Appointment.patientId`, `Appointment.doctorId`,
  `Appointment.status` — referenced by `visits.md` (`Visit.appointmentId`),
  `billing.md` (`Invoice.appointmentId`), and `notifications.md`
  (confirmation/reminder triggers).
- An appointment-created/-cancelled/-rescheduled event shape (structure
  TBD at implementation time, but must include `appointmentId`,
  `patientId`, `slotStartAt`, `status`) for Notifications to consume.

## Implementation Notes
- Generate availability slots in the API's configured clinic timezone but
  store/compare `slotStartAt` in UTC, consistent with
  `architecture-context.md`.
- Keep the reschedule endpoint's transaction short — no notification
  sending inside it (see Notifications module — that's triggered after
  commit).
- Implement after `clinic-config.md` — slot generation has a hard
  dependency on that module's data existing.
