# architecture-context.md — Detailed Architecture

Source of truth for architecture decisions and rationale. Module
requirement docs should reference this file rather than repeating it.

**Repository layout:** this is a monorepo with two top-level workspaces,
`backend/` and `frontend/` (see [CLAUDE.md](CLAUDE.md)). Everything in
this file describes the backend unless stated otherwise; all paths below
(`src/...`, `prisma/...`) are relative to `backend/`. `frontend/` is
currently an empty placeholder — no frontend technology has been chosen.

## Architecture Style

**Modular monolith.** One Express/TypeScript process, one PostgreSQL
database, one deployable unit. Business capabilities are separated into
modules under `src/modules/<name>/`, each with its own routes, controller,
service, repository, and Zod schemas. Modules communicate only through
explicit service-level function calls or documented contracts — never by
reaching into another module's repository or database tables directly.

Rejected for V1 (see [project-overview.md](project-overview.md) non-goals):
multi-tenancy, microservices, Kubernetes, event-driven architecture,
distributed transactions, message brokers. Single clinic, single database,
modest load — the complexity is not justified.

## Language Decision: TypeScript over plain JavaScript

**Problem:** the stack must be chosen and documented, not assumed.
**Alternatives considered:** plain JavaScript (+ JSDoc types), TypeScript.
**Decision: TypeScript.** The domain has enough structural complexity
worth catching at compile time — appointment/visit state machines, a
4-role permission matrix enforced across ~10 modules, Prisma models
shared by many services, Zod schemas that should stay in sync with DTO
types. A single/small engineering context (one clinic, likely one or two
developers over time) can't afford to rediscover type mismatches at
runtime. Prisma and Zod are also TS-first tools; using them from plain JS
gives up most of their value.
**Complexity introduced:** a build step (`tsc`) and slightly more
tooling. Judged worth it given the above.
**Revisit only if:** a concrete pain point with the TS toolchain itself
appears — not as a response to "TS felt slow to write."

## RBAC Modeling Decision: enum, not a database-driven Role/Permission table

**Problem:** the domain model brief lists `Role` as its own entity to
investigate — should roles/permissions be data (an admin-configurable
`Role`/`Permission` table) or code (a fixed `Role` enum + a permission
matrix enforced in middleware)?
**Alternatives considered:** (1) `Role` enum + hardcoded permission
matrix (current design, see `staff-auth.md`); (2) `Role`/`Permission`
join-table RBAC with an admin UI to manage them.
**Decision: enum + hardcoded matrix.** The clinic has exactly four fixed
roles with no stated need for custom roles or admin-configurable
permissions. A join-table RBAC system adds real complexity — permission
CRUD endpoints, migration risk if permissions and code drift out of sync,
an extra admin UI surface — for a capability nobody has asked for.
**Complexity avoided:** no permission-management UI/API, no risk of the
DB-stored permissions silently diverging from what the code actually
enforces.
**Revisit only if:** the clinic concretely asks for custom roles or
per-staff-member permission overrides — track that as a future idea, not
a current gap.

## Module Boundaries & Dependency Rules

```
Foundation (includes AuditLog table + recordAudit helper — see audit.md)
   │
   ▼
Staff & Auth
   │
   ├──────────────┐
   ▼              ▼
Clinic Config   Patients
   │              │
   │              ▼
   │         Attachments
   │              │
   └──────┬───────┘
          ▼
     Appointments
          │
          ▼
       Visits
          │
   ┌──────┼───────────┐
   ▼      ▼           ▼
Treatments Prescriptions │
   │                     │
   ▼                     │
 Billing ◄────────────────
   │
   ├───────┬──────────────┐
   ▼       ▼              ▼
 Audit  Notifications   Dashboard
                            │
                            ▼
                 Reporting / Inventory (future/optional, see project-overview.md)
```

Rule: a module may depend on modules above it in this graph (via their
published contracts) but never the reverse, and never on a sibling at the
same level except through explicit IDs (e.g., Billing references
`appointmentId`/`patientTreatmentId`, it does not import those modules'
internals).

**Module naming mapping.** The 2026-09-17 expanded master prompt uses the
names `auth`, `users`, `payments`, `clinic` for some modules. This repo
keeps its existing requirement-doc filenames for continuity (they were
already cross-referenced by several docs before the expansion):
`staff-auth.md` = auth + users, `billing.md` = payments/invoicing,
`clinic-config.md` = clinic. No renaming was done to avoid a wide,
low-value cross-file churn — new work should use the filenames as they
exist in `docs/requirements/`, not the prompt's generic names.

Note: the build order also differs from a naive catalogue-number reading
(Patients was originally listed before Staff/Auth in the first planning
pass). See [MEMORY.md](MEMORY.md) for why Staff & Auth was moved earlier
— Patients holds PII and needs authorization from day one.

## Database Architecture

