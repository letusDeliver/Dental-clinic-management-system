# ai-workflow-rules.md — How Development Agents Must Operate

This file defines the mandatory workflow for any agent implementing a
module in this repository. It applies to every task, regardless of size.

## Required Workflow

```
READ → UNDERSTAND → CHECK DEPENDENCIES → PLAN → IMPLEMENT →
TEST → REVIEW → VERIFY → DOCUMENT → UPDATE PROGRESS
```

## Session Start (mandatory, in this order)

Packaged as the `session-start` skill
([.claude/skills/session-start/SKILL.md](.claude/skills/session-start/SKILL.md))
— invoke it (`/session-start`) at the beginning of any session in this repo
instead of re-deriving these steps from memory. The steps below are the
authoritative version; the skill must be kept in sync with this list.

1. Read [CLAUDE.md](CLAUDE.md).
2. Read [MEMORY.md](MEMORY.md).
3. Read [progress-tracker.md](progress-tracker.md).
4. Identify the current module and read its requirement doc in
   `docs/requirements/`.
5. Read only the relevant sections of [architecture-context.md](architecture-context.md)
   and [code-standards.md](code-standards.md) — not the whole project.
6. Inspect existing implementation (source, migrations, tests) relevant to
   the module before writing any code. Do not assume documentation is
   correct if the code disagrees — see Source-of-Truth Hierarchy below.
7. Do **not** explore the entire repository. If the documented state looks
   inconsistent with the code, investigate the specific inconsistency only.

## Source-of-Truth Hierarchy

When determining what is currently true, trust in this order:
```
1. Actual source code
2. Database schema / migrations
3. Current API contracts
4. Current requirement document
5. architecture-context.md
6. progress-tracker.md
7. MEMORY.md
8. Older discussion/history
```
If documentation and implementation disagree: inspect the implementation,
identify the discrepancy, determine the intended behavior, then update the
documentation. Never silently assume the doc is right and the code is
wrong.

## Rules During Implementation

1. Implement only the scope defined in the current requirement document.
2. Respect the contracts published by already-completed modules (see each
   doc's "Cross-Module Contracts" section) — do not change them silently.
3. Never change another module's contract, schema, or API without going
   through Change Control (below).
4. Never silently change the architecture recorded in
   architecture-context.md — propose the change and update that file if
   accepted.
5. Validate assumptions against actual code/schema before relying on them.
6. Follow [code-standards.md](code-standards.md).
7. Add/extend tests for the module's acceptance criteria; run the test
   suite before declaring work done.
8. Run lint/type-check before declaring work done.
9. Review your own diff for scope creep, security issues, and consistency
   with existing patterns before handoff.

## Prohibited Claims & Status Vocabulary

- Never claim tests were run if they were not actually executed in this
  session.
- Never claim implementation is complete if it is only partially complete
  — state precisely what remains.
- Never claim a module is `VERIFIED` without satisfying every item in the
  Module Completion Gate below.
- Never say "production ready" without validating the specific
  production-readiness criteria that claim implies (see Module
  Completion Gate) — name which criteria were checked.

Use these exact status words when describing work, and do not blur them:
```
PLANNED      — written into a requirement/architecture doc, no code yet
DESIGNED     — data model/API/flow worked out in detail, still no code
IMPLEMENTED  — code written, not yet verified by tests
TESTED       — tests written and actually executed, with a result
VERIFIED     — passes the full Module Completion Gate
BLOCKED      — cannot proceed; state exactly what's blocking it
```

## Module Completion Gate

A module is complete only when **all** of the following hold:
```
Requirement implemented
Database migration created and verified (applies cleanly, matches schema intent)
API implemented per the requirement doc
Validation implemented (request schemas)
Authorization implemented and enforced server-side
Error handling implemented (consistent envelope, correct status codes)
Tests written for acceptance criteria
Tests passing
Lint/type checks passing
Security-sensitive paths reviewed (auth, PII, injection, authz bypass)
No unintended scope added beyond the requirement doc
Existing tests for other modules still pass
Documentation updated (this file's outputs: MEMORY.md, progress-tracker.md, CLAUDE.md, architecture-context.md as applicable)
```
Only then mark the module `VERIFIED` in [progress-tracker.md](progress-tracker.md).

## Change Control (cross-module impact discovered mid-task)

If implementing the current module reveals a needed change to another
module's contract, schema, or an architectural decision:

1. Do **not** silently modify the other module.
2. Document what was found and why it matters.
3. Update [architecture-context.md](architecture-context.md) if it is an
   architectural change.
4. Update [MEMORY.md](MEMORY.md) if it is a significant decision/discovery.
5. Update the affected *future* requirement document so the next task
   inherits the change.
6. Record it in [progress-tracker.md](progress-tracker.md) (Open
   Questions / Recent Changes).
7. Continue the current module only if it can proceed safely without the
   other module's change; otherwise flag `CONFLICT` and stop for human
   review rather than guessing.

## Architectural Conflicts

If a genuine conflict is found (two requirement docs imply incompatible
schemas, a requirement is infeasible under the chosen architecture, etc.),
report it as `CONFLICT` with the specific tension stated plainly. Do not
pick a resolution unilaterally when it affects security, data model
stability, or more than one module — surface it for explicit decision, then
record the resolution in architecture-context.md, MEMORY.md, the relevant
requirement doc, and progress-tracker.md.

## End-of-Task Documentation Update (mandatory)

Before ending a task, update:
- [progress-tracker.md](progress-tracker.md) — status, completed items,
  next step, blockers, test results.
- [MEMORY.md](MEMORY.md) — only if a meaningful decision or discovery
  occurred (not routine progress).
- [CLAUDE.md](CLAUDE.md) — only if project-level state changed (current
  module, current phase, blockers).
- [architecture-context.md](architecture-context.md) and/or the relevant
  `docs/requirements/*.md` — only if architecture or a future module's
  contract changed.

Keep every update concise. These files are read at the start of every
future session — bloat here has a permanent token cost.
