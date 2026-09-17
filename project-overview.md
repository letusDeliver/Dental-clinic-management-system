# project-overview.md — Product Overview

## What This Is

A backend system for a **single dental clinic**: public-facing appointment
booking, staff operations (patients, records, treatments, billing,
inventory), and reporting. Not a SaaS product for multiple clinics.

## Users

- **Patient** — interacts only through the public booking surface; no
  login/account in V1.
- **Admin** — full access; manages staff, oversees all data, reporting.
- **Dentist (Doctor)** — clinical records, treatment plans, own schedule.
- **Receptionist** — booking, patient intake, billing, front-desk ops.
- **Compounder** — assists with treatment prep, inventory/supplies.

## Major Capabilities

- Public clinic landing-page APIs (clinic info, services offered, clinic
  hours).
- Public appointment booking (no login required).
- Staff authentication, staff management & role-based access control.
- Clinic configuration (working hours, holidays, doctor availability).
- Patient management (identity, contact info).
- Visits: symptoms, clinical notes, diagnosis, follow-up scheduling.
- Prescriptions (medicine, dosage, frequency, duration, instructions).
- Attachments (X-rays, scanned documents) with signed-URL access control.
- Treatment plans & procedures (catalogue + per-patient treatments,
  anchored to a visit).
- Billing & invoicing (charges, discounts, partial/full payments,
  receipts).
- Audit logs (ADMIN-facing, covering every sensitive read/write across
  the system).
- Inventory/supplies tracking (lower priority / future — see Non-Goals
  note below).
- Notifications (appointment confirmations/reminders, follow-up
  reminders — abstraction now, provider integration later).
- Dashboard (real-time, role-scoped "today" view) and Reporting
  (historical, date-range analytics) — kept as two distinct modules, see
  [architecture-context.md](architecture-context.md).

Module ownership detail for each capability lives in
`docs/requirements/*.md`; cross-cutting domain rules (what's allowed,
what's ambiguous) live in [docs/business-rules.md](docs/business-rules.md).

## Key Workflows

- **Public Booking:** patient selects date → views available slots →
  selects a slot → enters details → system finds/creates the patient
  record → reserves the slot (concurrency-safe) → confirmation.
- **Reception:** appointment → check-in → patient queue → handed to
  doctor.
- **Doctor Consultation:** appointment → visit created → diagnosis
  recorded → treatment(s) assigned → prescription issued (optional) →
  follow-up scheduled (optional) → appointment marked completed.
- **Billing:** invoice generated from a visit's treatments/consultation →
  payments recorded (possibly partial, multiple methods) → outstanding
  balance tracked → receipt issued.
- **Inventory** (lower priority / future): compounder logs stock
  usage/receipt → low-stock visibility for reordering.

See each workflow's owning module doc for the exact state machine and
edge cases; see [docs/business-rules.md](docs/business-rules.md) for
ambiguous points not yet resolved (e.g. whether appointment confirmation
is a manual step).

## Business Rules (high level — full catalogue in docs/business-rules.md)

- Only one booking can hold a given doctor's time slot.
- Backend enforces all authorization; no client-side-only access control.
- Every treatment and prescription is anchored to a visit — neither can
  exist without one.
- Clinical record mutations and access to full clinical history are
  audited (see `docs/requirements/audit.md`).
- Financial history (invoices/payments) is corrected via adjustments, not
  by editing history in place.
- Nothing clinical or financial is ever hard-deleted.

Detailed, per-question answers (including explicitly UNRESOLVED items
awaiting clinic input) live in
[docs/business-rules.md](docs/business-rules.md) — do not re-answer a
question there is already tracked.

## Non-Goals (V1 — explicitly excluded)

- Multi-clinic / multi-tenancy.
- Patient portal / patient login & accounts.
- Insurance claims integration.
- Online payment gateway integration (payments are recorded by staff,
  not processed by the system).
- Advanced AI diagnosis features.
- Complex pharmacy management.
- Hospital-scale workflows (multi-department, multi-branch, referrals).

These may be documented as future ideas elsewhere but must not enter
current module requirement docs without an explicit scope decision
recorded in [MEMORY.md](MEMORY.md).

**Note on Inventory & Reporting:** these have requirement docs
(`docs/requirements/inventory.md`, `reporting.md`) from the first
planning pass, but the expanded master prompt's explicit staff-app
feature list (§2) does not mention them — `dashboard.md` covers the
real-time "today" view that overlaps somewhat with Reporting's remit.
They are not cut from scope, just deprioritized to the end of the build
order (see `progress-tracker.md`) until the clinic confirms they're
needed for V1.
