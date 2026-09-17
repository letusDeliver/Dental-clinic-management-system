# Module — Dashboard

## Objective
Give each staff role a fast, role-scoped "what's happening right now /
today" view when they log in — distinct from `reporting.md`, which
handles historical, date-range analytical reports. Dashboard is real-time
and today-focused; Reporting is historical and range-based.

## Scope
- One role-scoped dashboard endpoint composing live counts/lists from
  other modules.
- No new persisted entities — pure read-composition, like
  `reporting.md`.

## Dependencies
- `foundation.md`, `staff-auth.md` (role-scoped response).
- `appointments.md`: today's schedule.
- `visits.md`: pending consultations.
- `billing.md`: today's collections, outstanding balances.
- `inventory.md`: low-stock alerts (if inventory is implemented; degrade
  gracefully — omit the widget — if it is not yet built, since Inventory
  is a lower-priority module per [project-overview.md](../project-overview.md)).

## Entities / Data Model
None — read-only composition over existing tables, same posture as
`reporting.md`.

## Relationships
N/A.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/dashboard` | any authenticated staff | Role-scoped "today" snapshot |

Response shape varies by caller's role (server decides which widgets to
populate — the frontend should not need to hardcode role logic, per
[ui-context.md](../ui-context.md)):
- **ADMIN:** today's appointment count by status, today's revenue
  collected, outstanding balance total, low-stock item count.
- **DOCTOR:** own today's schedule, count of pending (not yet
  `COMPLETED`) consultations.
- **RECEPTIONIST:** today's appointment queue (with status), today's
  check-ins pending, outstanding-payment count.
- **COMPOUNDER:** today's scheduled procedures (for prep), low-stock item
  count.

## Business Rules
- Every widget's scope follows the same role rules already defined
  elsewhere (DOCTOR never sees clinic-wide revenue, RECEPTIONIST never
  sees another doctor's private notes, etc.) — this module composes
  existing authorized views, it does not introduce new access rules.
- "Today" is computed in the clinic's configured timezone
  (`architecture-context.md`), not UTC midnight.

## State Machines
Not applicable.

## Security / Privacy / Compliance
No clinical detail (notes/diagnosis/prescription content) appears in any
dashboard widget — counts and minimal identifying info only (patient
name, appointment time), consistent with `reporting.md`'s "aggregated
data only" rule.

## Transactions / Concurrency
Not applicable — read-only.

## Edge Cases
- A role with no relevant data for a widget (e.g. a brand-new DOCTOR with
  no appointments today) → widget returns an empty/zero state, not an
  error.
- Inventory module not yet implemented → omit that widget entirely
  rather than erroring (see Dependencies note).

## Acceptance Criteria
- Each role receives only the widgets defined for it above.
- Values match the underlying modules' own data for "today" in the
  clinic's timezone.
- No clinical free-text content appears in any response.

## Testing Requirements
- Integration: one test per role verifying the correct widget set and
  correct aggregate values against seeded "today" data.
- Integration: DOCTOR dashboard never includes clinic-wide revenue.

## Out of Scope
- Customizable/configurable dashboards (fixed per-role layout in V1).
- Real-time push updates (polling the endpoint is sufficient for V1).
- Historical trend charts (that's `reporting.md`).

## Cross-Module Contracts
This module is a leaf — it publishes no contracts for further modules.

## Implementation Notes
Build this after Billing and (optionally) Inventory, since it composes
their data; do not block it on Reporting or Notifications.
