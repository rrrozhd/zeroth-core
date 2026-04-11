---
gsd_state_version: 1.0
milestone: v3.0
milestone_name: Core Library Extraction, Studio Split & Documentation
status: executing
stopped_at: Phase 27 complete
last_updated: "2026-04-11T00:00:00Z"
last_activity: 2026-04-11 -- Phase 28 execution started
progress:
  total_phases: 8
  completed_phases: 3
  total_plans: 17
  completed_plans: 14
  percent: 82
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-10)

**Core value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.
**Current focus:** Phase 28 — pypi-publishing-econ-instrumentation-sdk-zeroth-core

## Current Position

Phase: 28 (pypi-publishing-econ-instrumentation-sdk-zeroth-core) — EXECUTING
Plan: 1 of 3
Status: Executing Phase 28
Last activity: 2026-04-11 -- Phase 28 execution started

Progress: [████░░░░░░] 38% (phases 22, 23, and 27 complete in the current split roadmap)

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
- Phase 27 completed: archive, namespace rename, CI/docstring gate, and post-rename verification all recorded in checked-in artifacts

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

- Plan and execute Phase 28 publication work (`econ-instrumentation-sdk` + `zeroth-core`)
- Complete the manual PyPI trusted-publisher setup for zeroth-core on pypi.org AND test.pypi.org (two separate registrations)

### Blockers/Concerns

- PyPI trusted-publisher setup for `zeroth-core` requires manual user action — must register the publisher on pypi.org (environment `pypi`) AND test.pypi.org (environment `testpypi`) separately. econ-instrumentation-sdk publishing lives in the Regulus repo and is out of scope for Phase 28.
- Local parent directory `/Users/dondoe/coding/zeroth/` needs to be renamed to `zeroth-archive/`; until then, both repos cannot coexist cleanly at the intended path

## Session Continuity

Last session: 2026-04-10T18:32:05Z
Stopped at: Phase 27 complete
Resume: Discuss/plan Phase 28, then prepare the publication prerequisites (PyPI trusted publishers + Regulus remote)
