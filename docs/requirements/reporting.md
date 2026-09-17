# Module 8 — Reporting & Analytics

## Objective
Provide role-scoped, date-range-filterable operational reports over
existing data using efficient PostgreSQL queries — no analytics warehouse,
no ETL pipeline, appropriate for a single clinic's data volume.

## Scope
- Daily/range appointment counts (booked, completed, cancelled, no-show).
- Patient counts (new patients in range).
- Treatment counts (by definition, in range).
- Revenue (from Billing: invoiced, collected, outstanding).
- Outstanding payments listing.
- Cancellation / no-show rate.
- Doctor workload (appointments/treatments per doctor).
- Inventory usage (movements by item/category in range).

## Dependencies
- `foundation.md`, `staff-auth.md` (role-scoped access, see below).
- `appointments.md`: `Appointment` rows.
- `patients.md`: `Patient` rows (counts only, not clinical detail).
- `treatments.md`: `PatientTreatment` rows.
- `billing.md`: `Invoice`/`Payment` rows.
- `inventory.md`: `StockMovement` rows.

This module reads across other modules' tables for aggregation — that is
the one exception to "call the other module's service, don't query its
tables directly" (`architecture-context.md`), justified because reports
are inherently cross-cutting read-only aggregations. Reporting must still
never *write* to another module's tables.

## Entities / Data Model
No new persisted entities — this module is query-only against existing
tables. If a report proves too slow as a live query at real data volume,
introducing a materialized view or summary table is a Change Control
decision (see `ai-workflow-rules.md`), not a default.

## Relationships
N/A (read-only aggregation module).

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/reports/appointments` | ADMIN/RECEPTIONIST | Appointment counts by status, date range |
| GET | `/api/v1/reports/patients` | ADMIN | New patient counts, date range |
| GET | `/api/v1/reports/treatments` | ADMIN | Treatment counts by definition, date range |
| GET | `/api/v1/reports/revenue` | ADMIN | Invoiced/collected/outstanding totals, date range |
| GET | `/api/v1/reports/outstanding-payments` | ADMIN/RECEPTIONIST | List of invoices with balance due |
| GET | `/api/v1/reports/no-show-rate` | ADMIN/RECEPTIONIST | Cancellation/no-show rate, date range |
| GET | `/api/v1/reports/doctor-workload` | ADMIN (all doctors) / DOCTOR (self only) | Appointments/treatments per doctor |
| GET | `/api/v1/reports/inventory-usage` | ADMIN/COMPOUNDER | Stock movement summary by item/category |

All accept `?from=DATE&to=DATE` (default: current month if omitted).

## Business Rules
- Reports return **aggregated data only** — never raw per-patient rows
  with clinical detail (e.g., `reports/patients` returns counts/trends,
  not a patient list with medical notes).
- `DOCTOR` calling `doctor-workload` always gets their own data regardless
  of any `doctorId` query param they might pass — the backend derives
  scope from the authenticated identity, not from client-supplied
  parameters (same "never trust client-supplied IDs for authorization"
  rule as `code-standards.md`).
- `RECEPTIONIST` gets operational reports (appointments, outstanding
  payments, no-show rate) but not clinic-wide `revenue` or `patients`
  counts, per the permission matrix in `staff-auth.md`.
- `COMPOUNDER` gets only `inventory-usage`.

## State Machines
Not applicable.

## Security / Privacy / Compliance
- No clinical detail (notes/diagnosis) ever appears in any report
  response — enforced by construction (reports query aggregates/counts,
  not the columns containing that data).
- Outstanding-payments report includes patient name/phone (operationally
  necessary for follow-up) but not medical history.
- Access strictly per the matrix above; a role calling an endpoint outside
  its scope gets `403`.

## Transactions / Concurrency
Not applicable — read-only.

## Edge Cases
- Date range spanning no data → `200` with zero-valued aggregates, not an
  error.
- `from` after `to` → `400` validation error.
- Very large date ranges (e.g., multi-year) — acceptable to be slower in
  V1; do not pre-optimize with caching/materialized views until a real
  performance problem is observed (Change Control if it becomes one).

## Acceptance Criteria
- Each report endpoint returns correct aggregates against known seeded
  data for a fixed date range.
- `DOCTOR` cannot retrieve another doctor's workload via any parameter
  manipulation.
- `RECEPTIONIST`/`COMPOUNDER` receive `403` on out-of-scope report
  endpoints.
- No response from any report endpoint contains clinical notes/diagnosis
  text.

## Testing Requirements
- Integration: seed known data, assert each report's aggregate values
  match expected computation.
- Integration: role-based access sweep across all report endpoints.
- Integration: doctor-workload ignores a spoofed `doctorId` query param.

## Out of Scope
- Data warehouse / ETL / OLAP cube.
- Real-time streaming dashboards (polling a report endpoint is sufficient
  for V1).
- Predictive analytics / forecasting.
- Exporting to external BI tools.

## Cross-Module Contracts
This is the last module in the dependency graph — it publishes no
contracts for further modules to consume.

## Implementation Notes
Favor direct, indexed SQL aggregate queries (`GROUP BY`, `COUNT`, `SUM`)
via Prisma's raw query support where the query shape doesn't map cleanly
to Prisma's query builder. Add database indexes on date/status columns
used for filtering if a report is slow — that's a normal implementation
detail, not an architectural change.
