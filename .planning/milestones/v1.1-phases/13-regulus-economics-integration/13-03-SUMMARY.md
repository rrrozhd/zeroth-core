---
phase: 13-regulus-economics-integration
plan: 03
subsystem: api
tags: [fastapi, httpx, regulus, cost-api, rest]

requires:
  - phase: 13-01
    provides: RegulusClient, RegulusSettings, CostEstimator, econ models
provides:
  - Cost REST endpoints (GET /v1/tenants/{id}/cost, GET /v1/deployments/{ref}/cost)
  - ServiceBootstrap wiring for RegulusClient
  - App factory cost route registration
  - Regulus transport flush on shutdown
affects: [13-02, 14-sandbox, 17-deployment]

tech-stack:
  added: []
  patterns: [app.state for per-request Regulus config, conditional BudgetEnforcer import]

key-files:
  created:
    - src/zeroth/service/cost_api.py
    - tests/test_cost_api.py
  modified:
    - src/zeroth/service/app.py
    - src/zeroth/service/bootstrap.py

key-decisions:
  - "BudgetEnforcer import is conditional (try/except ImportError) since Plan 13-02 may not have landed yet"
  - "Cost endpoints use app.state.regulus_base_url rather than bootstrap protocol to keep cost_api decoupled"
  - "budget_enforcer typed as object|None in dataclass to avoid import dependency on not-yet-existing module"

patterns-established:
  - "Regulus state injection: app.state.regulus_base_url/timeout set in create_app when RegulusClient present"
  - "Conditional module import: try/except ImportError for cross-plan dependencies in parallel execution"

requirements-completed: [ECON-04]

duration: 5min
completed: 2026-04-07
---

# Phase 13 Plan 03: Cost REST Endpoints and Bootstrap Wiring Summary

**Cost REST endpoints querying Regulus backend for tenant/deployment spend, with RegulusClient wired into ServiceBootstrap lifecycle**

## Performance

- **Duration:** 306s (~5 min)
- **Started:** 2026-04-07T07:43:53Z
- **Completed:** 2026-04-07T07:49:00Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- GET /v1/tenants/{tenant_id}/cost returns cumulative tenant spend from Regulus backend
- GET /v1/deployments/{deployment_ref}/cost returns deployment-level cost from Regulus backend
- Endpoints return 503 when Regulus is unreachable or not configured
- RegulusClient created in bootstrap_service() when regulus.enabled=True
- Regulus transport flushed on lifespan shutdown (addresses Pitfall 2)
- 6 unit tests covering success and error paths

## Task Commits

Each task was committed atomically:

1. **Task 1: Create cost REST endpoints** - `36bcbf3` (feat) -- TDD: tests + implementation
2. **Task 2: Wire Regulus components into ServiceBootstrap and app factory** - `cad929c` (feat)

## Files Created/Modified

- `src/zeroth/service/cost_api.py` - Cost REST endpoints with TenantCostResponse/DeploymentCostResponse models
- `tests/test_cost_api.py` - 6 unit tests for cost endpoints (success, 503 error, not configured)
- `src/zeroth/service/app.py` - Cost route registration, Regulus state injection, shutdown cleanup
- `src/zeroth/service/bootstrap.py` - regulus_client/budget_enforcer fields, Regulus wiring in bootstrap_service()

## Decisions Made

- BudgetEnforcer import wrapped in try/except ImportError since Plan 13-02 (which creates budget.py) runs in parallel and may not have landed yet
- Cost endpoints access Regulus URL via app.state rather than bootstrap Protocol, keeping cost_api module decoupled from ServiceBootstrap
- budget_enforcer field typed as `object | None` to avoid hard dependency on not-yet-existing BudgetEnforcer class

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Conditional BudgetEnforcer import for parallel plan execution**
- **Found during:** Task 2 (bootstrap wiring)
- **Issue:** Plan references `from zeroth.econ.budget import BudgetEnforcer` but budget.py does not exist yet (created by Plan 13-02 which runs in parallel)
- **Fix:** Wrapped BudgetEnforcer import in try/except ImportError; typed field as `object | None`
- **Files modified:** src/zeroth/service/bootstrap.py
- **Verification:** All 111 service/econ/config tests pass; lint clean
- **Committed in:** cad929c (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Necessary to handle parallel plan execution. BudgetEnforcer will activate automatically once Plan 13-02 lands.

## Issues Encountered

- Pre-existing test failures in `tests/live_scenarios/test_research_audit.py` (timeout in e2e scenario) and `tests/secrets/test_data_protection.py` (secret leakage assertion) -- both unrelated to this plan's changes and out of scope.

## User Setup Required

None - no external service configuration required.

## Known Stubs

None - all endpoints wire to real Regulus backend queries.

## Next Phase Readiness

- Cost endpoints ready for use once Regulus backend is running
- BudgetEnforcer will auto-activate when Plan 13-02 creates econ/budget.py
- Phase 13 integration complete pending Plan 13-02 (budget enforcement)

---
*Phase: 13-regulus-economics-integration*
*Completed: 2026-04-07*
