# MEMORY.md — Durable Decisions & Discoveries

Newest first. Compact. Not a history dump — only what a future session must
not rediscover the hard way.

---

## 2026-09-17 — Repository restructured as a monorepo: `backend/` + `frontend/`
Created `backend/` and `frontend/` as separate top-level workspaces.
Root-level planning docs (`CLAUDE.md`, `MEMORY.md`, `docs/`, etc.) stay at
the repo root since they govern the whole project, not just one
workspace. `docker-compose.yml` also stays at the repo root (builds from
`./backend/Dockerfile`) so a future `frontend` service can be added
without moving it.
**Why:** the user asked for backend and frontend to live in clearly
separated folders. No frontend technology has been chosen yet — `frontend/`
is an empty placeholder with a README pointing to `ui-context.md` and
instructing that a stack decision must be recorded here before any
frontend code is written.
**How to apply:** every path in `docs/requirements/*.md` and
`architecture-context.md` (e.g. `src/config/env.ts`) is relative to
`backend/`, not the repo root — this is called out at the top of
`architecture-context.md`, `code-standards.md`, and `foundation.md`.
Do not scaffold a frontend framework speculatively; get an explicit stack
decision first.

## 2026-09-17 — Domain model expanded: Visits, Prescriptions, Attachments, Audit, Clinic Config, Dashboard split into their own modules
A second, more detailed master prompt asked for Visits/Diagnosis/
Prescriptions/Follow-ups/Clinic Configuration/Staff Management/Audit Logs
as first-class concerns. Response: split what was a single `patients.md`
(which had embedded `ClinicalVisit`/`Attachment`) into five modules —
`patients.md` (identity/contact only), `visits.md` (Visit + Diagnosis +
follow-up fields), `prescriptions.md` (Prescription + PrescriptionItem,
new), `attachments.md` (extracted, own storage-abstraction ownership),
and `audit.md` (dedicated ADMIN query API + the consolidated
audit-requirement list, though the `AuditLog` table itself is created in
`foundation.md` since other modules need to write to it immediately).
Also added `clinic-config.md` (extracted `DoctorAvailability`/
`ClinicHoliday`/`ClinicProfile` out of `appointments.md`) and
`dashboard.md` (new — real-time per-role "today" view, distinct from
`reporting.md`'s historical range reports).
**Why:** `patients.md` was becoming an overloaded "everything about a
patient" module mixing PII-tier and clinical-tier sensitivity; the new
prompt's explicit entity list (Visit, Diagnosis, Prescription,
PrescriptionItem) doesn't fit cleanly inside it, and clinic scheduling
config has different write-frequency/ownership than appointment
transactions themselves.
**How to apply:** [[patients]] no longer exposes `/history`, `/visits`,
or `/attachments` — those moved to [[visits]] and [[attachments]].
[[appointments]] no longer defines `DoctorAvailability`/`ClinicHoliday` —
those moved to [[clinic-config]]. Every `PatientTreatment`
(treatments.md) and `Prescription` now requires a `visitId` (not
optional) — see business-rules.md "Can a patient have treatment without a
visit?" → No.

