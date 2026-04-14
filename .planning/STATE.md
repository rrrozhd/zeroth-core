---
gsd_state_version: 1.0
milestone: v4.1
milestone_name: Platform Hardening & Missing Implementations
status: executing
stopped_at: Phase 43 context gathered
last_updated: "2026-04-14T20:54:00.454Z"
last_activity: 2026-04-14 -- Phase 43 planning complete
progress:
  total_phases: 4
  completed_phases: 0
  total_plans: 2
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-04-13)

**Core value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.
**Current focus:** Roadmap is defined for v4.1; Phase 43 is ready for discussion and planning.

## Current Position

Phase: 43 — Parallel Subgraph Fan-Out & Merge Strategies
Plan: —
Status: Ready to execute
Last activity: 2026-04-14 -- Phase 43 planning complete

## Performance Metrics

**Velocity (v1.1):**

- Total plans completed: 40
- Phases: 11 (Phases 11-21)
- Timeline: 4 days (2026-04-06 to 2026-04-09)
- Commits: 168
- Files changed: 350 (+47,444 / -3,273 lines)

## Accumulated Context

### Roadmap Evolution

- Phase 27 added: Ship Zeroth as pip-installable library (zeroth-core)
- Phase 27 completed: archive, namespace rename, CI/docstring gate, and post-rename verification all recorded in checked-in artifacts
- Phases 43-46 added for v4.1: orchestration composition, cloud artifacts, template persistence, and circuit-breaker durability

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
- [Phase 29]: Use SimpleNamespace stub bootstrap inside scripts/dump_openapi.py so OpenAPI generation needs no DB or secrets
- [Phase 29]: Filter-repo produced 29 commits (not 32); planner baseline included tests/test_studio_api.py which is excluded from this filter
- [Phase 29]: CI drift gate uses git diff --exit-code after generate:api; drift step last so earlier failures surface first
- [Phase 29]: Added --passWithNoTests to vitest run; apps/studio has no test files yet
- [Phase 30]: Place tutorial helpers under src/zeroth/core/examples/ (installed package) instead of top-level examples/ so docs snippets can import them against the published wheel
- [Phase 30]: Deploy docs via `mkdocs gh-deploy --force` in a single GHA job rather than peaceiris/actions-gh-pages — matches mkdocs-material upstream recommendation, minimises marketplace trust surface
- [Phase 32]: CI install step uses uv sync --all-extras so OpenAPI drift gate can import zeroth.core.service.app (transitively needs the dispatch/redis extra)

### Pending Todos

- Discuss and plan Phase 43: Parallel Subgraph Fan-Out & Merge Strategies
- Sequence implementation for Phases 44-46 after Phase 43 scope is locked

### Blockers/Concerns

- PyPI trusted-publisher setup for `zeroth-core` requires manual user action — must register the publisher on pypi.org (environment `pypi`) AND test.pypi.org (environment `testpypi`) separately. econ-instrumentation-sdk publishing lives in the Regulus repo and is out of scope for Phase 28.
- Local parent directory `/Users/dondoe/coding/zeroth/` needs to be renamed to `zeroth-archive/`; until then, both repos cannot coexist cleanly at the intended path
- SITE-03 (PR preview deploys) deferred to follow-up phase — GitHub Pages does not natively support PR previews and the user accepted deferral during Phase 30 planning (CONTEXT D-06). Re-open if/when docs move to Cloudflare Pages or Netlify.

## Session Continuity

Last session: 2026-04-14T18:05:25.536Z
Stopped at: Phase 43 context gathered
Resume: Discuss Phase 43, then plan it into executable work
