# Module 4 — Treatment Plans & Procedures

## Objective
Maintain a clinic-wide treatment catalogue and, separately, the specific
treatments assigned to individual patients, with pricing frozen at
assignment time and an auditable status history.

## Scope
- Treatment catalogue (definitions: name, description, default price).
- Patient treatments (a catalogue item applied to a specific patient
  during a specific visit, optionally also linked to the originating
  appointment).
- Treatment status lifecycle.

## Dependencies
- `foundation.md`, `staff-auth.md` (roles/authz).
- `patients.md`: `Patient.id`.
- `visits.md`: `Visit.id` — every `PatientTreatment` is anchored to a
  visit (see [business-rules.md](../business-rules.md): "Can a patient
  have treatment without a visit?" → No).
- `appointments.md`: `Appointment.id` (optional, denormalized convenience
  link alongside `visitId`).

## Entities / Data Model
```prisma
model TreatmentDefinition {
  id           String    @id @default(uuid())
  name         String
  description  String?
  defaultPrice Decimal   @map("default_price")
  isActive     Boolean   @default(true) @map("is_active")
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")

  @@map("treatment_definitions")
}

enum PatientTreatmentStatus {
  PLANNED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

model PatientTreatment {
  id                    String                 @id @default(uuid())
  patientId             String                 @map("patient_id")
  treatmentDefinitionId String                 @map("treatment_definition_id")
  visitId               String                 @map("visit_id")
  appointmentId         String?                @map("appointment_id")
  priceAtAssignment     Decimal                @map("price_at_assignment")
  status                PatientTreatmentStatus @default(PLANNED)
  notes                 String?
  doctorId              String                 @map("doctor_id") // Staff.id
  createdAt             DateTime               @default(now()) @map("created_at")
  updatedAt             DateTime               @updatedAt @map("updated_at")

  @@map("patient_treatments")
}
```
`priceAtAssignment` is copied from `TreatmentDefinition.defaultPrice` at
creation time and never recalculated from a live join — protects
historical invoices from retroactively changing if the catalogue price
changes later.

## Relationships
`PatientTreatment.patientId` → `Patient.id`; `.treatmentDefinitionId` →
`TreatmentDefinition.id`; `.visitId` → `Visit.id` (required); `.appointmentId`
→ `Appointment.id` (optional); `.doctorId` → `Staff.id`. Billing
references `PatientTreatment.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/treatment-definitions` | any authenticated staff | Browse catalogue |
| POST | `/api/v1/treatment-definitions` | ADMIN | Create catalogue item |
| PATCH | `/api/v1/treatment-definitions/:id` | ADMIN | Update catalogue item (price/active flag) |
| GET | `/api/v1/patients/:patientId/treatments` | ADMIN/DOCTOR/COMPOUNDER(read) | List a patient's treatments |
| POST | `/api/v1/patients/:patientId/treatments` | ADMIN/DOCTOR | Assign a treatment to a patient |
| PATCH | `/api/v1/patient-treatments/:id/status` | ADMIN/DOCTOR | Transition status |

## Business Rules
- Deactivating a `TreatmentDefinition` (`isActive=false`) hides it from
  new assignments but does not affect existing `PatientTreatment` rows.
- `priceAtAssignment` is immutable after creation; correcting a pricing
  mistake requires an ADMIN-only override endpoint/flow decided at
  implementation time and recorded in `MEMORY.md`, not a plain `PATCH` to
  price.
- Only the treatment's assigned `doctorId` or ADMIN can transition its
  status.

## State Machines
```
PLANNED → IN_PROGRESS → COMPLETED
PLANNED → CANCELLED
IN_PROGRESS → CANCELLED
```
No transitions out of `COMPLETED`/`CANCELLED`.

## Security / Privacy / Compliance
- Treatment notes/diagnosis-adjacent free text follows the same logging
  restriction as `patients.md` (never logged in plaintext).
- COMPOUNDER gets read-only access (to prepare materials) — no
  create/update rights.

## Transactions / Concurrency
Not concurrency-sensitive in the way Appointments is; a plain transaction
around "create PatientTreatment + copy price" is sufficient (single
statement in practice, transaction only needed if paired with other
writes).

## Edge Cases
- Assigning an inactive catalogue item → `422`.
- Status transition attempted from a terminal state → `422`.
- Deleting a catalogue item that has existing `PatientTreatment` rows is
  disallowed — deactivate instead (no hard delete).

## Acceptance Criteria
- Assigning a treatment freezes the current catalogue price onto the
  record.
- Changing a catalogue's `defaultPrice` afterward does not alter any
  existing `PatientTreatment.priceAtAssignment`.
- Invalid status transitions are rejected with `422`.
- COMPOUNDER cannot create/update, only read.

## Testing Requirements
- Unit: price-freezing logic.
- Integration: status transition matrix (valid/invalid).
- Integration: role-based access sweep.

## Out of Scope
- Treatment templates that bundle multiple catalogue items into a
  "package" (future idea, not V1).
- Multi-step procedure scheduling across multiple visits with dependency
  tracking (V1 treats each `PatientTreatment` as one unit with a status).

## Cross-Module Contracts
Published for later modules:
- `PatientTreatment.id`, `.priceAtAssignment`, `.patientId` — used by
  `billing.md` to build invoice line items.
- `TreatmentDefinition.id`/`.name` — used for billing line-item labels and
  reporting (treatment counts).

## Implementation Notes
None beyond the above — this module is comparatively simple; resist
adding scheduling/dependency complexity not requested.
