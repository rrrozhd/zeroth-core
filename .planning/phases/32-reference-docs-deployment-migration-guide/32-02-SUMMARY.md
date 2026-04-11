---
phase: 32-reference-docs-deployment-migration-guide
plan: 02
subsystem: docs
tags: [docs, openapi, swagger-ui, drift-check, mkdocs]
requires:
  - openapi/zeroth-core-openapi.json (from Phase 29-01)
  - scripts/dump_openapi.py (from Phase 29-01)
  - mkdocs.yml with attr_list + md_in_html (from Phase 30)
provides:
  - Interactive HTTP API reference page at docs/reference/http-api.md
  - OpenAPI spec mirrored as mkdocs static asset at docs/assets/openapi/zeroth-core-openapi.json
  - Drift-check CLI mode via scripts/dump_openapi.py --check
affects:
  - Phase 32-06 (will wire --check into .github/workflows/docs.yml CI)
  - docs site Reference quadrant (HTTP API stub replaced)
tech-stack:
  added:
    - swagger-ui-dist@5.17.14 (via unpkg CDN, no local npm dep)
  patterns:
    - CDN-loaded Swagger UI embed over neoteroi-mkdocs plugin (one fewer dep, --strict safe)
    - Committed OpenAPI snapshot mirrored into docs_dir so mkdocs serves it
key-files:
  created:
    - docs/assets/openapi/zeroth-core-openapi.json
    - .planning/phases/32-reference-docs-deployment-migration-guide/32-02-SUMMARY.md
  modified:
    - scripts/dump_openapi.py
    - docs/reference/http-api.md
key-decisions:
  - Chose static Swagger UI via unpkg CDN over neoteroi-mkdocs plugin — one fewer dependency, works under mkdocs build --strict, and attr_list + md_in_html already enabled
  - Pinned swagger-ui-dist version to 5.17.14 for reproducibility
  - Mirror openapi/zeroth-core-openapi.json to docs/assets/openapi/ rather than reconfigure mkdocs docs_dir — keeps both versions committed and diffable
requirements-completed:
  - DOCS-08
duration: 2 min
completed: 2026-04-11
---

# Phase 32 Plan 02: HTTP API Reference (Swagger UI) Summary

Rendered the `zeroth-core` FastAPI OpenAPI spec as an interactive Swagger UI page and added a `--check` drift-detection mode to `scripts/dump_openapi.py` so CI can gate stale snapshots.

## Execution Metrics

- **Duration:** ~2 min
- **Start:** 2026-04-11T21:26:17Z
- **End:** 2026-04-11T21:28:14Z
- **Tasks completed:** 2/2
- **Files modified:** 4 (2 created, 2 modified)
- **Commits:** 2 feature commits

## What Changed

### Task 1: `--check` drift mode + asset mirror

Extended `scripts/dump_openapi.py` with a `--check` flag that:

- Requires `--out` to be set (errors otherwise).
- Exits 1 with `DRIFT: {out} does not exist` if the target file is missing.
- Compares freshly generated spec against committed file byte-for-byte; exits 1 with a `DRIFT: ... is stale. Run ... to update.` message if they differ.
- Exits 0 with `OK: {out} is up to date.` on match.

Copied `openapi/zeroth-core-openapi.json` to `docs/assets/openapi/zeroth-core-openapi.json` so mkdocs serves it as a static asset under the Swagger UI embed. Both files are committed and byte-identical; Phase 32-06 will add a CI step to enforce that.

**Commit:** `bccee08`

### Task 2: Swagger UI embed page

Replaced the `TBD` stub at `docs/reference/http-api.md` with a Swagger UI embed:

- Pulls `swagger-ui-dist@5.17.14` CSS + bundle from unpkg.
- Points `SwaggerUIBundle` at `../assets/openapi/zeroth-core-openapi.json` so the served site resolves it correctly from `/reference/http-api/` to `/assets/openapi/zeroth-core-openapi.json`.
- Documents the regenerate workflow (`dump_openapi.py --out ...` + `cp` into `docs/assets/`) and the CI drift gate.
- Cross-links the raw JSON for offline tooling (openapi-typescript, Postman import, ReDoc).

**Commit:** `783bb47`

## Verification

All plan-specified checks pass:

| Check | Result |
|---|---|
| `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` | exit 0, prints `OK: ...` |
| `--check` against stale `/tmp/gsd-stale.json` | exit 1, prints `DRIFT: ...` |
| `uv run mkdocs build --strict` | exit 0 |
| `site/reference/http-api/index.html` contains `swagger-ui` | PASS (`grep -q swagger-ui`) |
| `site/assets/openapi/zeroth-core-openapi.json` exists | PASS |
| `openapi/zeroth-core-openapi.json` byte-equal to `docs/assets/openapi/zeroth-core-openapi.json` | PASS (`diff -q`) |

## Must-Haves Satisfaction

| Truth | Status |
|---|---|
| http-api.md renders interactive Swagger UI from zeroth-core OpenAPI spec | Satisfied — verified in built site/ |
| `--check` exits non-zero when snapshot is stale vs live schema | Satisfied — tested positive + negative cases |
| OpenAPI spec consumed by docs page is the committed snapshot | Satisfied — embed URL points at `docs/assets/openapi/zeroth-core-openapi.json` which is a byte-identical copy of `openapi/zeroth-core-openapi.json` |

## Deviations from Plan

### [Rule 3 - Blocking] Installed dispatch extras to run dump_openapi.py

- **Found during:** Task 1 verification
- **Issue:** `scripts/dump_openapi.py` imports `zeroth.core.service.app.create_app`, which transitively imports `zeroth.core.service.health`, which has a top-level `from redis.asyncio import from_url`. `redis` is an optional dependency gated behind the `dispatch` extra, so `uv run python scripts/dump_openapi.py ...` failed with `ModuleNotFoundError: No module named 'redis'` in the default dev env.
- **Fix:** Ran `uv sync --all-extras` to install optional extras locally. This is a dev-env concern only; no code or dependency manifest changes were needed. The underlying pattern (top-level redis import in a core-service module) is a pre-existing issue outside this plan's scope; it will be handled naturally by CI environments that already install extras, or can be refactored in a later plan.
- **Files modified:** None (environment-only)
- **Verification:** Rerun of `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` returned exit 0 with `OK: ...`
- **Commit:** N/A (env-only)

**Total deviations:** 1 (1 environment blocker resolved)
**Impact:** None on deliverables. Plan executed otherwise exactly as written.

## Authentication Gates

None.

## Known Stubs

None. The Swagger UI embed renders live from a committed snapshot, which is wired end-to-end through mkdocs assets. Drift is gated by `--check` (CI wiring comes in Plan 32-06).

## Success Criteria

DOCS-08 satisfied: HTTP API Reference is rendered from the `zeroth-core` FastAPI OpenAPI spec and published alongside the Python reference on the docs site. Drift detection is available via `scripts/dump_openapi.py --check`; CI enforcement will be wired in Plan 32-06.

## Next Steps

Ready for the next Wave 1 plan (32-03 Configuration Reference) or later waves. Plan 32-06 will wire `scripts/dump_openapi.py --check` into `.github/workflows/docs.yml` and finalize the nav.

## Self-Check: PASSED

- `scripts/dump_openapi.py` — FOUND (modified, `--check` flag present)
- `docs/reference/http-api.md` — FOUND (Swagger UI embed present)
- `docs/assets/openapi/zeroth-core-openapi.json` — FOUND (byte-identical to root snapshot)
- Commit `bccee08` — FOUND in git log
- Commit `783bb47` — FOUND in git log