## 2026-09-17 — API envelope changed to `{ success, data|error }`, superseding the earlier `{ data, error: null }` draft
**Why:** The expanded master prompt's error-model example uses a
`success: false` discriminant. Adopted that shape everywhere instead of
running two conventions — simpler to discriminate on in client code, and
avoids the redundant `"error": null` / `"data": null` noise the first
draft had.
**How to apply:** `architecture-context.md` API Conventions section is
now the sole source of truth for the envelope; `code-standards.md`
references it. No requirement doc repeats the shape inline except as a
one-off example (e.g. foundation.md's health check).

## 2026-09-17 — Appointment state machine expanded; confirmation step left explicitly unresolved
Added `CONFIRMED` and `IN_CONSULTATION` to the appointment lifecycle
(`BOOKED → CONFIRMED → CHECKED_IN → IN_CONSULTATION → COMPLETED`).
**Why:** the expanded master prompt's proposed lifecycle included these;
they reflect a real front-desk workflow (confirm by phone, then a
distinct "doctor is now seeing them" state useful for the receptionist's
queue view).
**How to apply:** whether `BOOKED → CONFIRMED` is a real manual step or
appointments should just be created directly as `CONFIRMED` is tracked as
an **UNRESOLVED** item in [[business-rules]] — implementers must not
invent an answer; default behavior (documented in appointments.md) is
appointments start `CONFIRMED` directly until the clinic decides
otherwise.

## 2026-09-17 — RBAC modeled as an enum + hardcoded matrix, not a database Role/Permission table
**Why:** four fixed roles, no stated need for dynamic/custom permissions;
a join-table RBAC system would add a permission-management UI/API and a
drift risk (DB permissions vs. code enforcement) for a capability nobody
asked for. Full rationale in `architecture-context.md` "RBAC Modeling
Decision."
**How to apply:** do not introduce a `Role`/`Permission` table unless the
clinic concretely asks for custom roles — treat that as future work, not
a gap in the current design.

## 2026-09-17 — Language confirmed: TypeScript (not plain JavaScript)
Evaluated per the master prompt's explicit request to document the
choice. Rationale (state machines, RBAC matrix, Prisma/Zod being TS-first
tools) recorded in `architecture-context.md` "Language Decision." No
change from the original plan — this makes the reasoning explicit rather
than assumed.

## 2026-09-17 — Business rules catalogue created (`docs/business-rules.md`)
Ambiguous/cross-cutting domain questions (can a patient book multiple
appointments same day, can a visit exist without an appointment, etc.)
are now tracked in one file with explicit RESOLVED/UNRESOLVED status
instead of being answered silently inside module docs or left
undocumented.
**How to apply:** check this file before inventing an answer to a
domain-rule question during implementation; move UNRESOLVED items to
RESOLVED (with the decision) once the clinic/stakeholder actually
answers them, and update the owning module's doc at the same time.

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

## 2026-09-17 — API conventions fixed (envelope shape superseded later same day — see top of file)
Base path `/api/v1`. UUID v4 primary keys. Timestamps in UTC ISO-8601
(`created_at`, `updated_at`, `deleted_at` for soft delete where applicable).
The original envelope drafted here (`{ data, error: null }`) was replaced
by `{ success, data|error }` — see the "API envelope changed" entry near
the top of this file; that entry is authoritative for response shape.
**Why:** A single convention fixed once in [[architecture-context]] means
every module doc can just say "follow standard conventions" instead of
re-specifying response shape, saving requirement-doc token budget.
**How to apply:** Every module's API Endpoints section assumes the
current envelope in architecture-context.md; only deviations need to be
called out.

## 2026-09-17 — No insurance, online payments, or patient portal in V1
Explicitly out of scope per [project-overview.md](project-overview.md).
Billing module tracks payments recorded by staff
(cash/card/UPI/bank-transfer as a plain enum), not payment-gateway
integration.
**How to apply:** Do not let "billing" or "insurance" naming in any future
requirement pull in claims processing or gateway integration without an
explicit new requirement doc authorizing it.

---

## Open / Unresolved Decisions
- Object storage choice for attachments (local disk vs. S3-compatible) is
  deferred to when [[attachments]] is actually implemented — Foundation
  should provide a storage abstraction interface, not commit to a provider.
- Notification provider(s) (email/SMS/WhatsApp) deferred to [[notifications]]
  implementation time; only the abstraction is designed now.
- Full catalogue of domain-rule open questions (slot duration, clinic
  timezone, ad-hoc doctor unavailability, appointment
  confirmation-step mechanics) now tracked in
  [docs/business-rules.md](docs/business-rules.md) rather than
  duplicated here — check that file, not just this list.
