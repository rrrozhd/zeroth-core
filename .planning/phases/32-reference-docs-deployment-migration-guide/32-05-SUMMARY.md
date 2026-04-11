---
phase: 32-reference-docs-deployment-migration-guide
plan: 05
subsystem: docs
tags: [docs, migration, mkdocs, monolith, zeroth-core]

requires:
  - phase: 27-namespace-rename-zeroth-core
    provides: "PEP 420 zeroth.core namespace layout"
  - phase: 28-pypi-publishing
    provides: "zeroth-core>=0.1.1 on PyPI, econ-instrumentation-sdk>=0.1.1 PyPI pin"
provides:
  - "docs/how-to/migration-from-monolith.md — monolith → zeroth.core migration guide"
affects: [phase-32-nav-finalize, existing-monolith-users]

tech-stack:
  added: []
  patterns:
    - "Migration guide structure: TL;DR, per-topic before/after, troubleshooting"

key-files:
  created:
    - docs/how-to/migration-from-monolith.md
  modified: []

key-decisions:
  - "Grep+sed recipe documented for both macOS (BSD) and Linux (GNU) sed variants"
  - "Environment variables section explicitly documents no-op to preempt user confusion"
  - "LibCST codemod referenced as FUTURE-01 but not shipped; users get sed recipe today"
  - "Docker guidance notes no official image ships yet; users rebuild from their own Dockerfile"

patterns-established:
  - "Diataxis How-to guide with TL;DR + per-topic before/after + Troubleshooting tail"

requirements-completed: [DOCS-11]

duration: 5 min
completed: 2026-04-11
---

# Phase 32 Plan 05: Migration Guide Summary

**Single-page monolith → zeroth.core migration guide with import rewrite sed recipe, econ SDK swap, env var no-op note, and Docker retag guidance.**

## Performance

- **Duration:** ~5 min
- **Tasks:** 1 completed
- **Files created:** 1
- **Word count:** 1113 (target: 800-1600)

## Accomplishments

- Wrote `docs/how-to/migration-from-monolith.md`, a single comprehensive migration guide (~1100 words) walking existing monolith users through the one-time upgrade to `zeroth-core` on PyPI.
- Covered all four required topics with before/after examples: install swap, import rename, econ SDK path swap, env var no-op note, and Docker image retag.
- Provided a runnable grep+sed rename recipe with both macOS (BSD sed) and Linux (GNU sed) variants, plus a verification grep to catch stragglers.
- Added a Troubleshooting section anticipating PEP 420 shadowing, missed imports in stub/config files, duplicate econ SDK installs, and CI wheel caching.
- Cross-linked to `reference/configuration.md` and `deployment/docker-compose.md` (the latter is populated by Plan 32-04; the relative link is correct per the plan's key_links spec).

## Task Commits

1. **Task 1: Write migration-from-monolith.md** — `1c04b0f` (docs)

**Plan metadata:** pending final commit

## Files Created/Modified

- `docs/how-to/migration-from-monolith.md` — New migration guide covering install, imports, econ SDK, env vars, Docker retag, verification, and troubleshooting.

## Verification

Automated verify block from the plan:

- `test -f docs/how-to/migration-from-monolith.md` — PASS
- `grep -q "from zeroth.core"` — PASS
- `grep -q "econ-instrumentation-sdk"` — PASS
- `grep -q "ZEROTH_"` — PASS
- `grep -q "docker build"` — PASS
- Word count >= 800 — PASS (1113)
- Word count <= 1600 — PASS (1113)

Additional checks:

- `uv run mkdocs build` (non-strict) — PASS. New page detected; only warning on the new file is "not included in nav", which is expected — nav wiring is deferred to Plan 32-06 per plan scope.
- Pre-existing mkdocs warnings from Plan 32-04 deployment pages are unrelated and out of scope.

## Must-Haves Check

- [x] Walks an existing monolith user through the switch to `zeroth.core.*`
- [x] Covers all four topics: import rename, econ SDK path swap, env var changes, Docker image retag
- [x] At least one concrete before/after code example per topic (install, imports, econ, env confirmation, Docker)
- [x] Automated grep+sed recipe runnable on macOS and Linux (both variants shown)
- [x] Contains `from zeroth.core`
- [x] Relative link to `deployment/docker-compose.md` present

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## Next Phase Readiness

Ready for Plan 32-06 (finalize nav + CI gates), which will add this page to mkdocs.yml nav under How-to → Migration and run the full `mkdocs build --strict` gate.

## Self-Check: PASSED

- File exists: `docs/how-to/migration-from-monolith.md` — FOUND
- Commit `1c04b0f` — FOUND in `git log`
