# Module — Clinic Configuration

## Objective
Own the clinic's identity/info (for the public landing page) and the
scheduling configuration (working hours, holidays, doctor availability)
that Appointments depends on for slot generation. Extracted as its own
module so this data has one clear owner instead of living inside
Appointments.

## Scope
- Clinic profile (name, address, contact info, description) for public
  display.
- Weekly working hours per doctor (`DoctorAvailability`).
- Clinic-wide holidays (`ClinicHoliday`).
- Treatments/services list surfaced on the public landing page (reads
  `TreatmentDefinition` from `treatments.md` — does not own it).

## Dependencies
- `foundation.md`, `staff-auth.md` (ADMIN manages config).
- `staff-auth.md`: `Staff.id` for `DoctorAvailability.doctorId` (role
  `DOCTOR`).

## Entities / Data Model
```prisma
model ClinicProfile {
  id          String  @id @default(uuid())
  name        String
  address     String?
  phone       String?
  email       String?
  description String?
  // singleton row — single clinic, see architecture-context.md

  @@map("clinic_profile")
}

model DoctorAvailability {
  id          String @id @default(uuid())
  doctorId    String @map("doctor_id") // Staff.id, role DOCTOR
  dayOfWeek   Int    @map("day_of_week") // 0=Sunday..6=Saturday
  startTime   String @map("start_time")  // "09:00"
  endTime     String @map("end_time")    // "17:00"
  slotMinutes Int    @default(30) @map("slot_minutes")

  @@map("doctor_availability")
}

model ClinicHoliday {
  id   String   @id @default(uuid())
  date DateTime
  note String?

  @@map("clinic_holidays")
}
```
`ClinicProfile` is a singleton (exactly one row) — enforced at the
application layer (upsert-only, no create-many endpoint), not by a DB
constraint, since Prisma has no native "max one row" constraint.

## Relationships
`DoctorAvailability.doctorId` → `Staff.id`. Consumed by `appointments.md`
for slot generation; consumed by the public landing endpoints for
"doctor available on X day" display.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/public/clinic-info` | none | Clinic profile + services list for landing page |
| GET | `/api/v1/clinic/profile` | ADMIN | Read profile (admin view) |
| PUT | `/api/v1/clinic/profile` | ADMIN | Update clinic profile (upsert the singleton) |
| GET | `/api/v1/clinic/availability` | ADMIN/DOCTOR(own)/RECEPTIONIST | List doctor availability rules |
| POST | `/api/v1/clinic/availability` | ADMIN | Create a doctor availability rule |
| PATCH | `/api/v1/clinic/availability/:id` | ADMIN | Update a rule |
| DELETE | `/api/v1/clinic/availability/:id` | ADMIN | Remove a rule |
| GET | `/api/v1/clinic/holidays` | ADMIN/RECEPTIONIST | List holidays |
| POST | `/api/v1/clinic/holidays` | ADMIN | Add a holiday |
| DELETE | `/api/v1/clinic/holidays/:id` | ADMIN | Remove a holiday |

## Business Rules
- Only ADMIN manages availability/holidays/profile — a DOCTOR cannot
  change their own hours directly in V1 (must go through ADMIN); this
  keeps schedule-of-record centralized. Flag as a candidate future
  self-service enhancement, not built now.
- Changing `DoctorAvailability` does not retroactively affect already-
  booked appointments outside the new hours — existing bookings stand;
  only future slot generation reflects the change.
- `ClinicHoliday` dates block slot generation for that whole day for all
  doctors (no per-doctor holiday concept in V1).

## State Machines
Not applicable.

## Security / Privacy / Compliance
No PII. `GET /public/clinic-info` is intentionally public and must expose
only clinic-level info — never doctor personal data beyond
name/specialty-if-any, never any patient data.

## Transactions / Concurrency
Not concurrency-sensitive (low-frequency admin writes).

## Edge Cases
- Overlapping `DoctorAvailability` rows for the same doctor/day — reject
  at creation (`422`) rather than silently generating overlapping slots.
- Holiday added for a date that already has booked appointments — adding
  the holiday does not cancel existing appointments; ADMIN must handle
  those manually (surfaced as a warning in the response, not an error).

## Acceptance Criteria
- Public clinic-info endpoint returns profile + active treatments list
  without authentication.
- Availability rules with overlapping time ranges for the same
  doctor/day are rejected.
- Slot generation in `appointments.md` correctly excludes holiday dates.

## Testing Requirements
- Unit: overlap-detection logic for availability rules.
- Integration: public clinic-info endpoint shape and no-auth access.
- Integration: holiday exclusion reflected in appointment availability
  queries (cross-module test with appointments.md).

## Out of Scope
- Per-doctor holidays / ad-hoc leave blocking (see
  [business-rules.md](../business-rules.md) UNRESOLVED item).
- Multiple clinic locations/branches.

## Cross-Module Contracts
Published for later modules:
- `DoctorAvailability`, `ClinicHoliday` — read by `appointments.md` for
  slot generation.
- `ClinicProfile` — read by the public landing page.

## Implementation Notes
This module should be implemented right after Staff & Auth, before
Appointments, since Appointments' slot generation has a hard dependency
on this data existing.
