# architecture-context.md — Detailed Architecture

Source of truth for architecture decisions and rationale. Module
requirement docs should reference this file rather than repeating it.

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

## Module Boundaries & Dependency Rules

```
Foundation
   │
   ▼
Staff & Auth  ──────────┐
   │                    │
   ▼                    │
Patients & Records      │
   │                    │
   ▼                    │
Appointments ◄──────────┘ (needs staff/doctor identity + roles)
   │
   ▼
Treatments
   │
   ▼
Billing
   │
   ├──────────────┐
   ▼              ▼
Inventory     Notifications
   │              │
   └──────┬───────┘
          ▼
      Reporting
```

Rule: a module may depend on modules above it in this graph (via their
published contracts) but never the reverse, and never on a sibling at the
same level except through explicit IDs (e.g., Billing references
`appointmentId`/`treatmentId`, it does not import Appointments' internals).

Note: this build order differs from the original catalogue numbering
(Patients was originally listed before Staff/Auth). See
[MEMORY.md](MEMORY.md) 2026-09-17 entry for why Staff & Auth was moved
earlier — Patients holds PII and needs authorization from day one.

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
  { "data": <payload or null>, "error": null }
  { "data": null, "error": { "code": "STRING_CODE", "message": "human readable", "details": [] } }
  ```
- Pagination: `?page=1&limit=20` query params; response `data` is the array
  and a top-level `meta: { page, limit, total, totalPages }` accompanies it.
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
  constraint on this pair is the concurrency-safety mechanism — see
  MEMORY.md decision. Application catches unique-violation (Postgres code
  `23505`) and responds `409 Conflict`.
- Clinic hours / doctor availability / holidays are data-driven (stored),
  not hardcoded, so reception can adjust them.
- All appointment timestamps stored in UTC; clinic operates in a single
  timezone (configurable), converted at the API boundary for display.

## Patient / Clinical-Record Architecture

- Patient identity (demographics/contact) is a distinct concern from
  clinical data (visits, diagnosis, notes) — modeled as related entities,
  not one flat table, so access control can differentiate them later if
  needed.
- Attachments (X-rays, documents) are referenced by a storage key/URL, not
  stored as DB blobs. Storage provider is an interface
  (`StorageProvider.put/get`) — concrete provider (local disk vs. S3-
  compatible) is chosen at Patients-implementation time (open question in
  MEMORY.md).

## Billing Architecture

- Invoice → InvoiceLineItem → Payment, all linked to `patientId` and
  optionally `appointmentId`/`patientTreatmentId`.
- Payments recorded by staff after the fact (cash/card/UPI/other as a
  plain enum/string) — no payment-gateway integration in V1.
- Invoices, once issued, are corrected via adjustment/credit entries, not
  by mutating the original line items — preserves an auditable financial
  history.

## Notification Architecture

- Provider-agnostic `NotificationProvider` interface
  (`send(to, template, data)`); Email/SMS/WhatsApp are pluggable
  implementations selected by config. The core appointment/billing flows
  emit notification *events*, not direct provider calls.
- Concrete provider integration happens only when
  [docs/requirements/notifications.md](docs/requirements/notifications.md)
  is implemented, not before.

## Audit Architecture

- `AuditLog` table: `actor_staff_id`, `action`, `entity_type`, `entity_id`,
  `metadata` (jsonb), `created_at`, `ip_address`.
- Mandatory audit entries for: any read of a patient's full medical
  history, any write to clinical records, any staff role change, any
  invoice/payment mutation.
- Audit logs are append-only; no update/delete API is ever exposed for
  them.

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

- Dockerized app + Postgres via `docker-compose` for local/dev parity.
- Single deployable image for the monolith; migrations run as an explicit
  release step (`prisma migrate deploy`), not automatically on boot.
- Environment-based config (`development`, `test`, `production`) via a
  validated config module (fail fast on missing/invalid env vars).
