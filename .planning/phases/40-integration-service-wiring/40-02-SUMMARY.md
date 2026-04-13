---
phase: 40-integration-service-wiring
plan: 02
subsystem: service-api
tags: [rest-api, artifact, template, openapi]
dependency_graph:
  requires: [40-01]
  provides: [artifact-api, template-api, openapi-v4-endpoints]
  affects: [service-app, authorization, dump-openapi]
tech_stack:
  added: []
  patterns: [register_*_routes pattern, _extract_from_bootstrap helper pattern]
key_files:
  created:
    - src/zeroth/core/service/artifact_api.py
    - src/zeroth/core/service/template_api.py
    - tests/service/test_artifact_api.py
    - tests/service/test_template_api.py
  modified:
    - src/zeroth/core/service/app.py
    - src/zeroth/core/service/authorization.py
    - scripts/dump_openapi.py
    - openapi/zeroth-core-openapi.json
decisions:
  - Used TEMPLATE_ADMIN permission instead of nonexistent ADMIN permission for template write endpoints
  - DELETE /templates/{name}/{version} implemented via direct dict manipulation since TemplateRegistry has no delete method
  - Route registration moved to Task 1 (from Task 2) since tests require routes to be wired
metrics:
  duration: 6m
  completed: 2026-04-13
---

# Phase 40 Plan 02: Artifact & Template REST API Endpoints Summary

Artifact retrieval and template CRUD REST endpoints with full route registration, OpenAPI spec update, and 12 passing API tests.

## Tasks Completed

### Task 1: Create artifact and template API route modules with tests (TDD)

**RED phase:** Wrote 12 failing tests covering artifact GET (200/404/503) and template CRUD (list/create/get/get-version/delete/404/409/503).

**GREEN phase:** Implemented `artifact_api.py` with `register_artifact_routes` (GET /artifacts/{artifact_id}) and `template_api.py` with `register_template_routes` (GET/POST/DELETE /templates). Both follow the canonical `webhook_api.py` pattern with `_extract_from_bootstrap` helpers returning 503 when subsystem is not configured.

**Commits:**
- `9c071cd` test(40-02): add failing tests for artifact and template API endpoints
- `63edfb4` feat(40-02): artifact retrieval and template CRUD API endpoints

### Task 2: Register routes in app.py, update dump_openapi.py stub, regenerate OpenAPI spec

Added `artifact_store=None` and `template_registry=None` to the `dump_openapi.py` SimpleNamespace stub. Regenerated the OpenAPI spec which now includes four new path entries: `/v1/artifacts/{artifact_id}`, `/v1/templates`, `/v1/templates/{name}`, `/v1/templates/{name}/{version}`. Drift check passes. All 18 tests pass (12 new + 6 existing app tests).

**Commit:**
- `4b8b2ee` chore(40-02): update OpenAPI stub and regenerate spec with artifact/template endpoints

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Route registration moved from Task 2 to Task 1**
- **Found during:** Task 1 GREEN phase
- **Issue:** Tests require routes to be registered on the v1_router to function; without wiring in app.py, all tests return 404
- **Fix:** Added route imports and registrations to app.py in Task 1 instead of Task 2
- **Files modified:** src/zeroth/core/service/app.py

**2. [Rule 3 - Blocking] TEMPLATE_ADMIN permission added to authorization enum**
- **Found during:** Task 1 GREEN phase
- **Issue:** Plan references `Permission.ADMIN` which does not exist in the Permission enum; template write endpoints need a valid permission value
- **Fix:** Added `TEMPLATE_ADMIN = "template:admin"` to Permission enum; ADMIN role automatically includes it via `set(Permission)`
- **Files modified:** src/zeroth/core/service/authorization.py

## Verification Results

```
uv run pytest tests/service/test_artifact_api.py tests/service/test_template_api.py tests/service/test_app.py -v
# 18 passed in 1.90s

uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json
# OK: openapi/zeroth-core-openapi.json is up to date.

uv run ruff check src/zeroth/core/service/artifact_api.py src/zeroth/core/service/template_api.py
# All checks passed!
```

## Self-Check: PASSED

All 6 created/modified files exist. All 3 commit hashes verified.
