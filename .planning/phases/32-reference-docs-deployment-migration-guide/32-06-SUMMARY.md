---
phase: 32-reference-docs-deployment-migration-guide
plan: 06
subsystem: docs
tags: [docs, mkdocs, ci, drift-check, nav, phase-finalize]
requires:
  - "32-01: Python API Reference pages (nav targets)"
  - "32-02: HTTP API Swagger UI + --check drift mode + docs/assets mirror"
  - "32-03: scripts/dump_config.py + docs/reference/configuration.md"
  - "32-04: docs/how-to/deployment/* pages"
  - "32-05: docs/how-to/migration-from-monolith.md"
provides:
  - "Final mkdocs.yml nav with Deployment subsection (6 pages) and Migration entry under How-to"
  - "CI drift gates in .github/workflows/docs.yml: OpenAPI --check, config --check, asset parity diff"
  - "Strict mkdocs build preserved as final gate after drift checks"
affects:
  - ".github/workflows/docs.yml runs on every PR — drift surfaces before deploy"
  - "Phase 32 complete; v3.0 milestone finalized"
tech-stack:
  added: []
  patterns:
    - "CI drift gates BEFORE strict build — targeted errors rather than generic build failure"
    - "Strict diff on mirrored asset (openapi/ vs docs/assets/openapi/) forces updates into the PR that regenerated the snapshot"
key-files:
  created:
    - .planning/phases/32-reference-docs-deployment-migration-guide/32-06-SUMMARY.md
  modified:
    - mkdocs.yml
    - .github/workflows/docs.yml
key-decisions:
  - "Install step switched to uv sync --all-extras so the OpenAPI drift check can import zeroth.core.service.app (which transitively requires the dispatch/redis extra) — plan explicitly permitted this fallback"
  - "Drift gates ordered openapi → config → asset parity → strict build so a drift failure surfaces as a targeted error rather than a generic build failure"
  - "Deployment subsection placed after Cookbook (not as a sibling of How-to root) so it nests inside How-to Guides alongside Subsystems and Cookbook"
  - "Migration page added as a flat entry under How-to Guides (not under a Migration: parent) since it is a single comprehensive page"
requirements-completed:
  - DOCS-07
  - DOCS-08
  - DOCS-09
  - DOCS-10
  - DOCS-11
metrics:
  duration: "3 min"
  completed: 2026-04-11
  start: 2026-04-11T21:37:40Z
  end: 2026-04-11T21:40:43Z
  tasks: 3
  files_created: 1
  files_modified: 2
---

# Phase 32 Plan 06: Finalize Nav + CI Drift Gates Summary

Wired the Deployment (6 pages) and Migration (1 page) entries into `mkdocs.yml` nav, extended `.github/workflows/docs.yml` with three drift gates (OpenAPI --check, Configuration --check, asset parity diff) that run before the strict mkdocs build, and verified the full CI equivalent locally end-to-end. This closes Phase 32 and finalizes the v3.0 milestone.

## Tasks Executed

| Task | Name | Commit |
| ---- | ---- | ------ |
| 1 | Wire Deployment + Migration into mkdocs.yml nav | `e0e294a` |
| 2 | Add OpenAPI/config/asset drift gates to docs.yml workflow | `b5af249` |
| 3 | Local CI equivalent + all-extras install fix for drift check | `eb22fe3` |

## What Changed

### Task 1: mkdocs.yml nav

Added under `How-to Guides:` (after `Cookbook:`):

```yaml
      - Deployment:
        - how-to/deployment/index.md
        - Local development: how-to/deployment/local-dev.md
        - Docker Compose: how-to/deployment/docker-compose.md
        - Standalone service: how-to/deployment/standalone-service.md
        - Embedded library: how-to/deployment/embedded-library.md
        - With Regulus: how-to/deployment/with-regulus.md
      - Migration from monolith: how-to/migration-from-monolith.md
```

`Reference:` block was untouched — Phase 32-01 already wired the 20 Python API subsystem pages plus HTTP API and Configuration stubs (replaced by 32-02 and 32-03 content).

