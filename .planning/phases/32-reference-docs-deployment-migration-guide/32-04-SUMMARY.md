---
phase: 32-reference-docs-deployment-migration-guide
plan: 04
subsystem: docs
tags: [docs, deployment, how-to, mkdocs]
requires:
  - "32-01: Python API reference pages (cross-link target)"
  - "32-02: HTTP API reference page (cross-link target)"
  - "32-03: Configuration reference page (cross-link target)"
provides:
  - "Deployment how-to section covering all 5 supported modes"
  - "Runnable command blocks for local-dev, docker-compose, standalone, embedded, and with-regulus"
affects:
  - "docs/how-to/deployment/ (new directory)"
tech-stack:
  added: []
  patterns:
    - "Diátaxis how-to quadrant: one landing page + one page per deployment mode"
    - "Relative markdown links into reference/ for cross-quadrant navigation"
key-files:
  created:
    - "docs/how-to/deployment/index.md"
    - "docs/how-to/deployment/local-dev.md"
    - "docs/how-to/deployment/docker-compose.md"
    - "docs/how-to/deployment/standalone-service.md"
    - "docs/how-to/deployment/embedded-library.md"
    - "docs/how-to/deployment/with-regulus.md"
  modified: []
key-decisions:
  - "Documented the uvicorn factory pattern (app_factory + --factory) for standalone mode since zeroth.core.service.entrypoint exposes a factory rather than a module-level app"
  - "Pinned documented econ-instrumentation-sdk constraint to >=0.1.1 to match pyproject.toml rather than the stricter number in the planning context"
  - "Embedded-library page uses bootstrap_service + create_database directly rather than a build_bootstrap shortcut (the latter does not exist in the codebase)"
requirements-completed:
  - DOCS-10
duration: "~15 min"
completed: "2026-04-12"
---

# Phase 32 Plan 04: Deployment Guide Summary

Authored six deployment how-to pages under `docs/how-to/deployment/` covering the five supported modes plus a landing index. Each mode page is 340-514 words with runnable command blocks, verification steps, common gotchas, and cross-links to the Configuration, HTTP API, and Python API references from earlier Phase 32 plans.

The Deployment Guide closes requirement **DOCS-10**. Users shipping `zeroth-core` now have an end-to-end playbook for every supported deployment model: local hacking, full-stack compose, production single-node, embedded library, and the Regulus economics companion overlay.

## Tasks executed

| Task | Name | Commit |
|------|------|--------|
| 1 | Write deployment index + local-dev + docker-compose pages | `fb8ea14` |
| 2 | Write standalone-service + embedded-library + with-regulus pages | `9ecfb0b` |
| - | Fix broken links surfaced by `mkdocs build` | `88ab806` |

## Pages produced

| Page | Words | Anchors every page links to |
|------|------:|------------------------------|
| `index.md` | 340 | `reference/configuration.md`, `reference/http-api.md`, `reference/python-api.md` |
| `local-dev.md` | 362 | `reference/configuration.md`, `reference/http-api.md`, tutorials/getting-started |
| `docker-compose.md` | 392 | `reference/configuration.md`, `with-regulus.md` |
| `standalone-service.md` | 414 | `reference/configuration.md`, `reference/http-api.md` |
| `embedded-library.md` | 514 | `reference/configuration.md`, `reference/python-api/*` |
| `with-regulus.md` | 420 | `reference/configuration.md`, `reference/python-api/econ.md`, `concepts/econ.md`, `docker-compose.md`, `standalone-service.md` |

## Verification

- `ls docs/how-to/deployment/*.md | wc -l` -> **6** (as required)
- Every page has at least one fenced code block (`bash`, `yaml`, `ini`, `nginx`, or `python`)
- Every page links into `reference/configuration.md` or `reference/python-api/*` (confirmed via `grep -l` — 6/6)
- `uv run mkdocs build` succeeds. Initial run surfaced two broken link warnings (Python API index path + tutorials path) which were fixed in commit `88ab806`. Final build is clean apart from the expected "pages not in nav" notice — nav wiring is deferred to Plan 32-06.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Corrected broken cross-links**
- **Found during:** Verification (`mkdocs build`)
- **Issue:** Initial draft linked to `../../reference/python-api/index.md` and `../../tutorials/getting-started.md`, neither of which exist on disk. The actual targets are `docs/reference/python-api.md` (a landing stub) and `docs/tutorials/getting-started/index.md` (a directory).
- **Fix:** Updated the two links to match real paths.
- **Files modified:** `docs/how-to/deployment/index.md`, `docs/how-to/deployment/local-dev.md`
- **Verification:** Rerun `mkdocs build` produced zero link warnings.
- **Commit:** `88ab806`

**2. [Rule 1 - Bug] Corrected econ SDK version pin**
- **Found during:** Task 2 authoring
- **Issue:** Planner notes referenced `econ-instrumentation-sdk>=0.2.3`, but `pyproject.toml` actually pins `>=0.1.1`. Documentation must match the code.
- **Fix:** Changed the `with-regulus.md` pin references to `>=0.1.1` and tweaked the surrounding context.
- **Files modified:** `docs/how-to/deployment/with-regulus.md`
- **Verification:** `grep "econ-instrumentation-sdk" pyproject.toml` confirms the pin.
- **Commit:** `9ecfb0b`

**3. [Rule 3 - Blocking] Replaced non-existent `build_bootstrap` helper**
- **Found during:** Task 2 authoring (`embedded-library.md`)
- **Issue:** The plan snippet used `from zeroth.core.service.bootstrap import build_bootstrap` but no such symbol exists. The actual public API is `bootstrap_service(database, deployment_ref=...)` plus `create_database(settings)` from `zeroth.core.storage.factory`.
- **Fix:** Rewrote the minimal-pattern code block to use the real helpers (matching `entrypoint.py::_bootstrap()`).
- **Files modified:** `docs/how-to/deployment/embedded-library.md`
- **Commit:** `9ecfb0b`

**Total deviations:** 3 auto-fixed (2 bug, 1 blocking). **Impact:** All three were documentation accuracy fixes — no code behavior changed. The Deployment Guide now matches the actual codebase surface.

## Authentication Gates

None. Documentation-only plan.

## Self-Check: PASSED

- `docs/how-to/deployment/index.md` — FOUND
- `docs/how-to/deployment/local-dev.md` — FOUND
- `docs/how-to/deployment/docker-compose.md` — FOUND
- `docs/how-to/deployment/standalone-service.md` — FOUND
- `docs/how-to/deployment/embedded-library.md` — FOUND
- `docs/how-to/deployment/with-regulus.md` — FOUND
- Commits `fb8ea14`, `9ecfb0b`, `88ab806` — all present in `git log`
- `uv run mkdocs build` — clean (nav-wiring notice is expected, deferred to 32-06)

## Ready for next step

Plan 32-04 complete. Phase 32 Plan 05 (migration guide) already has a SUMMARY on disk (committed earlier); the remaining work is Plan 32-06 (finalize nav + CI gates), which will wire the new deployment pages into `mkdocs.yml`.
