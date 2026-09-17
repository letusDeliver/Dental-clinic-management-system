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

- Public clinic landing-page APIs (clinic info, services offered).
- Public appointment booking (no login required).
- Staff authentication & role-based access control.
- Patient management (identity, contact info).
- Dental/medical records (history, visits, clinical notes, diagnosis,
  attachments).
- Treatment plans & procedures (catalogue + per-patient treatments).
- Billing & invoicing (charges, discounts, payments, receipts).
- Inventory/supplies tracking.
- Notifications (appointment confirmations/reminders — abstraction now,
  provider integration later).
- Reporting & dashboard (appointments, revenue, workload, inventory usage).

## Key Workflows

- **Booking:** patient submits a booking request (name, phone, desired
  slot) → system checks availability → confirms or rejects with
  alternatives → reception can also book manually.
- **Visit:** reception checks in patient → doctor records clinical
  notes/diagnosis → treatment plan created/updated → billing generated.
- **Billing:** invoice generated from treatments/consultation → payments
  recorded (possibly partial) → outstanding balance tracked → receipt
  issued.
- **Inventory:** compounder logs stock usage/receipt → low-stock visibility
  for reordering.

## Business Rules (high level — detail lives in each module's requirement doc)

- Only one booking can hold a given doctor's time slot.
- Backend enforces all authorization; no client-side-only access control.
- Clinical record mutations and access to full medical history are
  audited.
- Financial history (invoices/payments) is corrected via adjustments, not
  by editing history in place.

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
