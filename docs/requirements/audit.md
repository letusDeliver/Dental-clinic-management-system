# Module — Audit Logs

## Objective
Provide the queryable, ADMIN-facing view over the `AuditLog` table that
every other module writes to, and be the canonical place documenting
*which* actions across the system must be audited. The logging mechanism
itself (the `AuditLog` table and a shared `recordAudit(...)` helper) is
established in `foundation.md`'s shared infrastructure; this module adds
the query/read API and consolidates the audit requirement list.

## Scope
- `AuditLog` schema ownership (defined here as the canonical source,
  though the table exists from Foundation onward for other modules to
  write to).
- Read/query API for ADMIN.
- The consolidated list of audited actions across all modules.

## Dependencies
- `foundation.md`: shared infra (the `AuditLog` table itself is created
  in Foundation since modules need to write to it from their first
  commit — this module only adds the ADMIN read API on top).
- `staff-auth.md`: `Staff.id` (actor), ADMIN-only access.

## Entities / Data Model
```prisma
model AuditLog {
  id          String   @id @default(uuid())
  actorId     String   @map("actor_id") // Staff.id
  action      String   // e.g. "PATIENT_HISTORY_VIEWED", "VISIT_CREATED"
  entityType  String   @map("entity_type") // e.g. "Patient", "Visit", "Invoice"
  entityId    String   @map("entity_id")
  metadata    Json?    // small, structured, non-sensitive context
  ipAddress   String?  @map("ip_address")
  createdAt   DateTime @default(now()) @map("created_at")

  @@map("audit_logs")
  @@index([entityType, entityId])
  @@index([actorId])
  @@index([createdAt])
}
```
Append-only: no update/delete API is ever exposed for this table, by any
role, including ADMIN.

## Relationships
`AuditLog.actorId` → `Staff.id`. `entityType`/`entityId` are a loose
polymorphic reference (no hard FK, since it spans many tables) to
whatever record the action concerned.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/audit-logs` | ADMIN | Query logs (filter by actor, entityType, entityId, date range, paginated) |
| GET | `/api/v1/audit-logs/:id` | ADMIN | Single log entry detail |

No write/delete endpoints exist — entries are created only via the
internal `recordAudit(...)` helper called by other modules' services.

## Business Rules
- `metadata` must never contain clinical free text (notes, diagnosis
  descriptions, prescription content), full PII (phone/email/address), or
  secrets — only small structured facts useful for the audit trail
  (e.g. `{ "previousRole": "RECEPTIONIST", "newRole": "ADMIN" }` for a
  role-change entry). This is enforced by convention/code review at each
  call site, not by a runtime filter — call sites must be deliberate.
- Audit logs are retained indefinitely in V1 (no automatic purge) — same
  posture as clinical/financial record retention (see `patients.md`).

## Consolidated Audit Requirement List
The following actions **must** call `recordAudit(...)` — this list is
the single source of truth; each owning module's doc references this
section rather than repeating it:

| Action | Owning Module |
|---|---|
| Full patient clinical history viewed | patients.md |
| Visit created / amended | visits.md |
| Diagnosis added / removed | visits.md |
| Prescription created / amended | prescriptions.md |
| Attachment uploaded / viewed / deleted | attachments.md |
| Staff role changed / deactivated | staff-auth.md |
| Login failure (rate-limit relevant) | staff-auth.md |
| Password reset (admin-triggered) | staff-auth.md |
| Invoice created / voided | billing.md |
| Payment recorded | billing.md |

## State Machines
Not applicable.

## Security / Privacy / Compliance
- ADMIN-only, no exceptions — audit logs can reveal patterns about who
  accessed what, which is itself sensitive.
- The log query API supports pagination and filtering but never a
  full-table unbounded export in a single call (enforce a max page size).

## Transactions / Concurrency
Writes are simple single-row inserts from the calling module's own
transaction (or immediately after, if the audited action itself doesn't
need to be atomic with the log write — decide per call site; a failed
audit-log insert should not be allowed to silently swallow — log a
`warn` server-side if it fails, per `code-standards.md` logging rules).

## Edge Cases
- Non-ADMIN calling any endpoint → `403`.
- Querying with an invalid/unbounded date range → `400`.

## Acceptance Criteria
- Every action in the Consolidated Audit Requirement List produces
  exactly one `AuditLog` row when exercised (verified via that owning
  module's own test suite, not retested here).
- `GET /audit-logs` supports filtering by actor/entityType/entityId/date
  and paginates correctly.
- Non-ADMIN access is rejected.

## Testing Requirements
- Integration: query filters return correct subsets against seeded data.
- Integration: non-ADMIN access rejected on both endpoints.
- Integration: pagination behaves correctly at boundary values.

## Out of Scope
- Real-time audit alerting/SIEM integration.
- Exporting logs to an external system.
- Tamper-evidence mechanisms (e.g. hash chaining) — not justified for a
  single small clinic's threat model in V1.

## Cross-Module Contracts
Published for all modules: the `recordAudit(actorId, action, entityType,
entityId, metadata?)` helper, importable from `src/shared/audit` (created
in Foundation) — every module listed in the Consolidated Audit
Requirement List calls this rather than writing to the `audit_logs` table
directly.

## Implementation Notes
Because other modules need to write audit entries from their very first
commit, the `AuditLog` Prisma model and the `recordAudit` helper should be
created as part of `foundation.md`'s shared infrastructure task, even
though this document (and the ADMIN-facing query API) is scheduled later
in the build order. Flag this dependency explicitly when creating the
Foundation task.
