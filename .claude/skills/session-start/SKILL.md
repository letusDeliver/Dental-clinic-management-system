---
name: session-start
description: Mandatory orientation checklist for the start of any session in this repo (Dental Clinic Management System) — read the planning docs, find the current module and its requirement doc, and read only the relevant slice of architecture/code-standards before touching code or docs. Use this before starting any task here, and always before writing implementation code.
---

# Session Start

This repo is documentation-first and agent-driven (see [CLAUDE.md](../../../CLAUDE.md)).
Do not start implementing, planning, or editing docs until this checklist is
done. It mirrors the mandatory "Session Start" section in
[ai-workflow-rules.md](../../../ai-workflow-rules.md) — that file is the
source of truth if the two ever drift; update this skill to match it, not
the other way around.

## Steps, in this exact order

1. Read [CLAUDE.md](../../../CLAUDE.md) — project purpose, stack, current
   phase, current module, implementation order, critical rules.
2. Read [MEMORY.md](../../../MEMORY.md) — durable decisions. Newest entries
   first; do not re-litigate anything already decided there.
3. Read [progress-tracker.md](../../../progress-tracker.md) — current
   module/task/status, open questions, blockers, what's actually done vs.
   just planned.
4. From step 1/3, identify the current module and read its requirement doc
   in [docs/requirements/](../../../docs/requirements/). If no module is
   in progress, the "Next Step" in progress-tracker.md says which one is
   next — do not assume it's the first one alphabetically or by number.
5. Read only the sections of
   [architecture-context.md](../../../architecture-context.md) and
   [code-standards.md](../../../code-standards.md) relevant to that module
   — not the whole file.
6. If any implementation already exists (`backend/src/`, migrations,
   tests) for the current module, inspect it before writing or changing
   anything. Code/schema/migrations outrank docs if they disagree — see
   the Source-of-Truth Hierarchy in ai-workflow-rules.md. Don't assume the
   doc is right and the code is wrong; investigate and reconcile.
7. Do not explore the rest of the repository beyond this. If something
   looks inconsistent, investigate that specific inconsistency only —
   don't turn orientation into a full audit.

## After orientation

State briefly (a few lines, not a report): current phase, current module,
and what you're about to do. Then proceed under the normal
[ai-workflow-rules.md](../../../ai-workflow-rules.md) workflow
(READ → UNDERSTAND → CHECK DEPENDENCIES → PLAN → IMPLEMENT → TEST → REVIEW
→ VERIFY → DOCUMENT → UPDATE PROGRESS).

## Do not

- Do not create a new giant requirement doc or bypass the per-module doc
  split described in CLAUDE.md/project-overview.md.
- Do not start implementing a module whose requirement doc you have not
  read this session, even if you recall it from a prior session.
- Do not mark anything `VERIFIED` without satisfying the full Module
  Completion Gate in ai-workflow-rules.md.
- Do not skip straight to code changes on the strength of this checklist
  alone if progress-tracker.md's "Blockers" or "Open Questions" section
  flags something relevant to the task at hand — surface it first.