- PostgreSQL, accessed exclusively through Prisma (schema in
  `prisma/schema.prisma`, one schema file for the whole monolith — modules
  get their own models, not their own databases).
- UUID v4 primary keys (`@default(uuid())`).
- `created_at` / `updated_at` on all tables; `deleted_at` for soft-deletable
  entities (patients, staff, treatments-catalogue) — never hard-delete
  clinical or financial history.
- All migrations go through `prisma migrate` — no manual schema edits.
- Foreign keys are real FK constraints, not application-only references.

## API Conventions

- Base path: `/api/v1`.
- JSON only. Response envelope (fixed, do not deviate per-module):
  ```json
  { "success": true, "data": <payload> }
  { "success": false, "error": { "code": "STRING_CODE", "message": "human readable", "details": {} } }
  ```
  This supersedes an earlier `{ data, error: null }` draft (see
  [MEMORY.md](MEMORY.md)) — the `success` boolean form is simpler to
  discriminate on and matches the error-model example the client/agent
  contract standardized on.
- Pagination: `?page=1&limit=20` query params; response `data` is the array
  and a top-level `meta: { page, limit, total, totalPages }` accompanies it.
- Error codes are UPPER_SNAKE_CASE and specific (e.g.
  `APPOINTMENT_SLOT_UNAVAILABLE`, not a generic `CONFLICT`) so clients can
  branch on them without parsing `message`.
- Standard HTTP status codes: 200/201 success, 400 validation, 401
  unauthenticated, 403 unauthorized, 404 not found, 409 conflict
  (e.g., slot race), 422 business-rule violation, 500 unexpected.
- Request validation via Zod schemas at the route boundary; controllers
  never touch `req.body`/`req.query` before validation.
- OpenAPI spec generated/maintained alongside routes (module owns its
  path definitions).

## Authentication Architecture

- Staff-only. JWT access token (short-lived, ~15 min) + refresh token
  (httpOnly, secure, `SameSite=strict` cookie, rotated on use, ~7 days).
- Passwords hashed with bcrypt (cost factor ≥ 12).
- Patients are **not** authenticated in V1 (see MEMORY.md). Public booking
  endpoints are unauthenticated but rate-limited and validated.
- Full detail owned by [docs/requirements/staff-auth.md](docs/requirements/staff-auth.md).

## Authorization Architecture

- Role-based, enforced server-side via middleware
  (`requireRole(...roles)`), never trusted from the client.
- Roles: `ADMIN`, `DOCTOR`, `RECEPTIONIST`, `COMPOUNDER`.
- Each module's requirement doc states its own permission matrix; the
  canonical matrix lives in
  [docs/requirements/staff-auth.md](docs/requirements/staff-auth.md).

## Appointment / Scheduling Architecture

- Slot identity for a doctor is `(doctor_id, slot_start_at)`. A unique
  constraint on this pair (as a partial index excluding `CANCELLED` rows)
  is the concurrency-safety mechanism — see MEMORY.md decision.
  Application catches unique-violation (Postgres code `23505`) and
  responds `409 Conflict`.
- Clinic hours / doctor availability / holidays are data-driven (stored
  in `clinic-config.md`, not `appointments.md`), so reception can adjust
  them without a code change.
- Appointment lifecycle: `BOOKED → CONFIRMED → CHECKED_IN →
  IN_CONSULTATION → COMPLETED`, with `CANCELLED`/`NO_SHOW` branches — see
  [docs/requirements/appointments.md](docs/requirements/appointments.md)
  for the full state machine and the currently-unresolved question of
  whether `BOOKED → CONFIRMED` is a real manual step.
- All appointment timestamps stored in UTC; clinic operates in a single
  timezone (configurable), converted at the API boundary for display.

## Patient / Clinical-Record Architecture

- Patient identity (demographics/contact, `patients.md`) is a distinct
  module from the clinical encounter record (`visits.md`: symptoms,
  diagnosis, clinical notes, follow-up) and from prescriptions
  (`prescriptions.md`) — three separate modules, not one flat "patient
  record" table, so access control, audit, and later reporting can treat
  identity/PII and clinical narrative differently if needed.
- A `Visit` is the unit of clinical audit trail: every `PatientTreatment`
  and every `Prescription` is anchored to a `Visit.id`, not directly to a
  `Patient.id` — this guarantees clinical actions always have a "when and
  in what encounter" context.
- `Diagnosis` is modeled as its own table (one-to-many per `Visit`)
  rather than a free-text field on the visit, so multiple concurrent
  issues (e.g. two different teeth) are structured data, not a
  text-parsing target for future reporting.

## Attachment Architecture

- Attachments (X-rays, documents) are their own module
  (`attachments.md`), referenced by `patientId` and optionally `visitId`
  — not embedded fields on Patient. Files are referenced by a storage
  key/URL, never stored as DB blobs.
