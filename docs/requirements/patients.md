# Module 2 — Patients & Dental/Medical Records

## Objective
Manage patient identity, contact information, and dental/medical clinical
history (visits, notes, diagnosis, attachments) with strict access control,
since this module holds the system's most sensitive data.

## Scope
- Patient identity & contact profile (CRUD).
- Clinical history: visits, clinical notes, diagnosis entries.
- Attachments (X-rays, documents) referenced by storage key.
- Patient search/lookup for staff use (by name, phone, patient code).
- Patient history retrieval (full vs. reduced view by role).

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

model ClinicalVisit {
  id           String   @id @default(uuid())
  patientId    String   @map("patient_id")
  patient      Patient  @relation(fields: [patientId], references: [id])
  visitDate    DateTime @map("visit_date")
  doctorId     String   @map("doctor_id") // Staff.id, role DOCTOR
  notes        String?  // free-text clinical note
  diagnosis    String?
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")

  @@map("clinical_visits")
}

model Attachment {
  id              String   @id @default(uuid())
  patientId       String   @map("patient_id")
  visitId         String?  @map("visit_id")
  storageKey      String   @map("storage_key")
  fileName        String   @map("file_name")
  contentType     String   @map("content_type")
  sizeBytes       Int      @map("size_bytes")
  uploadedBy      String   @map("uploaded_by") // Staff.id
  createdAt       DateTime @default(now()) @map("created_at")

  @@map("attachments")
}
```

## Relationships
- `ClinicalVisit.doctorId` → `Staff.id` (must have role `DOCTOR`).
- `Appointments` (future module) will reference `Patient.id`.
- `Billing` (future module) will reference `Patient.id` and may reference
  `ClinicalVisit.id`.
- `Treatments` (future module) will reference `Patient.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/patients` | ADMIN/DOCTOR/RECEPTIONIST | Search/list patients (paginated, by name/phone/code) |
| POST | `/api/v1/patients` | ADMIN/RECEPTIONIST | Create patient |
| GET | `/api/v1/patients/:id` | ADMIN/DOCTOR/RECEPTIONIST/COMPOUNDER (reduced) | Patient profile |
| PATCH | `/api/v1/patients/:id` | ADMIN/RECEPTIONIST | Update contact info |
| GET | `/api/v1/patients/:id/history` | ADMIN/DOCTOR | Full clinical history |
| POST | `/api/v1/patients/:id/visits` | ADMIN/DOCTOR | Add a clinical visit/note |
| PATCH | `/api/v1/visits/:id` | ADMIN/DOCTOR (author or admin) | Amend a visit note (creates an audit trail entry, does not silently overwrite) |
| POST | `/api/v1/patients/:id/attachments` | ADMIN/DOCTOR | Upload attachment metadata (actual upload via signed URL, see Implementation Notes) |
| GET | `/api/v1/attachments/:id` | ADMIN/DOCTOR (or RECEPTIONIST if non-clinical doc) | Retrieve attachment (signed URL redirect) |

COMPOUNDER's "reduced view" on `GET /patients/:id` excludes `notes`,
`diagnosis`, and attachments — only demographics + today's scheduled
procedure type (once Appointments/Treatments exist), per the permission
matrix in `staff-auth.md`.

## Business Rules
- `patientCode` is system-generated (sequential/human-friendly), immutable
  once assigned.
- Patients are never hard-deleted — `deletedAt` (soft delete) only, and
  only by ADMIN, to preserve clinical/financial history integrity.
- A `ClinicalVisit` note, once created, is not destructively edited —
  amendments append an audit trail entry (who changed what, when) rather
  than overwriting silently, matching clinical record-keeping norms.
- Duplicate-patient prevention: creating a patient with an identical phone
  number prompts a "possible duplicate" warning in the response
  (`data.possibleDuplicates: [...]`) rather than blocking creation
  outright — reception makes the final call.

## State Machines
Not applicable at the entity level (no status field on Patient itself in
V1); `ClinicalVisit` has an implicit `created → amended` audit trail but
no formal status enum.

## Security / Privacy / Compliance
This module holds PII and PHI-equivalent clinical data. Explicit controls
(do not assume these are inferred):
- **Access control:** enforced exactly per the permission matrix in
  `staff-auth.md`. `GET /patients/:id/history` (full clinical history) is
  ADMIN/DOCTOR only — RECEPTIONIST and COMPOUNDER must receive `403`, not
  a filtered response, if they call it directly.
- **Audit requirement:** every read of `GET /patients/:id/history` and
  every write to `ClinicalVisit`/`Attachment` must create an `AuditLog`
  entry (`actorStaffId`, `action`, `entity_type`, `entity_id`, timestamp).
  This is mandatory, not optional instrumentation.
- **Logging restriction:** application logs must never contain
  `notes`, `diagnosis`, patient `phone`/`email`/`address`, or attachment
  file contents/names in plaintext log lines — log entity IDs only.
- **Data minimization:** list/search endpoints (`GET /patients`) return
  only `id, patientCode, firstName, lastName, phone` — not full clinical
  data — even to roles that could otherwise see the detail view.
- **Attachment security:** attachments are never served as raw public
  URLs. Access goes through an authenticated endpoint that verifies role
  + generates a short-lived signed URL to the storage backend. File type
  is validated (allow-list: images, PDF) and size-capped (define limit,
  e.g. 10MB, in Foundation-level config) before accepting an upload.
- **Retention:** no automatic deletion job in V1 — clinical records are
  retained indefinitely (soft-delete only), pending future legal/retention
  requirements which are explicitly out of scope for this module to
  decide unilaterally.

## Transactions / Concurrency
Patient creation + duplicate-check read is not required to be transactional
(duplicate check is advisory, not a hard constraint). Visit + attachment
creation in one request (if the API allows attaching a file at visit-creation
time) should be wrapped in a transaction so a partial write can't leave an
orphaned attachment record.

## Edge Cases
- Search with no matches → `200` with empty `data` array, not `404`.
- COMPOUNDER calling `GET /patients/:id/history` directly → `403`, not a
  silently filtered body.
- Amending a visit authored by a different doctor → allowed only for
  ADMIN or the original author; others get `403`.
- Uploading a disallowed file type/oversized file → `422` with a specific
  error code, not a generic `500`.

## Acceptance Criteria
- ADMIN/RECEPTIONIST can create and update patients; DOCTOR cannot create
  (read/history/visit-write only, per matrix) — confirm this exact split
  against the matrix before finalizing, since "create" wasn't listed for
  DOCTOR.
- COMPOUNDER receives a demographics-only view from `GET /patients/:id`
  and `403` from `/history`.
- Every call to `/patients/:id/history` produces exactly one `AuditLog`
  row.
- Attachment upload rejects disallowed MIME types and oversized files.
- Soft-deleted patients are excluded from search/list by default.

## Testing Requirements
- Integration: full permission-matrix sweep across all endpoints (each
  role × each endpoint → expected status).
- Integration: audit log row created on history read and on visit/
  attachment write.
- Unit: duplicate-phone detection logic.
- Unit: attachment MIME/size validation.

## Out of Scope
- Patient-facing login/portal.
- Automated retention/deletion policies.
- Full-text search across clinical notes (simple field search only in V1).
- E-signatures or formal consent-form workflows.

## Cross-Module Contracts
Published for later modules:
- `Patient.id`, `Patient.patientCode` — referenced by Appointments,
  Treatments, Billing.
- `ClinicalVisit.id` — may be referenced by Billing (invoice tied to a
  visit) and Treatments.
- A patient lookup capability (`findPatientById`, `searchPatients`)
  exported from this module's service layer for Appointments/Billing to
  call rather than querying the `patients` table directly.

## Implementation Notes
- Attachment upload flow: client requests a signed upload URL from this
  module, uploads directly to storage, then confirms with this module to
  create the `Attachment` metadata row — avoids proxying large files
  through the API process. Storage provider is the interface designed in
  `architecture-context.md`; concrete choice is decided during this
  module's implementation (see MEMORY.md open question).