`uv run mkdocs build --strict` exits 0 with zero `WARNING:` or `ERROR:` lines from mkdocs itself. The colored `⚠ Warning from the Material for MkDocs team` and `WARNING: MkDocs may break support...` lines are a third-party Material-for-mkdocs propaganda banner about an unrelated "MkDocs 2.0 / ProperDocs" controversy; they are emitted to stderr by the Material plugin but do NOT fail `--strict` and are not real build warnings.

### Task 2: .github/workflows/docs.yml drift gates

Inserted three steps between `Install docs dependencies` and `Build site (strict)`:

```yaml
      - name: Check OpenAPI snapshot is up to date
        run: |
          uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json

      - name: Check Configuration reference is up to date
        run: |
          uv run python scripts/dump_config.py --check --out docs/reference/configuration.md

      - name: Check docs OpenAPI asset mirrors root snapshot
        run: |
          diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
```

The `Build site (strict)` and `Deploy to GitHub Pages` steps are preserved unchanged.

### Task 3: Local CI equivalent verification + install fix

Running the equivalent of the CI sequence locally surfaced a blocker in the default docs-only install: `scripts/dump_openapi.py` imports `zeroth.core.service.app.create_app`, which transitively imports `zeroth.core.service.health` → `from redis.asyncio import from_url`. `redis` lives in the `dispatch` extra, so `uv sync --extra docs` is insufficient for the drift gate to run. Switched the workflow's `Install docs dependencies` step to `uv sync --all-extras` (Plan 32-06 Task 2 explicitly permitted this fallback). After the switch, all five CI commands exit 0:

| Command | Result |
| ------- | ------ |
| `uv sync --all-extras` | OK |
| `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` | `OK: ... is up to date.` |
| `uv run python scripts/dump_config.py --check --out docs/reference/configuration.md` | `OK: ... is up to date` |
| `diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json` | (empty, matches) |
| `uv run mkdocs build --strict` | exit 0, builds to `site/` in 2.59s |

## Built-site Spot Checks

- `ls site/reference/python-api/*/index.html | wc -l` → **20** (all Phase 32-01 pages render)
- `grep -q "swagger-ui-bundle.js" site/reference/http-api/index.html` → PASS
- `test -f site/reference/configuration/index.html` → PASS
- `ls site/how-to/deployment/` → `docker-compose  embedded-library  index.html  local-dev  standalone-service  with-regulus` (6 pages: 1 index + 5 modes)
- `test -f site/how-to/migration-from-monolith/index.html` → PASS
- `test -f site/how-to/deployment/local-dev/index.html` → PASS

## Verification vs Plan's `<verification>` Block

| Check | Result |
| ----- | ------ |
| `uv run mkdocs build --strict` exits 0 | PASS |
| `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` exits 0 | PASS |
| `uv run python scripts/dump_config.py --check --out docs/reference/configuration.md` exits 0 | PASS |
| `diff -u openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json` exits 0 | PASS |
| `.github/workflows/docs.yml` contains all three drift-check steps before the strict build | PASS |
| `site/reference/python-api/graph/index.html` exists | PASS |
| `site/reference/http-api/index.html` exists | PASS |
| `site/reference/configuration/index.html` exists | PASS |
| `site/how-to/deployment/local-dev/index.html` exists | PASS |
| `site/how-to/migration-from-monolith/index.html` exists | PASS |

## Must-Haves Satisfaction

