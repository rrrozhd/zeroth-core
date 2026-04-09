---
phase: 20-bootstrap-integration-wiring
plan: 01
subsystem: orchestrator
tags: [memory, budget-enforcement, dependency-injection, dispatch-wiring]

# Dependency graph
requires:
  - phase: 13-regulus-economics-integration
    provides: BudgetEnforcer for pre-execution budget checks
  - phase: 14-memory-connectors-container-sandbox
    provides: MemoryConnectorResolver, InMemoryConnectorRegistry, register_memory_connectors
  - phase: 18-cross-phase-integration-wiring
    provides: Bootstrap wiring patterns, memory registry population at startup
provides:
  - RuntimeOrchestrator dispatch-time injection of memory_resolver and budget_enforcer onto AgentRunner
  - Bootstrap creates MemoryConnectorResolver from populated registry and wires into orchestrator
  - Budget enforcement fires pre-execution via orchestrator -> runner injection
affects: [21-gap-closure, runtime-testing, agent-execution]

# Tech tracking
tech-stack:
  added: []
  patterns: [dispatch-time-injection-with-try-finally-restore, orchestrator-field-propagation-to-runner]

key-files:
  created:
    - tests/orchestrator/test_memory_budget_wiring.py
  modified:
    - src/zeroth/orchestrator/runtime.py
    - src/zeroth/service/bootstrap.py

key-decisions:
  - "Used object | None typing for memory_resolver and budget_enforcer on RuntimeOrchestrator to match existing pattern and avoid import-time coupling"
  - "Dispatch-time injection pattern: save originals before try, inject conditionally, restore in finally -- matches existing provider wrapping approach"

patterns-established:
  - "Dispatch-time injection: orchestrator saves runner field originals, injects its own values, restores in finally block to prevent state leakage across dispatches"

requirements-completed: [MEM-01, MEM-02, MEM-03, MEM-04, MEM-05, MEM-06, ECON-03]

# Metrics
duration: 6min
completed: 2026-04-09
---

# Phase 20 Plan 01: Bootstrap Integration Wiring Summary

**MemoryConnectorResolver and BudgetEnforcer wired into RuntimeOrchestrator dispatch path with try/finally state isolation, closing INT-01 and INT-02 integration gaps**

## Performance

- **Duration:** 6 min (367s)
- **Started:** 2026-04-09T09:21:04Z
- **Completed:** 2026-04-09T09:27:11Z
- **Tasks:** 3
- **Files modified:** 3

## Accomplishments
- RuntimeOrchestrator now has memory_resolver and budget_enforcer fields that are injected onto AgentRunner at dispatch time
- bootstrap_service() creates MemoryConnectorResolver from the populated InMemoryConnectorRegistry and wires both resolver and budget_enforcer into the orchestrator
- 4 integration tests verify injection, restoration, None-safety, and exception-safety of the dispatch-time wiring

## Task Commits

Each task was committed atomically:

1. **Task 1: Add orchestrator fields and dispatch-time injection** - `3aafae6` (feat)
2. **Task 2: Wire MemoryConnectorResolver and BudgetEnforcer into orchestrator at bootstrap** - `2c31e00` (feat)
3. **Task 3: Integration tests for dispatch-time memory and budget injection** - `f1ebda9` (test)

## Files Created/Modified
- `src/zeroth/orchestrator/runtime.py` - Added memory_resolver/budget_enforcer fields; dispatch-time injection with try/finally restore in _dispatch_node
- `src/zeroth/service/bootstrap.py` - Import MemoryConnectorResolver; create resolver from registry; wire both into orchestrator; add memory_resolver to ServiceBootstrap
- `tests/orchestrator/test_memory_budget_wiring.py` - 4 integration tests proving dispatch-time injection works correctly

## Decisions Made
- Used `object | None` type for new RuntimeOrchestrator fields to match existing pattern (webhook_service, etc.) and avoid import-time coupling
- Wrapped the entire AgentNode dispatch block in try/finally to ensure originals are restored even on exception -- this is the same pattern used for provider wrapping in the plan's design

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Worktree rebased to include phases 10-19 code**
- **Found during:** Task 1 (reading source files)
- **Issue:** Worktree was based on an old commit (phase 9) missing all code from phases 10-19 that the plan depends on
- **Fix:** Rebased worktree branch to local main which includes all phases through 18
- **Verification:** All expected imports and code patterns present after rebase

**2. [Rule 1 - Bug] Fixed pre-existing line-too-long in bootstrap.py**
- **Found during:** Task 2 (ruff check)
- **Issue:** Line 293 `register_memory_connectors(...)` call exceeded 100-char line limit
- **Fix:** Wrapped arguments across multiple lines
- **Files modified:** src/zeroth/service/bootstrap.py
- **Committed in:** 2c31e00 (Task 2 commit)

---

**Total deviations:** 2 auto-fixed (1 blocking, 1 bug)
**Impact on plan:** Both necessary for execution. No scope creep.

## Issues Encountered
None beyond the deviations noted above.

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - all wiring connects to real components. Memory resolver uses the populated InMemoryConnectorRegistry. Budget enforcer is conditionally created from Regulus settings.

## Next Phase Readiness
- INT-01 and INT-02 integration gaps from v1.1 audit are now closed
- Memory reads/writes in agent execution resolve to real connectors via dispatch-time injection
- Budget enforcement fires pre-execution when a tenant's BudgetEnforcer is available
- All 11 orchestrator tests pass (7 existing + 4 new)
- All 6 bootstrap tests pass

---
*Phase: 20-bootstrap-integration-wiring*
*Completed: 2026-04-09*
