# Module 0 — Foundation

## Objective
Establish the runnable project skeleton (TypeScript/Express/Prisma/
Postgres) with cross-cutting infrastructure — config, logging, error
handling, validation wiring, security middleware, testing setup, Docker,
and OpenAPI scaffolding — so business modules can be built on top of it
without each one re-inventing plumbing.

## Scope
- Project scaffolding: `package.json`, `tsconfig.json`, lint/format config.
- `src/config/env.ts`: Zod-validated environment config, fail-fast on boot.
- `src/shared/logger`: Pino structured logger, PII-safe redaction list.
- `src/shared/errors`: typed error classes (`NotFoundError`,
  `ValidationError`, `ConflictError`, `ForbiddenError`, `UnauthorizedError`)
  + a single Express error-handling middleware mapping them to the
  standard response envelope.
- `src/shared/http`: response envelope helpers (`ok(data)`,
  `paginated(data, meta)`), so controllers don't hand-build JSON shapes.
- Security middleware wiring: `helmet`, CORS allow-list, `express-rate-limit`
  (defaults; per-route overrides come later with real routes).
- `src/shared/prisma`: Prisma client singleton.
- `prisma/schema.prisma`: initial file with only a `Staff` table stub and
  the `Role` enum (`ADMIN`, `DOCTOR`, `RECEPTIONIST`, `COMPOUNDER`) — full
  Staff model detail belongs to `staff-auth.md`, but the enum and table
  must exist here because later modules' schemas reference `Staff`.
- `src/app.ts` (Express app factory) and `src/server.ts` (boot entrypoint).
- Health check endpoint: `GET /api/v1/health` → `{ data: { status: "ok" }, error: null }`, unauthenticated.
- Test setup: Vitest config, a test-database strategy (separate Postgres
  DB/schema for tests), one smoke test hitting `/api/v1/health`.
- `Dockerfile` + `docker-compose.yml` (app + Postgres) for local dev.
- OpenAPI scaffolding: a mechanism (e.g. `zod-to-openapi`) wired so future
  modules can register their schemas/paths; does not need any real paths
  documented yet beyond `/health`.
- `.env.example` covering every env var `env.ts` reads.
- Update `CLAUDE.md` "Important Commands" section with the real `dev`,
  `build`, `start`, `test`, `lint`, `migrate` scripts once they exist.

## Dependencies
None — this is the first module.

## Entities / Data Model
Only the minimal stub needed by later modules:
```prisma
enum Role {
  ADMIN
  DOCTOR
  RECEPTIONIST
  COMPOUNDER
}

model Staff {
  id        String   @id @default(uuid())
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("staff")
}
```
`staff-auth.md` will extend `Staff` with real fields (email, passwordHash,
role, name, isActive, etc.) via a migration in that module's task — do not
guess those fields here.

## Relationships
None yet — this module has no business relationships.

## API Endpoints
| Method | Route | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/health` | none | Liveness/readiness check |

## Business Rules
- The app must fail to boot (not fail on first request) if required env
  vars are missing/invalid.
- No business logic in this module beyond the health check.

## State Machines
None.

## Security / Privacy / Compliance
- No PII is handled by this module.
- Security middleware (helmet/CORS/rate-limit) must be present even though
  no sensitive routes exist yet, so later modules inherit it automatically.
- Logger must redact known sensitive keys (`password`, `token`,
  `authorization`) by default, even before any module logs them.

## Transactions / Concurrency
Not applicable.

## Edge Cases
- Missing/invalid required env var → process exits with a clear error
  message at startup, not a runtime crash on first request.
- Database unreachable at boot → health check should reflect `503`/error
  status rather than hanging.

## Acceptance Criteria
- `npm run dev` starts the server using env config.
- `GET /api/v1/health` returns `200` with the standard envelope.
- `npm test` runs and the smoke test passes.
- `npm run lint` and `npm run build` (tsc) succeed with zero errors.
- `docker-compose up` brings up app + Postgres and the health check
  succeeds against the containerized app.
- Prisma migration for the `Role` enum and `Staff` stub applies cleanly
  (`prisma migrate dev`).
- `.env.example` is complete and accurate relative to `env.ts`.

## Testing Requirements
- Unit test: env validation rejects missing required vars.
- Integration test: `GET /api/v1/health` returns `200` and the correct
  envelope shape.

## Out of Scope
- Any real Staff fields beyond the stub (belongs to `staff-auth.md`).
- Any business endpoints.
- Any authentication logic.
- Choosing/integrating a concrete object-storage or notification provider.

## Cross-Module Contracts
Published for all later modules:
- Standard response envelope helpers in `src/shared/http`.
- Standard error classes in `src/shared/errors` (later modules throw
  these; they must not invent parallel error types).
- `Role` enum values and the `Staff` table name (`staff`) — later modules
  extend, they do not redefine.
- Prisma client singleton import path (`src/shared/prisma`).
- Base API path `/api/v1`.

## Implementation Notes
- Keep the initial Prisma schema minimal on purpose — `staff-auth.md`
  owns the real `Staff` model migration next; don't pre-guess its fields
  beyond the stub above.
- Do not add any business routes "to test the infrastructure" — the
  health check is sufficient per the master plan's Module 0 guidance.
