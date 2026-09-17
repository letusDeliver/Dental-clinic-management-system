# CLAUDE.md — Dental Clinic Backend

## Project Purpose
Backend for a **single** dental clinic: public booking, staff auth, patient
records, appointments, treatments, billing, inventory, notifications,
reporting. Not multi-tenant. See [project-overview.md](project-overview.md).

## Technology Stack
Node.js (LTS) + TypeScript, Express.js, PostgreSQL, Prisma ORM, Zod,
JWT (access + refresh) + bcrypt, Helmet/CORS/rate-limiting, Pino logging,
OpenAPI/Swagger, Vitest, Docker. Rationale: [architecture-context.md](architecture-context.md).

## Architecture Summary
Modular monolith. One Express app, one Postgres database, modules under
`src/modules/<module>/` (routes → controller → service → repository →
Prisma). Shared cross-cutting code in `src/shared/`. Full detail:
[architecture-context.md](architecture-context.md).

## Repository Structure (target — built out in Foundation module)
```
src/
  config/
  modules/
    staff-auth/ patients/ appointments/ treatments/
    billing/ inventory/ notifications/ reporting/
  shared/          # middleware, errors, logger, validation, prisma client
  app.ts server.ts
prisma/
  schema.prisma migrations/
docs/
  requirements/    # one bounded module spec per file
tests/
```
As of now the repository contains only planning documentation — no
application code has been written yet.

## Current Phase
**Planning** — persistent memory system and module requirement documents
established. No implementation has started.

## Current Module
None implemented yet. Next up: **Module 0 — Foundation**
([docs/requirements/foundation.md](docs/requirements/foundation.md)).

## Implementation Order (see MEMORY.md for why this differs from a naive reading)
```
0. Foundation
1. Staff & Auth        (moved ahead of Patients — Patients needs roles/authz)
2. Patients & Records
3. Appointments
4. Treatments
5. Billing
6. Inventory  ┐
7. Notifications ┘ (parallel, both depend on 3/4/5)
8. Reporting
```

## Important Commands
Not yet applicable — no `package.json` exists. Module 0 must establish and
document real commands here (`dev`, `build`, `test`, `migrate`, `lint`).

## Critical Engineering Rules
- One module = one bounded requirement doc = one implementation task.
- Never implement business modules without reading the module's
  requirement doc in `docs/requirements/` plus [MEMORY.md](MEMORY.md) and
  [progress-tracker.md](progress-tracker.md) first.
- Never change another module's contract silently — see
  [ai-workflow-rules.md](ai-workflow-rules.md) §Change Control.
- Backend enforces authorization; never trust the frontend for access
  control.
- No multi-tenancy, microservices, or event-driven infra unless a concrete
  requirement justifies it.

## Documentation Map
| File | Purpose |
|---|---|
| [MEMORY.md](MEMORY.md) | Durable decisions & discoveries |
| [progress-tracker.md](progress-tracker.md) | Current execution state |
| [ai-workflow-rules.md](ai-workflow-rules.md) | How agents must operate |
| [architecture-context.md](architecture-context.md) | Detailed architecture |
| [code-standards.md](code-standards.md) | Engineering conventions |
| [project-overview.md](project-overview.md) | Product scope, actors, non-goals |
| [ui-context.md](ui-context.md) | Frontend/backend contract |
| `docs/requirements/*.md` | Per-module specs |

## Current Blockers
None. Awaiting decision to start Module 0 (Foundation) implementation task.
