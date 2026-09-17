# Module 1 — Staff, Roles & Auth

## Objective
Provide staff identity, authentication, and role-based authorization so
every later module can protect its endpoints server-side. This module is
implemented immediately after Foundation and before Patients — see
[MEMORY.md](../../MEMORY.md) 2026-09-17 for why.

## Scope
- Staff user model (extends the `Staff` stub from `foundation.md`).
- Login, logout, token refresh.
- Password hashing & management (set/change/reset-by-admin).
- Role-based authorization middleware usable by all future modules.
- Staff management (Admin creates/edits/deactivates staff accounts).
- The canonical permission matrix referenced by every other module.

## Dependencies
- `foundation.md`: `Role` enum, `Staff` table stub, response envelope,
  error classes, Prisma client, `/api/v1` base path.

## Entities / Data Model
```prisma
model Staff {
  id           String    @id @default(uuid())
  name         String
  email        String    @unique
  passwordHash String    @map("password_hash")
  role         Role
  isActive     Boolean   @default(true) @map("is_active")
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  deletedAt    DateTime? @map("deleted_at")

  @@map("staff")
}

model RefreshToken {
  id        String   @id @default(uuid())
  staffId   String   @map("staff_id")
  staff     Staff    @relation(fields: [staffId], references: [id])
  tokenHash String   @map("token_hash")
  expiresAt DateTime @map("expires_at")
  revokedAt DateTime? @map("revoked_at")
  createdAt DateTime @default(now()) @map("created_at")

  @@map("refresh_tokens")
}
```
Store only a hash of the refresh token (never the raw token) — same
principle as passwords.