| Truth | Status |
| ----- | ------ |
| mkdocs.yml nav includes 6 deployment pages under How-to → Deployment and migration page under How-to → Migration | PASS |
| `uv run mkdocs build --strict` passes end-to-end with every Phase 32 page reachable from nav | PASS |
| CI runs `scripts/dump_openapi.py --check` and fails on OpenAPI drift | PASS |
| CI runs `scripts/dump_config.py --check` and fails on configuration drift | PASS |
| CI diffs `openapi/` vs `docs/assets/openapi/` to prevent asset drift | PASS |
| Strict mkdocs build step preserved after drift checks | PASS |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Upgraded Install step to `uv sync --all-extras`**
- **Found during:** Task 3 local verification (first `dump_openapi.py --check` run)
- **Issue:** `scripts/dump_openapi.py` imports `zeroth.core.service.app.create_app`, which transitively imports `zeroth.core.service.health`, which has a top-level `from redis.asyncio import from_url`. `redis` is only installed with the `dispatch` extra, so `uv sync --extra docs` leaves CI unable to run the drift gate at all (`ModuleNotFoundError: No module named 'redis'`). Plan 32-02's SUMMARY already documented this as a pre-existing pattern.
- **Fix:** Changed the `Install docs dependencies` step in `.github/workflows/docs.yml` from `uv sync --extra docs` to `uv sync --all-extras`, with a comment explaining why. Plan 32-06 Task 2 explicitly authorized this as the fallback when drift-check imports fail.
- **Files modified:** `.github/workflows/docs.yml`
- **Verification:** Ran `uv sync --all-extras` locally, then `uv run python scripts/dump_openapi.py --check ...` → exit 0 with `OK: ...`. All five CI-equivalent commands pass.
- **Commit:** `eb22fe3`

**Total deviations:** 1 auto-fixed (1 blocking). **Impact:** None on deliverables — the drift gate now actually runs in CI instead of hard-failing on import. The underlying top-level redis import in `zeroth.core.service.health` is a pre-existing code-smell outside this plan's scope; a future refactor could lazy-import it or move it behind a feature flag.

## Authentication Gates

None.

## Known Stubs

None.

## Threat Flags

None. No new network surface, auth paths, file access patterns, or schema changes.

## Deferred Issues

None. Pre-existing `src/` modifications in `git status` are untouched — they predate this plan and are out of scope per the scope boundary.

## Phase 32 Completion

This is the last plan in Phase 32, which is itself the last phase of the v3.0 milestone.

**Requirements closed:** DOCS-07, DOCS-08, DOCS-09, DOCS-10, DOCS-11 — all five Phase 32 requirements completed (each was wired up by the earlier plan but becomes "reachable from nav and gated by CI" only with this plan's merge).

**Plan-by-plan requirements map:**

| Plan | Requirement | Closed by |
| ---- | ----------- | --------- |
| 32-01 | DOCS-07 (Python API Reference) | Content + mkdocstrings plugin; nav already wired in 32-01 |
| 32-02 | DOCS-08 (HTTP API Reference)   | Swagger UI embed + `dump_openapi.py --check`; nav drift gate here |
| 32-03 | DOCS-09 (Configuration Reference) | `dump_config.py` + generated page; drift gate here |
| 32-04 | DOCS-10 (Deployment Guide)     | 6 pages written; **nav wired in this plan** |
| 32-05 | DOCS-11 (Migration Guide)      | 1 page written; **nav wired in this plan** |
| 32-06 | (finalize)                     | CI drift gates + strict build + nav |

## Next Steps

Phase 32 complete. All plans have SUMMARYs. v3.0 milestone (Core Library Extraction, Studio Split & Documentation) ready for final verification. Suggest `/gsd-verify-work 32` → `/gsd-complete-milestone`.

## Self-Check: PASSED

- `mkdocs.yml` — FOUND (Deployment + Migration entries under `How-to Guides:`)
- `.github/workflows/docs.yml` — FOUND (three drift gates + strict build preserved)
- `site/reference/python-api/graph/index.html` — FOUND (built)
- `site/reference/http-api/index.html` — FOUND (built, Swagger UI embedded)
- `site/reference/configuration/index.html` — FOUND (built)
- `site/how-to/deployment/local-dev/index.html` — FOUND (built)
- `site/how-to/migration-from-monolith/index.html` — FOUND (built)
- Commit `e0e294a` — FOUND in `git log` (nav wiring)
- Commit `b5af249` — FOUND in `git log` (drift gates)
- Commit `eb22fe3` — FOUND in `git log` (all-extras install fix)
