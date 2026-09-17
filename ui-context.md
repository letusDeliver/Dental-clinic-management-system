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
Needs (per role): today's appointments, outstanding tasks, quick stats
(from Reporting module). Role determines which widgets are populated —
backend should let the frontend ask "what can I see" rather than the
frontend hardcoding role logic; permission info can ride on the login
response (role + capability list).

## Patient Screens (staff-facing)
Needs: search/list patients, patient detail (demographics + history
summary), create/edit patient, view full clinical history (role-gated —
COMPOUNDER sees a reduced view). States: not-found, forbidden (403, when a
role tries to view restricted clinical detail), empty history for new
patients.

## Appointment Screens (staff-facing)
Needs: calendar/day view per doctor, create/reschedule/cancel, check-in,
mark no-show. States: past-slot editing should be rejected (422), double
-booking rejected (409), cancellation-window rule violations rejected
(422 with a clear reason code).

## Doctor Consultation Screen
Needs: patient history read, add clinical note/diagnosis, create/update
treatment plan for the visit. States: draft vs. finalized note (if
applicable), audit indicator not needed in UI but access itself is logged
server-side.

## Treatment Screens
Needs: treatment catalogue browse (for building a plan), patient treatment
list with status (planned/in-progress/completed), pricing shown from
catalogue at time of assignment (frozen price on the patient-treatment
record, not a live join, so historical invoices stay accurate).

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
