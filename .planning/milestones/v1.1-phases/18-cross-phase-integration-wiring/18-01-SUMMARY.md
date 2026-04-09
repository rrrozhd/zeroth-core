---
phase: 18-cross-phase-integration-wiring
plan: 01
subsystem: infra
tags: [dispatch, arq, cost-api, memory, bootstrap, pydantic-settings]

requires:
  - phase: 11-config-postgres-storage
    provides: ZerothSettings unified config model
  - phase: 13-regulus-economics-integration
    provides: CostEstimator and RegulusClient
  - phase: 14-memory-connectors-container-sandbox
    provides: register_memory_connectors factory
  - phase: 16-distributed-dispatch-horizontal-scaling
    provides: ARQ wakeup pool (create_arq_pool)
  - phase: 17-deployment-packaging-operations
    provides: API versioning router with /v1/ prefix
provides:
  - DispatchSettings class with arq_enabled, shutdown_timeout, poll_interval
  - ARQ pool creation in bootstrap when dispatch.arq_enabled is True
  - CostEstimator wired in bootstrap when Regulus is enabled
  - Real ZerothSettings and redis_client passed to memory connector factory
  - Cost API routes without hardcoded /v1/ prefix (compatible with API versioning router)
affects: [18-02, deployment, cost-api, memory, dispatch]

tech-stack:
  added: []
  patterns: [request.app.state for router-compatible state access]

key-files:
  created: []
  modified:
    - src/zeroth/config/settings.py
    - src/zeroth/service/bootstrap.py
    - src/zeroth/service/cost_api.py
    - tests/test_cost_api.py

key-decisions:
  - "Use request.app.state instead of captured app.state for APIRouter compatibility in cost routes"
  - "Redis client creation guarded by mode != disabled check with ImportError fallback"
  - "CostEstimator creation nested inside regulus.enabled block since it requires Regulus"

patterns-established:
  - "request.app.state: Use request.app.state (not captured app variable) when routes may be registered on APIRouter"

requirements-completed: [OPS-04, OPS-05, MEM-01, MEM-06, ECON-04]

duration: 193s
completed: 2026-04-08
---

# Phase 18 Plan 01: Cross-Phase Integration Wiring Summary

**DispatchSettings wired into bootstrap with ARQ pool, CostEstimator, real memory settings, and cost API double-prefix fix**

## Performance

- **Duration:** 193s
- **Started:** 2026-04-08T08:47:17Z
- **Completed:** 2026-04-08T08:50:30Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments
- Added DispatchSettings class to ZerothSettings with arq_enabled, shutdown_timeout, poll_interval fields
- Wired ARQ pool creation, CostEstimator, and real ZerothSettings into bootstrap_service()
- Fixed cost API double /v1/v1/ prefix by removing hardcoded /v1/ from route decorators
- Updated tests to use APIRouter(prefix="/v1") matching production mounting pattern

## Task Commits

Each task was committed atomically:

1. **Task 1: Add DispatchSettings and wire bootstrap** - `52ed53c` (feat)
2. **Task 2: Fix cost API double prefix and update tests** - `b9ce5f2` (fix)

## Files Created/Modified
- `src/zeroth/config/settings.py` - Added DispatchSettings class and dispatch field on ZerothSettings
- `src/zeroth/service/bootstrap.py` - Added cost_estimator, arq_pool, redis_client fields; ARQ pool creation; real memory settings
- `src/zeroth/service/cost_api.py` - Removed /v1/ prefix from route decorators; use request.app.state
- `tests/test_cost_api.py` - Updated to use APIRouter(prefix="/v1") for route mounting

## Decisions Made
- Used request.app.state instead of captured app.state closure variable for APIRouter compatibility (Rule 1 bug fix)
- Redis client creation guarded by settings.redis.mode != "disabled" with ImportError fallback for missing redis package
- CostEstimator creation placed inside regulus.enabled block since cost estimation requires Regulus backend

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed app.state access for APIRouter compatibility**
- **Found during:** Task 2 (cost API prefix fix)
- **Issue:** Route handlers used `getattr(app.state, ...)` where `app` is the captured outer variable. When registering on APIRouter (not FastAPI), APIRouter has no `.state` attribute, causing AttributeError
- **Fix:** Changed to `request.app.state` which always references the root FastAPI application
- **Files modified:** src/zeroth/service/cost_api.py
- **Verification:** All 6 cost API tests pass
- **Committed in:** b9ce5f2 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential for correctness when routes are mounted via APIRouter. No scope creep.

## Issues Encountered
None

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- All 5 of 6 worktree merge gaps closed by this plan
- Ready for Plan 18-02 (async bootstrap + lifespan wiring)
- dispatch.arq_enabled can be set via ZEROTH_DISPATCH__ARQ_ENABLED env var

---
*Phase: 18-cross-phase-integration-wiring*
*Completed: 2026-04-08*
