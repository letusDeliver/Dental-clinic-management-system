# Module 5 — Billing & Invoicing

## Objective
Generate invoices for patient consultations/treatments, record payments
(including partial payments), and track outstanding balances — with
financial history that is corrected via adjustments, never in-place edits.

## Scope
- Invoice generation from patient treatments/consultation charges.
- Discounts (per-line or invoice-level).
- Payment recording (supports partial payments, multiple payments per
  invoice).
- Outstanding balance tracking.
- Receipts.

## Dependencies
- `foundation.md`, `staff-auth.md` (roles/authz).
- `patients.md`: `Patient.id`.
- `appointments.md`: `Appointment.id` (optional link).
- `treatments.md`: `PatientTreatment.id`, `.priceAtAssignment` (source of
  invoice line items).
- `audit.md`: `recordAudit(...)` helper — invoice/payment writes are on
  the consolidated audit requirement list.

## Entities / Data Model
```prisma
enum InvoiceStatus {
  OPEN
  PARTIALLY_PAID
  PAID
  VOID
}

model Invoice {
  id             String        @id @default(uuid())
  patientId      String        @map("patient_id")
  appointmentId  String?       @map("appointment_id")
  status         InvoiceStatus @default(OPEN)
  subtotal       Decimal
  discountTotal  Decimal       @default(0) @map("discount_total")
  total          Decimal
  amountPaid     Decimal       @default(0) @map("amount_paid")
  createdBy      String        @map("created_by") // Staff.id
  createdAt      DateTime      @default(now()) @map("created_at")
  updatedAt      DateTime      @updatedAt @map("updated_at")

  @@map("invoices")
}

model InvoiceLineItem {
  id                 String   @id @default(uuid())
  invoiceId          String   @map("invoice_id")
  patientTreatmentId String?  @map("patient_treatment_id")
  description        String
  unitPrice          Decimal  @map("unit_price")
  quantity           Int      @default(1)
  discount           Decimal  @default(0)
  lineTotal          Decimal  @map("line_total")

  @@map("invoice_line_items")
}

enum PaymentMethod {
  CASH
  CARD
  UPI
  BANK_TRANSFER
  OTHER
}

model Payment {
  id         String        @id @default(uuid())
  invoiceId  String        @map("invoice_id")
  amount     Decimal
  method     PaymentMethod
  receivedBy String        @map("received_by") // Staff.id
  note       String?
  createdAt  DateTime      @default(now()) @map("created_at")

  @@map("payments")
}
```

## Relationships
`Invoice.patientId` → `Patient.id`; `.appointmentId` → `Appointment.id`;
`InvoiceLineItem.patientTreatmentId` → `PatientTreatment.id`;
`Payment.invoiceId` → `Invoice.id`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/patients/:patientId/invoices` | ADMIN/RECEPTIONIST | Create invoice from selected treatments/charges |
| GET | `/api/v1/invoices/:id` | ADMIN/RECEPTIONIST/DOCTOR(read) | Invoice detail with line items + payments |
| GET | `/api/v1/patients/:patientId/invoices` | ADMIN/RECEPTIONIST/DOCTOR(read) | List invoices for a patient |
| POST | `/api/v1/invoices/:id/payments` | ADMIN/RECEPTIONIST | Record a payment |
| POST | `/api/v1/invoices/:id/void` | ADMIN | Void an invoice (with reason) |
| GET | `/api/v1/invoices/:id/receipt` | ADMIN/RECEPTIONIST | Generate/view receipt |

## Business Rules
- `Invoice.total = subtotal - discountTotal`; `subtotal` is the sum of
  `InvoiceLineItem.lineTotal`.
- A payment cannot cause `amountPaid > total` (no overpayment) — reject
  with `422`.
- Invoice status is derived: `amountPaid = 0` → `OPEN`;
  `0 < amountPaid < total` → `PARTIALLY_PAID`; `amountPaid = total` →
  `PAID`. Status is recomputed server-side on every payment, never set
  directly by the client.
- Once `PAID` or `VOID`, no further payments are accepted.
- Correcting a mistake on an issued invoice is done by voiding it and
  issuing a new one (or, if partially implemented, an explicit credit
  line item) — line items are never edited/deleted after creation once a
  payment has been recorded against the invoice.
- DOCTOR has read-only access (needs to see a patient's billing status,
  cannot create/modify).

## State Machines
```
OPEN → PARTIALLY_PAID → PAID
OPEN → PAID (single full payment)
OPEN or PARTIALLY_PAID → VOID (ADMIN only, with reason)
```
No transitions out of `PAID`/`VOID`.

## Security / Privacy / Compliance
- Payment and invoice data is financial PII-adjacent — access restricted
  per the matrix above; never exposed on any public endpoint.
- Audit log entries required for invoice creation, payment recording, and
  voiding — see the consolidated list in [audit.md](audit.md).
- Receipts must not include other patients' data even indirectly (no
  cross-patient aggregate leakage in a single receipt document).

## Transactions / Concurrency
- Invoice creation (invoice row + all line items) is one transaction.
- Payment recording (insert `Payment` + recompute/update
  `Invoice.amountPaid` and `.status`) is one transaction to avoid a
  torn state under concurrent payment recording on the same invoice.

## Edge Cases
- Recording a payment that would overpay → `422`, invoice unchanged.
- Voiding an invoice that already has payments recorded → requires an
  explicit reason and is ADMIN-only; historical payments remain visible
  on the voided invoice for audit purposes (not deleted).
- Creating an invoice with zero line items → `422`.

## Acceptance Criteria
- Creating an invoice from treatments correctly sums line items into
  `subtotal`/`total`.
- Partial payments correctly update `amountPaid` and `status`.
- Overpayment attempts are rejected.
- Voided invoices reject further payments.
- DOCTOR can view but not create/modify invoices or payments.

## Testing Requirements
- Unit: invoice total/discount calculation, status-derivation logic.
- Integration: full payment lifecycle (open → partial → paid), overpayment
  rejection, void flow.
- Integration: role-based access sweep.

## Out of Scope
- Insurance claims processing (explicitly future scope — see
  `project-overview.md` non-goals).
- Online payment gateway integration — payments are recorded after the
  fact by staff, the system does not process card/UPI transactions
  itself.
- Multi-currency support.
- Tax calculation beyond a flat total (add only if a concrete requirement
  demands it).

## Cross-Module Contracts
Published for later modules:
- `Invoice.id`, `.status`, `.total`, `.amountPaid` — used by `reporting.md`
  for revenue/outstanding-balance reports.
- `Payment.id`, `.method`, `.amount` — used by reporting for payment-method
  breakdowns.

## Implementation Notes
- Use `Decimal` (Prisma `Decimal` type, backed by Postgres `numeric`) for
  all money fields — never floating point.
