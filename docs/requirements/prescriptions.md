# Module — Prescriptions

## Objective
Let a doctor record medication prescribed during a visit, structured
enough to print/export as a prescription slip.

## Scope
- Prescription (one per visit, or occasionally more if reissued).
- Prescription line items (medicine, dosage, frequency, duration,
  instructions).

## Dependencies
- `foundation.md`, `staff-auth.md` (DOCTOR/ADMIN write; roles/authz).
- `visits.md`: `Visit.id`, `Visit.patientId`.

## Entities / Data Model
```prisma
model Prescription {
  id        String              @id @default(uuid())
  visitId   String              @map("visit_id")
  doctorId  String              @map("doctor_id") // Staff.id, role DOCTOR
  notes     String?             // general instructions
  createdAt DateTime            @default(now()) @map("created_at")
  updatedAt DateTime            @updatedAt @map("updated_at")

  @@map("prescriptions")
}

model PrescriptionItem {
  id             String @id @default(uuid())
  prescriptionId String @map("prescription_id")
  medicineName   String @map("medicine_name")
  dosage         String // e.g. "500mg"
  frequency      String // e.g. "twice daily"
  durationDays   Int    @map("duration_days")
  instructions   String? // e.g. "after food"

  @@map("prescription_items")
}
```

## Relationships
`Prescription.visitId` → `Visit.id`; `.doctorId` → `Staff.id`.
`PrescriptionItem.prescriptionId` → `Prescription.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/visits/:visitId/prescriptions` | ADMIN/DOCTOR | List prescriptions for a visit |
| POST | `/api/v1/visits/:visitId/prescriptions` | ADMIN/DOCTOR | Create a prescription with line items |
| GET | `/api/v1/prescriptions/:id` | ADMIN/DOCTOR | Prescription detail |
| GET | `/api/v1/prescriptions/:id/print` | ADMIN/DOCTOR | Printable/exportable representation |
| PATCH | `/api/v1/prescriptions/:id` | ADMIN/DOCTOR (author or admin) | Amend (audited, not silent overwrite) |

RECEPTIONIST/COMPOUNDER have no access to this module — prescriptions are
clinical data, same tier of sensitivity as `visits.md`.

## Business Rules
- A prescription always belongs to exactly one visit; there is no
  "standalone" prescription unattached to a clinical encounter.
- At least one `PrescriptionItem` is required to create a `Prescription`
  (`422` if empty).
- Amending an issued prescription is audit-trailed, consistent with the
  pattern established for visit notes — not a silent overwrite, since a
  prescription is a document that may already have been handed to the
  patient.

## State Machines
Not applicable — no status field in V1.

## Security / Privacy / Compliance
Same tier as `visits.md`: ADMIN/DOCTOR only, every write audited
(`audit.md`), medicine/dosage content never written to application logs.

## Transactions / Concurrency
Creating a prescription with its line items is one transaction.

## Edge Cases
- Creating a prescription for a visit belonging to a different doctor
  (not the caller, not admin) → `403`.
- Empty line-item list → `422`.

## Acceptance Criteria
- Creating a prescription with items persists correctly and is
  retrievable via visit detail and its own endpoint.
- RECEPTIONIST/COMPOUNDER receive `403` on every endpoint.
- Every write produces an `AuditLog` entry.

## Testing Requirements
- Integration: create/amend flow, permission sweep, empty-items
  rejection.
- Integration: audit log row per write.

## Out of Scope
- Drug-interaction checking or a medicine master catalogue (free-text
  medicine name in V1 — a structured drug database is a future
  enhancement, not justified for a single small clinic now).
- E-prescription submission to pharmacies.
- Print/PDF rendering implementation detail beyond returning structured
  data at `/print` — actual PDF generation library choice is an
  implementation detail decided when this module is built.

## Cross-Module Contracts
This module is a leaf — it does not publish contracts other modules
depend on, beyond `Prescription.id` potentially appearing in
`reporting.md`/`dashboard.md` aggregate counts (prescriptions issued),
which read counts only, never medicine detail, on any broadly-scoped
endpoint.

## Implementation Notes
Keep `medicineName`/`dosage`/`frequency` as plain strings in V1 — resist
the temptation to build a medicine catalogue/autocomplete unless the
clinic asks for it (see Decision Discipline in `architecture-context.md`).
