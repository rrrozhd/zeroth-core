---
phase: 16-distributed-dispatch-horizontal-scaling
plan: 02
subsystem: dispatch
tags: [arq, redis, wakeup, graceful-shutdown, worker]

requires:
  - phase: 16-01
    provides: "DispatchSettings, LeaseManager, RunWorker foundation"
provides:
  - "ARQ wakeup module (pool factory, enqueue, consumer coroutine)"
  - "RunWorker wakeup handler for immediate claim on signal"
  - "RunWorker graceful shutdown with lease release to PENDING"
affects: [16-03, deployment, worker-scaling]

tech-stack:
  added: [arq>=0.26]
  patterns: [fire-and-forget-enqueue, wakeup-signal-not-queue, graceful-shutdown-lease-release]

key-files:
  created:
    - src/zeroth/dispatch/arq_wakeup.py
    - tests/dispatch/test_arq_wakeup.py
  modified:
    - src/zeroth/dispatch/__init__.py
    - src/zeroth/dispatch/worker.py
    - tests/dispatch/test_worker.py
    - pyproject.toml

key-decisions:
  - "ARQ wakeup is fire-and-forget: enqueue_wakeup never raises, logs on failure"
  - "RunWorker._release_to_pending uses synchronous repo calls matching existing sync RunRepository pattern"
  - "ARQ exports guarded by try/except ImportError so dispatch works without arq installed"

patterns-established:
  - "Wakeup signal pattern: ARQ job is just a notification, authoritative claim always from lease store"
  - "Graceful shutdown: set stopping flag, wait for in-flight, release remaining leases to PENDING"

requirements-completed: [OPS-04, OPS-05]

duration: 4min
completed: 2026-04-07
---

# Phase 16 Plan 02: ARQ Wakeup and Worker Graceful Shutdown Summary

**ARQ wakeup module with fire-and-forget enqueue and RunWorker graceful shutdown releasing leases to PENDING on SIGTERM**

## Performance

- **Duration:** 4 min (218s)
- **Started:** 2026-04-07T08:38:11Z
- **Completed:** 2026-04-07T08:42:09Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Created ARQ wakeup module with pool factory, fire-and-forget enqueue, and consumer coroutine
- Enhanced RunWorker with handle_wakeup() for immediate claim without poll wait
- Added graceful_shutdown() that waits for in-flight tasks then releases remaining leases back to PENDING
- All 28 dispatch tests pass (6 new ARQ + 5 new worker + 17 existing)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create ARQ wakeup module** - `8cab35f` (feat)
2. **Task 2: Enhance RunWorker with wakeup handler and graceful shutdown** - `d9d9858` (feat)

## Files Created/Modified
- `src/zeroth/dispatch/arq_wakeup.py` - ARQ pool factory, wakeup enqueue, consumer coroutine
- `src/zeroth/dispatch/__init__.py` - Added ARQ exports with ImportError guard
- `src/zeroth/dispatch/worker.py` - handle_wakeup, graceful_shutdown, _stopping flag, shutdown_timeout
- `tests/dispatch/test_arq_wakeup.py` - 6 tests for ARQ wakeup module
- `tests/dispatch/test_worker.py` - 5 new tests for wakeup handler, shutdown, stopping flag
- `pyproject.toml` - Added arq>=0.26 dependency

## Decisions Made
- ARQ wakeup enqueue is fire-and-forget (never raises, logs debug on failure) per D-05
- RunWorker uses synchronous repo calls in _release_to_pending matching existing sync RunRepository pattern (plan had async `await` calls -- fixed as Rule 1 deviation)
- ARQ exports in __init__.py guarded by try/except ImportError so dispatch module works without arq installed

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed async/sync mismatch in _release_to_pending**
- **Found during:** Task 2 (graceful shutdown implementation)
- **Issue:** Plan specified `await self.run_repository.get(run_id)` and `await self.run_repository.put(run)` but RunRepository methods are synchronous
- **Fix:** Made _release_to_pending synchronous, calling repo methods without await
- **Files modified:** src/zeroth/dispatch/worker.py
- **Verification:** All tests pass, ruff check clean
- **Committed in:** d9d9858 (Task 2 commit)

---

**Total deviations:** 1 auto-fixed (1 bug)
**Impact on plan:** Essential fix for correctness. No scope creep.

## Issues Encountered
None

## Known Stubs
None - all functions are fully implemented with real logic.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- ARQ wakeup module ready for Plan 03 to integrate with app lifecycle
- Worker graceful shutdown ready for SIGTERM handler wiring in deployment
- All dispatch tests green (28/28)

---
*Phase: 16-distributed-dispatch-horizontal-scaling*
*Completed: 2026-04-07*

## Self-Check: PASSED

All 5 source/test files verified present. Both commit hashes (8cab35f, d9d9858) confirmed in git log.
