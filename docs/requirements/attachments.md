# Module — Attachments

## Objective
Store and securely serve file attachments (X-rays, scanned documents,
photos) linked to a patient and, optionally, a specific visit — as its
own module so the storage-provider abstraction and access-control rules
have one home instead of being embedded in Patients.

## Scope
- Attachment metadata (file reference, owner links, uploader).
- Signed upload/download flow via a storage-provider abstraction.
- Access control for retrieval.

## Dependencies
- `foundation.md`, `staff-auth.md` (roles/authz).
- `patients.md`: `Patient.id`.
- `visits.md`: `Visit.id` (optional link).

## Entities / Data Model
```prisma
model Attachment {
  id          String   @id @default(uuid())
  patientId   String   @map("patient_id")
  visitId     String?  @map("visit_id")
  storageKey  String   @map("storage_key")
  fileName    String   @map("file_name")
  contentType String   @map("content_type")
  sizeBytes   Int      @map("size_bytes")
  uploadedBy  String   @map("uploaded_by") // Staff.id
  createdAt   DateTime @default(now()) @map("created_at")

  @@map("attachments")
}
```

## Relationships
`Attachment.patientId` → `Patient.id`; `.visitId` → `Visit.id` (nullable);
`.uploadedBy` → `Staff.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/attachments/upload-url` | ADMIN/DOCTOR | Request a short-lived signed upload URL |
| POST | `/api/v1/attachments` | ADMIN/DOCTOR | Confirm upload, create metadata row |
| GET | `/api/v1/patients/:patientId/attachments` | ADMIN/DOCTOR | List a patient's attachments |
| GET | `/api/v1/attachments/:id` | ADMIN/DOCTOR (or RECEPTIONIST for a non-clinical doc, see note) | Retrieve via short-lived signed download URL |
| DELETE | `/api/v1/attachments/:id` | ADMIN | Remove (metadata + underlying object) |

Note on RECEPTIONIST: if a future need arises for non-clinical documents
(e.g. a signed consent form) to be receptionist-visible, that requires an
explicit `category` field and a matrix update — not built speculatively
now. In V1, treat all attachments as clinical-tier (ADMIN/DOCTOR only).

## Business Rules
- Upload is a two-step flow: get a signed upload URL, upload directly to
  storage, then confirm with this module to create the metadata row —
  avoids proxying large files through the API process.
- Allowed content types: images (`image/jpeg`, `image/png`) and PDF
  only — reject anything else at both the signed-URL-request step and the
  confirm step.
- Max file size: 10 MB (configurable via env, validated in
  `foundation.md`'s config).
- Deleting an attachment removes both the metadata row and the
  underlying stored object (no orphaned files, no orphaned metadata).

## State Machines
Not applicable.

## Security / Privacy / Compliance
- Attachments are never served as raw public URLs — every retrieval goes
  through this module's authenticated endpoint, which verifies role and
  then generates a short-lived signed URL to the storage backend.
- File type and size are validated before accepting an upload
  confirmation, not just trusted from client-supplied metadata.
- Every attachment view/download and every upload/delete produces an
  `AuditLog` entry (`audit.md`) — attachments frequently contain the most
  sensitive clinical images (X-rays).
- Filenames/content are never written to application logs beyond the
  entity ID.

## Transactions / Concurrency
Not concurrency-sensitive; a single-row insert/delete per operation.

## Edge Cases
- Confirming an upload for a `storageKey` that was never actually
  uploaded to (client skipped the upload step) → validate object
  existence at confirm time if the storage provider supports a cheap
  existence check; otherwise document this as a known V1 trust boundary
  gap acceptable for a single small clinic (record in MEMORY.md if
  accepted as-is).
- Disallowed MIME type or oversized file → `422` at request-upload-URL
  time, not discovered only at confirm time.
- Deleting an attachment that a `Visit` or report references only by ID
  (no hard FK enforcement from Visit) → allowed; consumers must handle a
  missing attachment gracefully (`404` on `GET /attachments/:id`).

## Acceptance Criteria
- Upload flow (request URL → upload → confirm) produces a retrievable
  attachment.
- Disallowed file type/size rejected before storage.
- Every upload, download, and delete produces exactly one `AuditLog` row.
- RECEPTIONIST/COMPOUNDER receive `403` on all endpoints in V1.

## Testing Requirements
- Integration: full upload → confirm → retrieve → delete cycle.
- Unit: MIME/size validation.
- Integration: audit log row per operation.
- Integration: permission sweep.

## Out of Scope
- Client-side image processing/thumbnailing (return the original file;
  add derivatives only if a concrete UI need justifies it).
- Virus/malware scanning of uploads (flag as a future hardening item if
  the clinic's risk tolerance requires it — not built by default).
- Versioning of attachments (each upload is a new row; no in-place
  replace).

## Cross-Module Contracts
Published for later modules:
- `Attachment.id`, `.patientId`, `.visitId` — referenced by `patients.md`
  and `visits.md` detail views (list of attachment IDs/thumbnails).
- The `StorageProvider` interface (`put`/`get`/`delete` against a signed-
  URL-capable backend) — defined here, concrete implementation (local
  disk vs. S3-compatible) chosen at this module's implementation time
  (open question tracked in [MEMORY.md](../../MEMORY.md)).

## Implementation Notes
This module was originally sketched as embedded fields inside
`patients.md`; it has been split out so the storage abstraction and
access rules are unambiguous and reusable by `visits.md`. See
[MEMORY.md](../../MEMORY.md) for the decision record.
