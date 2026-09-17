# ui-context.md — Frontend/Backend Contract Context

No frontend exists yet in this repository. This file documents what the
backend must provide for each expected screen/flow so future API design
stays usable by a real UI. Backend-focused; do not implement frontend
code from this file unless explicitly requested.

## Public Landing Page
Needs: clinic info (name, address, hours, services list, doctor profiles)
via a public read-only endpoint. No auth.

## Appointment Booking Flow
Needs: available slots for a date/doctor (public read), submit booking
(name, phone, optional email, desired slot, reason), confirmation
response, and a way to look up an existing booking (by phone + booking
reference, since there's no patient login). States to support:
- **Loading:** slot list fetch in progress.
- **Empty:** no slots available for the selected date/doctor.
- **Validation:** invalid phone/missing required field, surfaced per-field
  from the standard error envelope's `error.details`.
- **Conflict:** slot was taken between fetch and submit → `409`, UI should
  refetch availability.

## Staff Login
Needs: login (email/username + password) → access + refresh token;
refresh endpoint; logout (invalidate refresh token). States: invalid
credentials (401, generic message — do not reveal which field was wrong),
account locked/disabled (403).

## Staff Dashboard
Needs (per role, from `dashboard.md` — real-time "today" snapshot, not
historical): today's appointments, outstanding tasks, quick stats. Role
determines which widgets are populated (see `dashboard.md`'s per-role
table) — backend should let the frontend ask "what can I see" rather than
the frontend hardcoding role logic; permission info can ride on the login
response (role + capability list, `GET /auth/me`).

## Patient Screens (staff-facing)
Needs: search/list patients, patient detail (demographics only — from
`patients.md`), a cross-module summary widget (visit count, last visit,
open balance, attachment count via `GET /patients/:id/summary`). Full
clinical history is a separate screen backed by `visits.md`'s endpoints,
not this one. States: not-found, forbidden (403, when a role tries to
view restricted clinical detail), empty summary for new patients.

## Visit / Clinical History Screens (staff-facing, ADMIN/DOCTOR only)
Needs: list of a patient's visits (`visits.md`), visit detail with
diagnosis entries, create visit + add diagnosis, follow-up date/notes
display. States: forbidden (403) for RECEPTIONIST/COMPOUNDER attempting
any access, empty state for a patient with no visits yet, amendment
history shown as an append-only trail rather than an overwritten field.

## Appointment Screens (staff-facing)
Needs: calendar/day view per doctor, create/confirm/reschedule/cancel,
check-in, start-consultation, mark no-show/completed — the full state
machine from `appointments.md` (`BOOKED → CONFIRMED → CHECKED_IN →
IN_CONSULTATION → COMPLETED`). States: past-slot editing should be
rejected (422), double-booking rejected (409), invalid state transitions
rejected (422 with a clear reason code).

## Doctor Consultation Screen
Needs: patient visit history read (`visits.md`), add clinical
note/diagnosis, create/update treatment plan for the visit
(`treatments.md`), issue a prescription (`prescriptions.md`), set a
follow-up date. States: audit indicator not needed in UI but access
itself is logged server-side; empty state for a fresh visit before any
diagnosis is added.

## Prescription Screen
Needs: add prescription line items (medicine, dosage, frequency,
duration, instructions) during/after a visit, printable view
(`GET /prescriptions/:id/print`). States: at-least-one-item validation
(422 if empty), forbidden for RECEPTIONIST/COMPOUNDER.

## Treatment Screens
Needs: treatment catalogue browse (for building a plan), patient treatment
list with status (planned/in-progress/completed), pricing shown from
catalogue at time of assignment (frozen price on the patient-treatment
record, not a live join, so historical invoices stay accurate). Every
patient treatment is created in the context of a visit — the UI should
not offer "add treatment" outside an open visit screen.

## Attachment Screens
Needs: upload X-ray/document (two-step: request signed URL, upload, then
confirm), thumbnail/list view on patient and visit detail screens,
authenticated download (never a raw public link). States: disallowed file
type/oversized file rejected before upload starts (422), forbidden for
RECEPTIONIST/COMPOUNDER.

## Clinic Configuration Screens (ADMIN only)
Needs: edit clinic profile (name/address/contact), manage doctor weekly
availability, manage holiday list. States: overlap-conflict rejection
(422) when availability rules collide.

## Audit Log Screen (ADMIN only)
Needs: filterable/paginated log view (by actor, entity type/id, date
range). Read-only — no edit/delete UI should ever be built against this
data, since the API has no such endpoints.

## Billing Screens
Needs: generate invoice from a visit's treatments, add discount, record
payment (method + amount, supports partial), view outstanding balance,
print/view receipt. States: overpayment attempt (422), invoice already
fully paid (no further payment allowed), partial payment shown clearly.

## Inventory Screens
Needs: item list with current stock level, log stock in/out movement, low
-stock indicator, supplier reference. States: negative-stock prevention
(422), expiry warning if expiry tracking is in scope for an item.

## Reports Screens
Needs: date-range-filterable reports (appointments, revenue, outstanding
payments, cancellations/no-shows, doctor workload, inventory usage), each
scoped by role (a Doctor sees only their own workload, not clinic-wide
revenue). Backend returns aggregated data, not raw rows, for report
endpoints.

## Cross-Cutting UI States
- **Loading:** every list/detail endpoint should be fast enough for a
  simple spinner; no long-poll patterns needed for V1.
- **Empty:** distinguish "no data yet" from "no results for this filter"
  in list responses (`meta.total === 0` is enough; no special flag needed).
- **Validation:** always via the standard error envelope with
  `error.details` keyed by field where possible.
- **Error:** unexpected failures return `500` with a generic message; the
  UI should never need to parse a stack trace.
