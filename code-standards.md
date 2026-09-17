# code-standards.md — Engineering Standards

Practical conventions. Prefer maintainable code over abstraction for its
own sake — three similar lines beat a premature helper.

## TypeScript / JavaScript Conventions

- TypeScript everywhere in `src/`; `strict: true` in `tsconfig.json`.
- No `any` except at genuine external-boundary edges (and even then,
  narrow it immediately with a Zod parse).
- Prefer `type` for data shapes, `interface` only for things meant to be
  implemented/extended (e.g., `StorageProvider`).
- Async/await only — no raw `.then` chains, no callback-style APIs.

## Naming

- Files: `kebab-case.ts`. Classes/Types: `PascalCase`. Functions/vars:
  `camelCase`. DB tables/columns: `snake_case` (Prisma `@@map`/`@map`).
- Route files: `*.routes.ts`, controllers `*.controller.ts`, services
  `*.service.ts`, repositories `*.repository.ts`, schemas `*.schema.ts`.

## Module Structure

Each business module under `src/modules/<name>/` owns:
```
<name>.routes.ts       # Express Router, wires middleware + controller
<name>.controller.ts   # HTTP-in/HTTP-out only, no business logic
<name>.service.ts      # business logic, orchestration, transactions
<name>.repository.ts   # Prisma queries only
<name>.schema.ts       # Zod request/response schemas
<name>.types.ts        # shared TS types for the module
<name>.test.ts / __tests__/
```
A module never imports another module's `repository.ts` directly — it
calls the other module's `service.ts` exported functions, or, if that
creates a cycle, the shared contract is lifted into `src/shared/`.

## Layer Responsibilities

- **Controller:** parse/validate request (via schema), call service,
  shape HTTP response/status. No SQL, no business rules.
- **Service:** business rules, authorization checks that require domain
  data (e.g., "doctor can only cancel their own appointment"), transaction
  boundaries. No direct `req`/`res` access.
- **Repository:** Prisma calls only. No business logic, no validation.

## Validation

- Every route validates `body`/`query`/`params` with a Zod schema before
  the controller logic runs. Reject with `400` and the envelope's
  `error.details` populated from the Zod issues.
- Never trust client-supplied IDs for authorization decisions — always
  re-derive "is this mine" from the authenticated staff/session context.

## Error Handling

- Throw typed application errors (`NotFoundError`, `ValidationError`,
  `ConflictError`, `ForbiddenError`, etc. in `src/shared/errors/`) from
  services; a single Express error-handling middleware maps them to the
  standard response envelope and status code.
- Never leak stack traces or raw DB errors to clients; log them server-side
  with request correlation ID.

## API Responses

- Always the envelope from [architecture-context.md](architecture-context.md#api-conventions).
- Never return Prisma models directly — map to explicit response DTOs so
  internal fields (password hashes, soft-delete markers) can't leak by
  accident when the schema changes.

## Transactions

- Any write touching more than one table that must succeed/fail together
  uses `prisma.$transaction(...)`.
- Keep transactions short — no external network calls (email/SMS) inside
  a DB transaction.

## Database Access

- All access through Prisma Client; no raw SQL except narrowly-justified
  performance cases, and those must be parameterized (`Prisma.sql`), never
  string-concatenated.
- Migrations are additive/backward-compatible where feasible; destructive
  migrations (drop column/table) require a note in
  [MEMORY.md](MEMORY.md).

## Logging

- Structured logging (Pino), one logger instance from `src/shared/logger`.
- Log level `info` for request lifecycle, `warn` for handled failures,
  `error` for unexpected exceptions.
- Never log: passwords, tokens, full patient medical history, payment
  details. Log entity IDs, not entity contents, for anything PII/PHI-
  adjacent.

## Security

- `helmet`, CORS allow-list, `express-rate-limit` on public and auth
  routes (see [architecture-context.md](architecture-context.md#security-architecture)).
- Authorization middleware runs before the controller for every
  staff-only route; there is no route that relies on the frontend to hide
  a button instead of enforcing access server-side.
- Secrets only from environment variables, validated at startup (fail
  fast, not lazily at first use).

## Testing

- Vitest. Unit tests for services (mock repository layer), integration
  tests for routes against a real test Postgres (via Docker/test
  container or a dedicated test DB), not against mocked Prisma for
  integration-level tests.
- Every module's acceptance criteria (from its requirement doc) map to at
  least one test.
- Tests must actually be run before being reported as passing — no
  claiming success without execution output.

## Environment Variables

- Declared and validated in one place (`src/config/env.ts`) using Zod;
  the app refuses to boot on missing/invalid config.
- `.env.example` kept in sync with every variable the app reads.

## Dependency Management

- Add a dependency only when it removes meaningfully more complexity than
  it adds. Prefer the stack already chosen in
  [architecture-context.md](architecture-context.md) over introducing an
  alternative library that does the same job.
- Pin versions in `package.json` (no floating majors) for anything
  security- or correctness-sensitive (auth, crypto, validation).

## Comments & Documentation

- Default to no comments. Add one only when it explains a non-obvious
  *why* (a workaround, a subtle invariant, a constraint from another
  module) — never a restatement of what the code does.
- No docstring blocks explaining self-evident functions.
- Public module contracts (what another module may call) get a short
  comment at the export site if the name alone doesn't make usage obvious.
