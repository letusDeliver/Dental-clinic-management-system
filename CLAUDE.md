# CLAUDE.md — Dental Clinic Backend

> **Starting a new session?** Run the `session-start` skill
> (`.claude/skills/session-start/SKILL.md`, invoke with `/session-start`)
> before doing anything else — it walks through this file, MEMORY.md,
> progress-tracker.md, and the current module's requirement doc in order.

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

## Repository Structure (monorepo: backend/ + frontend/ as separate workspaces)
```
backend/                # Node.js/TypeScript API — see backend/README.md
  src/
    config/
    modules/
      staff-auth/ clinic-config/ patients/ attachments/
      appointments/ visits/ treatments/ prescriptions/
      billing/ audit/ notifications/ dashboard/
      inventory/ reporting/      # lower-priority / future, see project-overview.md
    shared/        # middleware, errors, logger, validation, prisma client, audit helper
    app.ts server.ts
  prisma/
    schema.prisma migrations/
  package.json tsconfig.json Dockerfile
  tests/
frontend/               # placeholder — no tech stack chosen yet, see frontend/README.md
docs/
  requirements/    # one bounded module spec per file
  business-rules.md   # cross-cutting domain-rule catalogue, RESOLVED/UNRESOLVED
docker-compose.yml # orchestrates backend (+ Postgres); builds from ./backend
```
All backend paths in `docs/requirements/*.md` and `architecture-context.md`
(e.g. `src/config/env.ts`) are relative to `backend/`, the backend
workspace root, unless stated otherwise. As of now no application code
has been written yet in either workspace.

## Current Phase
**Planning** — persistent memory system and module requirement documents
established, since expanded to cover Visits/Diagnosis/Prescriptions/
Attachments/Audit/Clinic-Config/Dashboard as first-class modules. No
implementation has started.

## Current Module
None implemented yet. Next up: **Module 0 — Foundation**
([docs/requirements/foundation.md](docs/requirements/foundation.md)),
which now also owns the `AuditLog` table + `recordAudit` helper (see
[docs/requirements/audit.md](docs/requirements/audit.md)).

## Implementation Order (see MEMORY.md for why this differs from a naive reading)
```
0. Foundation (incl. AuditLog table + recordAudit helper)
1. Staff & Auth
2. Clinic Config
3. Patients
4. Attachments
5. Appointments
6. Visits
7. Treatments
8. Prescriptions
9. Billing / Payments
10. Audit (ADMIN query API)
11. Notifications
12. Dashboard
13. Inventory   ┐ lower priority / future — not in the current
14. Reporting   ┘ master prompt's explicit staff-app list
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
| [.claude/skills/session-start/SKILL.md](.claude/skills/session-start/SKILL.md) | Run this first (`/session-start`) — session orientation checklist |
| [MEMORY.md](MEMORY.md) | Durable decisions & discoveries |
| [progress-tracker.md](progress-tracker.md) | Current execution state |
| [ai-workflow-rules.md](ai-workflow-rules.md) | How agents must operate |
| [architecture-context.md](architecture-context.md) | Detailed architecture |
| [code-standards.md](code-standards.md) | Engineering conventions |
| [project-overview.md](project-overview.md) | Product scope, actors, non-goals |
| [ui-context.md](ui-context.md) | Frontend/backend contract |
| `docs/requirements/*.md` | Per-module specs |
| [docs/business-rules.md](docs/business-rules.md) | Domain-rule catalogue, RESOLVED/UNRESOLVED |

## Current Blockers
None. Awaiting decision to start Module 0 (Foundation) implementation task.