## Relationships
`Staff.id` is referenced by later modules as the actor for: appointments
(doctor/created-by), clinical notes (author), invoices/payments
(recorded-by), inventory movements (logged-by), and `AuditLog.actorStaffId`.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/auth/login` | none | Email+password → access token + refresh cookie |
| POST | `/api/v1/auth/refresh` | refresh cookie | Rotate refresh token, issue new access token |
| POST | `/api/v1/auth/logout` | access token | Revoke current refresh token |
| GET | `/api/v1/auth/me` | access token | Current staff identity + role + capabilities |
| GET | `/api/v1/staff` | ADMIN | List staff (paginated) |
| POST | `/api/v1/staff` | ADMIN | Create staff account |
| GET | `/api/v1/staff/:id` | ADMIN | Staff detail |
| PATCH | `/api/v1/staff/:id` | ADMIN | Update staff (role, name, active status) |
| POST | `/api/v1/staff/:id/reset-password` | ADMIN | Admin-triggered password reset |
| PATCH | `/api/v1/auth/password` | access token (self) | Staff changes own password |

Request/response bodies follow standard validation (Zod) and the standard
envelope. Login request: `{ email, password }`. Login response data:
`{ accessToken, staff: { id, name, email, role } }` (refresh token goes in
an httpOnly cookie, never in the JSON body).

## Business Rules
- Only `ADMIN` can create, edit, deactivate staff, or force-reset a
  password.
- A deactivated (`isActive = false`) staff member cannot log in and any
  outstanding refresh tokens are revoked immediately on deactivation.
- Refresh tokens are single-use: refreshing issues a new refresh token and
  revokes the old one (rotation). Reuse of a revoked token revokes the
  entire token family for that staff member (theft detection).
- Access tokens are short-lived (~15 min); refresh tokens ~7 days.
- Login failure responses are identical whether the email doesn't exist or
  the password is wrong (`401`, generic message) — no user enumeration.

## Permission Matrix (canonical — referenced by all modules)
| Capability | ADMIN | DOCTOR | RECEPTIONIST | COMPOUNDER |
|---|---|---|---|---|
| Manage staff accounts | Yes | No | No | No |
| Patients: create/edit contact info | Yes | Yes | Yes | No |
| Patients: view full clinical history | Yes | Yes | No | No (reduced view only) |
| Appointments: create/reschedule/cancel | Yes | Own only | Yes | No |
| Appointments: view schedule | Yes | Own only | Yes | Today's queue (read-only) |
| Treatments: manage catalogue | Yes | No | No | No |
| Treatments: create/update patient plan | Yes | Yes | No | Read-only |
| Billing: create invoice/record payment | Yes | No (read-only) | Yes | No |
| Inventory: manage stock | Yes | No (read-only) | No (read-only) | Yes |
| Notifications: manage templates | Yes | No | No | No |
| Reporting: clinic-wide reports | Yes | No | Operational reports only | Inventory reports only |
| Reporting: own workload | Yes | Yes | N/A | N/A |

Every later module's requirement doc should reference this table rather
than redefine it; if a module needs a finer-grained rule than fits this
table, it documents the exception explicitly in its own doc.

## State Machines
Refresh token lifecycle: `active → rotated (replaced by new token)` or
`active → revoked (logout, deactivation, or reuse-detected theft)`.

## Security / Privacy / Compliance
- Passwords hashed with bcrypt (cost ≥ 12); never logged, never returned
  in any API response.
- Refresh tokens stored hashed; raw token only ever exists in the httpOnly
  cookie and in-transit.
- `requireRole(...roles)` middleware must run before controller logic on
  every non-public route added by any module from this point forward.
- Audit log entries required for: login failures (rate-limit relevant),
  role changes, staff deactivation, password resets.
- Rate limit `/api/v1/auth/login` aggressively (e.g. per-IP + per-email)
  to blunt credential-stuffing attempts.

## Transactions / Concurrency
Token rotation (revoke old + insert new refresh token) happens in a single
transaction to avoid a window where both or neither are valid.

## Edge Cases
- Expired access token on a protected route → `401`, frontend should call
  `/auth/refresh` and retry once.
- Refresh token reused after rotation → treat as theft, revoke all tokens
  for that staff member, require fresh login.
- Last remaining ADMIN account cannot be deactivated or have its role
  changed away from ADMIN (prevents lockout).

## Acceptance Criteria
- Login with valid credentials returns an access token and sets the
  refresh cookie; invalid credentials return `401` with a generic message.
- Refresh rotates the token and old token is rejected on reuse.
- `requireRole` middleware blocks a non-ADMIN from every `/staff` write
  endpoint with `403`.
- Deactivated staff cannot log in and their existing tokens are rejected.
- Attempting to deactivate/demote the sole remaining ADMIN is rejected
  with `422`.

## Testing Requirements
- Unit: password hashing/verification, JWT issuance/verification, role
  middleware allow/deny for each role combination.
- Integration: login → refresh → logout full cycle; staff CRUD
  authorization matrix (each role against each staff endpoint).

## Out of Scope
- Patient authentication/accounts (not part of this system in V1 — see
  MEMORY.md).
- OAuth/SSO/social login.
- Multi-factor authentication (may be a future enhancement, not V1).
- Fine-grained per-resource permissions beyond the role matrix above.

## Cross-Module Contracts
Published for all later modules:
- `Staff.id`, `Staff.role` — the actor identity every module attaches to
  its records (`createdBy`, `doctorId`, etc.).
- `requireAuth` and `requireRole(...roles)` Express middleware, importable
  from `src/modules/staff-auth/`.
- The permission matrix above — later docs reference it instead of
  restating it.
- `req.staff` (or equivalent context) populated by `requireAuth` with
  `{ id, role }` for use in service-layer "is this mine" checks.

## Implementation Notes
- Keep JWT payload minimal (`sub`, `role`, `iat`, `exp`) — do not embed PII
  in the token.
- `GET /auth/me`'s "capabilities" field can simply be the row of the
  permission matrix for the caller's role, serialized — avoids the
  frontend hardcoding role logic (see [ui-context.md](../../ui-context.md)).