- Storage provider is an interface (`StorageProvider.put/get/delete`) —
  concrete provider (local disk vs. S3-compatible) is chosen at
  Attachments-implementation time (open question in MEMORY.md).
- Upload is two-step (signed upload URL → direct upload → confirm) so
  large files never proxy through the API process.

## Prescription Architecture

- `Prescription` (one per issuing event) + `PrescriptionItem`
  (medicine/dosage/frequency/duration line items), both anchored to a
  `Visit.id`. Medicine name/dosage are plain strings in V1 — no drug
  master catalogue or interaction-checking (not justified for a single
  small clinic; see `prescriptions.md` Out of Scope).

## Dashboard vs. Reporting

- `dashboard.md` (real-time, today-focused, fixed per-role widgets) and
  `reporting.md` (historical, date-range, deeper aggregates) are
  deliberately separate modules rather than one — different read
  patterns (today snapshot vs. range query) and different urgency (a
  dashboard should feel instant; a report can be a bit slower).

## Billing Architecture

- Invoice → InvoiceLineItem → Payment, all linked to `patientId` and
  optionally `appointmentId`/`patientTreatmentId`.
- Payments recorded by staff after the fact
  (`CASH`/`CARD`/`UPI`/`BANK_TRANSFER`/`OTHER` enum) — no payment-gateway
  integration in V1.
- Invoices, once issued, are corrected via adjustment/credit entries, not
  by mutating the original line items — preserves an auditable financial
  history.

## Error Model

Every thrown application error maps to one category, each with a fixed
HTTP status and error-code prefix:

| Category | Status | Example code |
|---|---|---|
| Validation | 400 | `VALIDATION_ERROR` |
| Authentication | 401 | `UNAUTHENTICATED` |
| Authorization | 403 | `FORBIDDEN` |
| Not Found | 404 | `NOT_FOUND` |
| Conflict | 409 | `APPOINTMENT_SLOT_UNAVAILABLE` |
| Business Rule | 422 | `INVOICE_OVERPAYMENT` |
| Database/Unexpected | 500 | `INTERNAL_ERROR` |

Services throw the typed error classes from `src/shared/errors`
(`foundation.md`); the single Express error-handling middleware maps them
to this table and the envelope in API Conventions. Unexpected/database
errors are never passed through with their original message/stack to the
client — only `INTERNAL_ERROR` with a generic message, full detail logged
server-side with a request-correlation ID.

## Notification Architecture

- Provider-agnostic `NotificationProvider` interface
  (`send(to, template, data)`); Email/SMS/WhatsApp are pluggable
  implementations selected by config. The core appointment/billing flows
  emit notification *events*, not direct provider calls.
- Concrete provider integration happens only when
  [docs/requirements/notifications.md](docs/requirements/notifications.md)
  is implemented, not before.

## Audit Architecture

- `AuditLog` table: `actor_id`, `action`, `entity_type`, `entity_id`,
  `metadata` (jsonb), `created_at`, `ip_address`. Created in
  `foundation.md` (needed from every module's first commit); the
  ADMIN-facing query API and the consolidated list of exactly which
  actions must be audited live in
  [docs/requirements/audit.md](docs/requirements/audit.md) — that list is
  the single source of truth, not repeated here.
- Audit logs are append-only; no update/delete API is ever exposed for
  them, for any role including ADMIN.

## Transaction & Concurrency Strategy

- Default isolation: Postgres default (`READ COMMITTED`).
- Multi-step writes that must be atomic (e.g., create invoice + line
  items) run inside a single Prisma `$transaction`.
- Concurrency-sensitive paths (appointment booking) rely on unique
  constraints + conflict handling rather than elevated isolation levels
  (see decision above) — keep this the default posture unless a specific
  module proves it insufficient, in which case escalate through Change
  Control, not silently.

## Security Architecture

- Helmet for HTTP headers, CORS restricted to known frontend origin(s),
  `express-rate-limit` on public/auth endpoints.
- All PII/PHI-bearing fields excluded from application logs (structured
  logger redacts known sensitive keys).
- Secrets via environment variables only, never committed; `.env.example`
  documents required vars without values.
- Input validation (Zod) at every route boundary; output shaping to avoid
  over-exposing internal fields (e.g., password hashes never serialized).

## Deployment Architecture

- Dockerized app + Postgres via a repo-root `docker-compose.yml` (builds
  the app image from `./backend/Dockerfile`) for local/dev parity — kept
  at the repo root rather than inside `backend/` so it can add a
  `frontend` service later without moving.
- Single deployable image for the backend monolith; migrations run as an
  explicit release step (`prisma migrate deploy`), not automatically on
  boot.
- Environment-based config (`development`, `test`, `production`) via a
  validated config module (fail fast on missing/invalid env vars).
- The frontend (once a stack is chosen) is deployed as its own
  independent artifact/service — this is a monorepo for shared planning
  docs and coordinated versioning, not a single combined deployable.
