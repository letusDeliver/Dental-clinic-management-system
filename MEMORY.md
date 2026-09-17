# MEMORY.md — Durable Decisions & Discoveries

Newest first. Compact. Not a history dump — only what a future session must
not rediscover the hard way.

---

## 2026-09-17 — Planning session: repository was empty at project start
Repository contained only a one-line README before this session. All
technology and architecture decisions below are therefore fresh choices,
not preservations of prior work. **How to apply:** future sessions should
not go looking for "existing conventions" predating this date — this
session's docs *are* the origin of the conventions.

## 2026-09-17 — Implementation order deviates from the naive module-number order
The module numbering in the original planning brief (1 Patients, 2
Appointments, 3 Staff/Auth, ...) is a *catalogue* order, not a *build*
order. Staff & Auth is promoted to be implemented immediately after
Foundation and **before** Patients.
**Why:** Patients & Records holds PII/medical data and its API endpoints
must be authorization-protected from the first commit (role checks:
ADMIN/DOCTOR full, RECEPTIONIST limited, COMPOUNDER minimal). Building
Patients before Auth would mean either shipping unprotected PII endpoints
or retrofitting authz onto an already-"complete" module.
**How to apply:** always implement in the order in
[progress-tracker.md](progress-tracker.md) / [CLAUDE.md](CLAUDE.md), not
the section numbering of any older planning document. The requirement doc
filenames (`patients.md`, `staff-auth.md`, ...) are unchanged — only the
task-creation sequence changed. See [[architecture-context]].

## 2026-09-17 — Patients are not authenticated users in V1
Public booking and the patient-facing surface do not require patient
login/accounts. Patients are identified by contact details (name + phone,
optionally email) captured at booking time and reconciled by
reception/admin staff against existing patient records.
**Why:** Keeps V1 scope bounded (per [project-overview.md](project-overview.md)
non-goals) and matches a real single-clinic front-desk workflow where
identity is verified in person or by phone, not via a patient portal login.
**How to apply:** [[staff-auth]] governs only staff identities. Any future
"patient portal / patient login" is a distinct, explicitly future-scoped
feature — do not fold it into staff-auth's user model or JWT issuance.

## 2026-09-17 — Appointment slot booking concurrency strategy chosen
Use a Postgres **unique constraint** on `(doctor_id, slot_start_at)` (or
equivalent slot identity) as the source of truth for "only one booking per
slot," with the application catching the unique-violation (Postgres error
23505) and returning `409 Conflict`. Not row-locking/`SELECT ... FOR
UPDATE`, not SERIALIZABLE isolation for the whole transaction.
**Why:** Simplest mechanism that is still fully correct under concurrent
writes; avoids the complexity and retry-handling burden of stricter
isolation levels for a single-clinic, non-high-throughput system.
**How to apply:** [[appointments]] requirement doc must specify this
constraint explicitly in its Entities/Concurrency sections. Any future
change to this strategy is an architectural decision — record it here and
update [[architecture-context]] before changing the schema.

## 2026-09-17 — Response envelope & API conventions fixed
All JSON APIs return `{ "data": ..., "error": null }` on success and
`{ "data": null, "error": { "code", "message", "details" } }` on failure.
Base path `/api/v1`. UUID v4 primary keys. Timestamps in UTC ISO-8601
(`created_at`, `updated_at`, `deleted_at` for soft delete where applicable).
**Why:** A single convention fixed once in [[architecture-context]] means
every module doc can just say "follow standard conventions" instead of
re-specifying response shape, saving requirement-doc token budget.
**How to apply:** Every module's API Endpoints section assumes this
envelope; only deviations need to be called out.

## 2026-09-17 — No insurance, online payments, or patient portal in V1
Explicitly out of scope per [project-overview.md](project-overview.md).
Billing module tracks payments recorded by staff (cash/card/UPI as a plain
`method` string), not payment-gateway integration.
**How to apply:** Do not let "billing" or "insurance" naming in any future
requirement pull in claims processing or gateway integration without an
explicit new requirement doc authorizing it.

---

## Open / Unresolved Decisions
- Object storage choice for patient attachments (local disk vs. S3-compatible)
  is deferred to when [[patients]] is actually implemented — Foundation
  should provide a storage abstraction interface, not commit to a provider.
- Notification provider(s) (email/SMS/WhatsApp) deferred to [[notifications]]
  implementation time; only the abstraction is designed now.
