# Module 7 — Notifications

## Objective
Notify patients about appointment lifecycle events (confirmation,
reminder, cancellation, reschedule) and staff about payment follow-ups,
through a provider-agnostic abstraction so a concrete channel (Email/SMS/
WhatsApp) can be swapped without touching the modules that trigger
notifications.

## Scope
- Notification abstraction (`NotificationProvider` interface).
- Notification templates.
- Notification log (what was sent, to whom, when, status).
- Triggers wired to Appointments (confirmation, reminder, cancellation,
  reschedule) and Billing (payment reminder for outstanding balances).
- At least one concrete provider implementation (choice deferred to
  implementation time — see MEMORY.md open question).

## Dependencies
- `foundation.md`, `staff-auth.md` (ADMIN manages templates).
- `appointments.md`: appointment created/cancelled/rescheduled event data
  (`appointmentId`, `patientId`, `slotStartAt`, `status`).
- `patients.md`: patient contact info (`phone`, `email`) via that
  module's lookup service — this module does not query the `patients`
  table directly.
- `billing.md`: `Invoice.id`, `.total`, `.amountPaid` for payment
  reminders.

## Entities / Data Model
```prisma
enum NotificationChannel {
  EMAIL
  SMS
  WHATSAPP
}

enum NotificationStatus {
  PENDING
  SENT
  FAILED
}

model NotificationTemplate {
  id       String              @id @default(uuid())
  key      String              @unique // e.g. "appointment_confirmation"
  channel  NotificationChannel
  subject  String?             // for EMAIL
  body     String              // supports {{placeholders}}
  isActive Boolean             @default(true) @map("is_active")

  @@map("notification_templates")
}

model NotificationLog {
  id           String              @id @default(uuid())
  templateKey  String              @map("template_key")
  channel      NotificationChannel
  recipient    String              // phone or email at send time
  patientId    String?             @map("patient_id")
  relatedType  String?             @map("related_type") // e.g. "appointment"
  relatedId    String?             @map("related_id")
  status       NotificationStatus  @default(PENDING)
  errorMessage String?             @map("error_message")
  sentAt       DateTime?           @map("sent_at")
  createdAt    DateTime            @default(now()) @map("created_at")

  @@map("notification_logs")
}
```

## Relationships
`NotificationLog.patientId` → `Patient.id` (nullable — a log entry must
still exist even if the patient record is later removed/soft-deleted).
`.relatedId` loosely references `Appointment.id` or `Invoice.id` depending
on `.relatedType` (no hard FK, since it's polymorphic).

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/notification-templates` | ADMIN | List templates |
| PATCH | `/api/v1/notification-templates/:id` | ADMIN | Edit template body/active flag |
| GET | `/api/v1/notification-logs` | ADMIN | View send history (paginated, filterable by patient/status) |
| POST | `/api/v1/notification-logs/:id/retry` | ADMIN | Retry a failed send |

There is no public endpoint in this module — all triggers are internal,
called from Appointments/Billing service code.

## Business Rules
- Notifications are triggered **after** the originating transaction
  commits (e.g., after an appointment booking transaction succeeds), never
  inside it — a notification failure must never roll back a booking or
  payment.
- If a template is inactive, its notification type is silently skipped
  (logged as such), not sent and not treated as an error.
- Failed sends are logged with `status=FAILED` and `errorMessage`, and can
  be retried manually by ADMIN; there is no automatic infinite retry loop
  in V1.

## State Machines
```
PENDING → SENT
PENDING → FAILED → (manual retry) → PENDING → SENT/FAILED
```

## Security / Privacy / Compliance
- `NotificationLog.recipient` (phone/email) is PII — same logging
  restriction as `patients.md`: never write full log contents/recipient
  into application debug logs beyond what's already stored structured in
  `NotificationLog` itself.
- Only ADMIN can view send history or edit templates (may contain
  clinic-identifying info, and send history reveals patient contact
  patterns).
- Provider credentials (API keys for the chosen Email/SMS/WhatsApp
  provider) are environment variables only, validated at boot alongside
  other config, never logged.

## Transactions / Concurrency
Not concurrency-sensitive. Sending should be dispatched asynchronously
relative to the triggering request (e.g., fire-and-log, don't make the
booking API wait on an SMS provider's round-trip) — exact mechanism
(in-process async call vs. a lightweight queue) is an implementation
decision; a distributed message broker is explicitly not justified for
this scale (see `project-overview.md` non-goals).

## Edge Cases
- Provider call throws/times out → logged as `FAILED`, triggering flow
  (booking/payment) still returns success to its caller.
- Patient has no phone/email on file matching the template's channel →
  skip and log as `FAILED` with a clear `errorMessage`, don't crash the
  trigger.

## Acceptance Criteria
- Booking an appointment results in a `NotificationLog` entry attempting
  a confirmation send.
- A notification failure does not affect the HTTP response of the
  triggering booking/payment request.
- ADMIN can view logs and retry a failed send; retry updates the log
  entry rather than creating a duplicate ambiguously.
- Non-ADMIN roles cannot access templates or logs.

## Testing Requirements
- Unit: template placeholder substitution.
- Integration: booking triggers a log entry; provider failure doesn't
  propagate to the booking response.
- Integration: retry transitions a `FAILED` log to `SENT`/`FAILED` again
  without creating a duplicate row.

## Out of Scope
- Two-way messaging (patients replying to SMS/WhatsApp).
- Marketing/bulk campaign sending.
- Choosing the concrete provider now — that decision is made when this
  module is actually implemented (see MEMORY.md open question); this doc
  only fixes the abstraction.

## Cross-Module Contracts
This module is a leaf consumer — it does not publish contracts other
modules depend on, beyond the fact that Appointments/Billing call its
`notify(templateKey, recipientPatientId, data)`-style service function
rather than any module talking to a provider SDK directly.

## Implementation Notes
Design the `NotificationProvider` interface first, implement one provider
(even a "log-only" no-op provider is acceptable for initial delivery if a
real provider isn't chosen yet) so Appointments/Billing integration can be
verified end-to-end without blocking on a vendor decision.
