---
gsd_state_version: 1.0
milestone: v3.0
milestone_name: Core Library Extraction, Studio Split & Documentation
status: executing
stopped_at: Phase 27 context gathered
last_updated: "2026-04-10T17:52:03.436Z"
last_activity: 2026-04-10
progress:
  total_phases: 8
  completed_phases: 2
  total_plans: 14
  completed_plans: 11
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-10)

**Core value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.
**Current focus:** Phase 27 — ship-zeroth-as-pip-installable-library-zeroth-core

## Current Position

Phase: 27 (ship-zeroth-as-pip-installable-library-zeroth-core) — EXECUTING
Plan: 2 of 4
Status: Ready to execute
Last activity: 2026-04-10

Progress: [░░░░░░░░░░] 0% (v3.0, 6 phases)

## Performance Metrics

**Velocity (v1.1):**

- Total plans completed: 30
- Phases: 11 (Phases 11-21)
- Timeline: 4 days (2026-04-06 to 2026-04-09)
- Commits: 168
- Files changed: 350 (+47,444 / -3,273 lines)

## Accumulated Context

### Roadmap Evolution

- Phase 27 added: Ship Zeroth as pip-installable library (zeroth-core)

### Decisions

See: .planning/PROJECT.md Key Decisions table

Recent (v3.0):

- v3.0 pivot: extract core as pip-installable library before finishing Studio (user needs backend library for embedded use in their own services)
- Take EVERYTHING into `zeroth.core.*` namespace — pure rename, no file-level core/platform split (pragmatic: avoids cascading `__init__.py` breakage)
- PEP 420 namespace package (no top-level `zeroth/__init__.py`) — leaves room for future sibling packages under `zeroth.*`
- Studio moves to separate public repo — independent release cadence, simpler CI
- v2.0 phases 24-26 are not cancelled — they continue in the new `zeroth-studio` repo
- Multi-layer archive strategy (tarball + bare mirror + pushed branches + GitHub archive) to preserve all 36 worktree branches, 2 stashes, and 1 detached-HEAD worktree before any structural change

v2.0 (retained for reference):

- Vue 3 + Vue Flow for Studio (same stack as n8n reference, MIT-licensed)
- n8n as design reference only (SUL license prevents forking)
- REST-only in Phase 22, WebSocket introduced in Phase 24
- [Phase 22]: Used TS 5.8/vue-tsc 2.2/vitest 3.1 (latest stable)
- [Phase 22]: Modified existing nginx service for Studio rather than adding separate service
- [Phase 23]: Command pattern with 50-item history limit for undo/redo
- [Phase 23]: Compound command pattern for auto-layout
- [Phase 23]: Palette replaces workflow list conditionally using editor mode + workflow loaded check
- [Phase 23]: Used inject/provide for cross-component validation state

### Pending Todos

- Requirements & roadmap for v3.0 (in progress — via /gsd:new-milestone)
- Execution continuation: Phase 1 of the split is partially done in /tmp/zeroth-split/zeroth-core-build (filter-repo rename + codemod + pyproject + verified test collection). After roadmap approval, this work maps onto the first phase(s) of v3.0.
- Decide: rename ad-hoc `pre-split-head` branch commits into formal phase artifacts, or treat scratch as throwaway and rebuild inside the real phase execution.

### Blockers/Concerns

- Regulus has no GitHub remote yet — blocks publishing `econ-instrumentation-sdk` to PyPI, which blocks a clean `zeroth-core` dependency declaration
- PyPI trusted publisher setup for `zeroth-core` and `econ-instrumentation-sdk` requires manual user action on pypi.org
- Local parent directory `/Users/dondoe/coding/zeroth/` needs to be renamed to `zeroth-archive/`; until then, both repos cannot coexist cleanly at the intended path

## Session Continuity

Last session: 2026-04-10T15:42:16.826Z
Stopped at: Phase 27 context gathered
Resume: Continue /gsd:new-milestone workflow → requirements → roadmap → execution
