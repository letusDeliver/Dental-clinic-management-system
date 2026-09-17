# Module — Visits, Diagnosis & Follow-ups

## Objective
Record what actually happened during a patient's clinical encounter —
symptoms, diagnosis, clinical notes, and follow-up plan — as the
authoritative clinical audit trail. Every treatment and prescription is
anchored to a visit.

## Scope
- Visit lifecycle (a clinical encounter, distinct from the scheduling
  concept of an `Appointment`).
- Diagnosis entries (structured, one-to-many per visit).
- Clinical notes / symptoms (free text).
- Follow-up scheduling data (date + instructions).

Out of this module's scope: appointments/slots (`appointments.md`),
actual treatments performed (`treatments.md`), prescriptions
(`prescriptions.md`), attachments (`attachments.md`) — a Visit links to
all of these but does not own them, keeping this module focused.

## Dependencies
- `foundation.md`, `staff-auth.md` (DOCTOR/ADMIN write; roles/authz).
- `patients.md`: `Patient.id`.
- `appointments.md`: `Appointment.id` (optional link — see
  [business-rules.md](../business-rules.md) "Can a visit exist without an
  appointment?").

## Entities / Data Model
```prisma
model Visit {
  id                 String    @id @default(uuid())
  patientId          String    @map("patient_id")
  appointmentId      String?   @map("appointment_id")
  doctorId           String    @map("doctor_id") // Staff.id, role DOCTOR
  visitDate          DateTime  @default(now()) @map("visit_date")
  symptoms           String?
  clinicalNotes      String?   @map("clinical_notes")
  followUpDate       DateTime? @map("follow_up_date")
  followUpNotes      String?   @map("follow_up_notes")
  createdAt          DateTime  @default(now()) @map("created_at")
  updatedAt          DateTime  @updatedAt @map("updated_at")

  @@map("visits")
}

model Diagnosis {
  id          String   @id @default(uuid())
  visitId     String   @map("visit_id")
  description String
  toothNumber String?  @map("tooth_number") // optional, dental-specific reference
  createdAt   DateTime @default(now()) @map("created_at")

  @@map("diagnoses")
}
```
Diagnosis is modeled as its own table (one-to-many per visit) rather than
a single free-text field, so a visit addressing multiple issues (e.g. two
different teeth) is represented structurally, which also supports future
reporting (diagnosis frequency by type) without a text-parsing hack.

## Relationships
`Visit.patientId` → `Patient.id`; `.appointmentId` → `Appointment.id`
(nullable); `.doctorId` → `Staff.id`. `Diagnosis.visitId` → `Visit.id`.
`treatments.md`'s `PatientTreatment.visitId` and `prescriptions.md`'s
`Prescription.visitId` both reference `Visit.id`.
`attachments.md`'s `Attachment.visitId` (optional) also references it.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/patients/:patientId/visits` | ADMIN/DOCTOR | List a patient's visits |
| POST | `/api/v1/patients/:patientId/visits` | ADMIN/DOCTOR | Create a visit (optionally from an appointment) |
| GET | `/api/v1/visits/:id` | ADMIN/DOCTOR | Visit detail incl. diagnoses |
| PATCH | `/api/v1/visits/:id` | ADMIN/DOCTOR (author or admin) | Amend notes/follow-up (audited, not silent overwrite) |
| POST | `/api/v1/visits/:id/diagnoses` | ADMIN/DOCTOR | Add a diagnosis entry |
| DELETE | `/api/v1/visits/:id/diagnoses/:diagnosisId` | ADMIN/DOCTOR (author or admin) | Remove an erroneous diagnosis entry (audited) |

This module has no RECEPTIONIST/COMPOUNDER write access at all, per the
permission matrix (staff-auth.md) and business-rules.md ("Can
receptionist modify medical records?" → No).

## Business Rules
- A visit is immutable in its core clinical facts once created in the
  sense that amendments are audit-trailed (see `patients.md`'s established
  pattern) — `PATCH` appends an audit entry rather than silently
  overwriting `clinicalNotes`.
- `Visit.appointmentId` is nullable to support walk-ins/emergencies
  (business-rules.md).
- Setting `followUpDate` is what `notifications.md` uses to trigger a
  follow-up reminder — no separate "FollowUp" entity is needed for V1.

## State Machines
No formal status enum on `Visit` in V1 — a visit exists once created;
there is no draft/finalized distinction (kept simple; add only if a
concrete workflow need appears — Change Control decision).

## Security / Privacy / Compliance
Same controls as established in `patients.md` for clinical data — applies
in full here since this module *is* the clinical record:
- Only ADMIN/DOCTOR may read or write.
- Every write (visit create/amend, diagnosis add/remove) produces an
  `AuditLog` entry (`audit.md`).
- No visit content (`symptoms`, `clinicalNotes`, `followUpNotes`,
  diagnosis `description`) may appear in application logs.

## Transactions / Concurrency
Creating a visit with initial diagnoses in one request is wrapped in a
transaction so a partial write can't leave the visit without its
diagnoses.

## Edge Cases
- Creating a visit referencing an `appointmentId` that belongs to a
  different patient → `422` (data integrity guard).
- RECEPTIONIST/COMPOUNDER attempting any write → `403`.
- Amending a visit authored by a different doctor → ADMIN or original
  author only, else `403` (mirrors the rule already established for
  clinical notes in `patients.md`).

## Acceptance Criteria
- Creating a visit from an appointment correctly links both records.
- A visit can exist with `appointmentId = null` (walk-in).
- Diagnosis entries are retrievable as part of visit detail.
- Every visit/diagnosis write produces exactly one `AuditLog` row.
- RECEPTIONIST/COMPOUNDER get `403` on every write endpoint in this
  module.

## Testing Requirements
- Integration: create visit with/without appointment link.
- Integration: diagnosis add/remove, including permission sweep.
- Integration: audit log row created per write.
- Unit: cross-patient appointment-link integrity check.

## Out of Scope
- Formal visit status/workflow states beyond existence (see State
  Machines above).
- Structured symptom coding (e.g. ICD-style codes) — free text only in
  V1.
- Automatic follow-up reminder scheduling logic itself (that's
  `notifications.md` — this module only stores the date/notes).

## Cross-Module Contracts
Published for later modules:
- `Visit.id`, `.patientId`, `.followUpDate` — used by `treatments.md`,
  `prescriptions.md`, `attachments.md`, and `notifications.md`.
- `Diagnosis` rows — may be read by `reporting.md`/`dashboard.md` for
  diagnosis-frequency views (aggregate only, never raw text on a public
  or under-scoped endpoint).

## Implementation Notes
Historical note: earlier planning had folded a lighter `ClinicalVisit`
model directly into `patients.md`. This module supersedes that — see
[MEMORY.md](../../MEMORY.md) for the decision record. `patients.md` no
longer owns visit/diagnosis data.
