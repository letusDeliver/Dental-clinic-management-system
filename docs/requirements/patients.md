# Module 2 — Patients

## Objective
Manage patient identity and contact information with strict access
control. This module owns only patient demographics/contact — clinical
history now lives in `visits.md`, and file attachments in
`attachments.md` (both split out; see [MEMORY.md](../../MEMORY.md) for
the decision record). Patients remains the sensitive-PII anchor that
those modules link back to.

## Scope
- Patient identity & contact profile (CRUD).
- Patient search/lookup for staff use (by name, phone, patient code).
- A patient's "profile" read composes a summary from `visits.md`/
  `attachments.md`/`treatments.md` (counts/links), but does not own that
  data.

## Dependencies
- `foundation.md`: shared infra, envelope, errors.
- `staff-auth.md`: `Staff.id`/`role`, `requireAuth`/`requireRole`
  middleware, the permission matrix (Patients rows).

## Entities / Data Model
```prisma
model Patient {
  id           String    @id @default(uuid())
  patientCode  String    @unique @map("patient_code") // human-friendly, e.g. P-000123
  firstName    String    @map("first_name")
  lastName     String    @map("last_name")
  phone        String
  email        String?
  dateOfBirth  DateTime? @map("date_of_birth")
  gender       String?
  address      String?
  createdBy    String    @map("created_by") // Staff.id
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  deletedAt    DateTime? @map("deleted_at")

  @@map("patients")
}
```

## Relationships
- `Appointment.patientId` (`appointments.md`), `Visit.patientId`
  (`visits.md`), `Attachment.patientId` (`attachments.md`),
  `PatientTreatment.patientId` (`treatments.md`), and
  `Invoice.patientId` (`billing.md`) all reference `Patient.id` — this
  module publishes the identity that everything else hangs off of.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/patients` | ADMIN/DOCTOR/RECEPTIONIST | Search/list patients (paginated, by name/phone/code) |
| POST | `/api/v1/patients` | ADMIN/RECEPTIONIST | Create patient |
| GET | `/api/v1/patients/:id` | ADMIN/DOCTOR/RECEPTIONIST/COMPOUNDER (reduced) | Patient profile |
| PATCH | `/api/v1/patients/:id` | ADMIN/RECEPTIONIST | Update contact info |
| GET | `/api/v1/patients/:id/summary` | ADMIN/DOCTOR | Cross-module summary: visit count, last visit date, open balance, attachment count (links only — full detail comes from each owning module's own endpoints) |

Full clinical history now lives at `GET /api/v1/patients/:patientId/visits`
(`visits.md`), and attachments at `GET
/api/v1/patients/:patientId/attachments` (`attachments.md`) — this module
no longer exposes `/history` or `/visits` directly.

COMPOUNDER's "reduced view" on `GET /patients/:id` excludes everything
from `/summary` — only demographics + today's scheduled procedure type
(once Appointments/Treatments exist), per the permission matrix in
`staff-auth.md`.

## Business Rules
- `patientCode` is system-generated (sequential/human-friendly), immutable
  once assigned.
- Patients are never hard-deleted — `deletedAt` (soft delete) only, and
  only by ADMIN, to preserve clinical/financial history integrity in the
  modules that reference `Patient.id`.
- Duplicate-patient prevention: creating a patient with an identical phone
  number prompts a "possible duplicate" warning in the response
  (`data.possibleDuplicates: [...]`) rather than blocking creation
  outright — reception makes the final call.

## State Machines
Not applicable — no status field on Patient in V1.

## Security / Privacy / Compliance
This module holds PII (contact/demographics) but no clinical free text —
that tier of sensitivity now belongs to `visits.md`/`prescriptions.md`.
Explicit controls (do not assume these are inferred):
- **Access control:** enforced exactly per the permission matrix in
  `staff-auth.md`.
- **Logging restriction:** application logs must never contain patient
  `phone`/`email`/`address` in plaintext log lines — log entity IDs only.
- **Data minimization:** list/search endpoints (`GET /patients`) return
  only `id, patientCode, firstName, lastName, phone` — not the full
  profile — even to roles that could otherwise see the detail view.
- **Retention:** no automatic deletion job in V1 — patient records are
  retained indefinitely (soft-delete only), pending future legal/retention
  requirements which are explicitly out of scope for this module to
  decide unilaterally.

## Transactions / Concurrency
Patient creation + duplicate-check read is not required to be
transactional (duplicate check is advisory, not a hard constraint).

## Edge Cases
- Search with no matches → `200` with empty `data` array, not `404`.
- COMPOUNDER calling `GET /patients/:id/summary` directly → `403`.

## Acceptance Criteria
- ADMIN/RECEPTIONIST can create and update patients.
- COMPOUNDER receives a demographics-only view from `GET /patients/:id`
  and `403` from `/summary`.
- Soft-deleted patients are excluded from search/list by default.
- `/summary` correctly aggregates counts from `visits.md`/`billing.md`/
  `attachments.md` without duplicating their detail data.

## Testing Requirements
- Integration: full permission-matrix sweep across all endpoints (each
  role × each endpoint → expected status).
- Unit: duplicate-phone detection logic.
- Integration: `/summary` aggregation against seeded cross-module data.

## Out of Scope
- Patient-facing login/portal.
- Automated retention/deletion policies.
- Clinical history, attachments, treatments, billing — each owned by its
  own module (see Objective).

## Cross-Module Contracts
Published for later modules:
- `Patient.id`, `Patient.patientCode` — referenced by Appointments,
  Visits, Treatments, Billing, Attachments.
- A patient lookup capability (`findPatientById`, `searchPatients`)
  exported from this module's service layer for other modules to call
  rather than querying the `patients` table directly.

## Implementation Notes
None beyond the above — this module is intentionally now a thin identity
module; resist re-absorbing clinical/attachment scope back into it.
