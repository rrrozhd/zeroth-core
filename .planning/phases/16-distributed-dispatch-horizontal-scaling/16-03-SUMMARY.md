---
phase: 16-distributed-dispatch-horizontal-scaling
plan: 03
subsystem: dispatch/service
tags: [arq, wakeup, sigterm, graceful-shutdown, bootstrap, lifespan]
dependency_graph:
  requires: [16-01, 16-02]
  provides: [arq-service-wiring, sigterm-graceful-shutdown, wakeup-on-run-create]
  affects: [service-bootstrap, app-lifespan, run-api, approval-api]
tech_stack:
  added: []
  patterns: [signal-handler-event-loop, fire-and-forget-wakeup, contextlib-suppress-cleanup]
key_files:
  created:
    - tests/dispatch/test_integration.py
  modified:
    - pyproject.toml
    - src/zeroth/service/bootstrap.py
    - src/zeroth/service/app.py
    - src/zeroth/service/run_api.py
    - src/zeroth/service/approval_api.py
decisions:
  - "arq>=0.27 version constraint (up from 0.26 in Plan 01)"
  - "settings = get_settings() moved before RunWorker construction so dispatch settings are available"
  - "contextlib.suppress(Exception) for ARQ pool close per ruff SIM105"
metrics:
  duration: 270s
  completed: "2026-04-07"
---

# Phase 16 Plan 03: ARQ Service Wiring and SIGTERM Graceful Shutdown Summary

ARQ wakeup pool created at bootstrap, enqueued after run creation and approval continuation, consumed in lifespan alongside poll loop, with SIGTERM-aware graceful shutdown releasing in-flight leases.

## Task Completion

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add arq dependency and wire ARQ pool into bootstrap and service layer | ae42125 | pyproject.toml, bootstrap.py, run_api.py, approval_api.py |
| 2 | Add SIGTERM handling, ARQ consumer to lifespan, and integration tests | d085fce | app.py, test_integration.py |

## What Was Built

### Task 1: ARQ Pool Bootstrap and Wakeup Injection

- Updated arq dependency from >=0.26 to >=0.27 in pyproject.toml
- Added `arq_pool: object | None = None` field to ServiceBootstrap dataclass
- Wired ARQ pool creation in bootstrap_service when `settings.dispatch.arq_enabled=true`
- Passed `poll_interval` and `shutdown_timeout` from DispatchSettings to RunWorker
- Injected `enqueue_wakeup` after `run_repository.create()` in run_api.py
- Injected `enqueue_wakeup` after `schedule_continuation()` in approval_api.py
- Both wakeup calls guarded by `getattr(bootstrap, "arq_pool", None)` for backward compatibility

### Task 2: SIGTERM Handling, ARQ Consumer, and Integration Tests

- Added ARQ consumer task in app lifespan (started when both worker and arq_pool are present)
- Added SIGTERM/SIGINT signal handlers using asyncio event loop `add_signal_handler`
- Added shutdown watcher task that calls `worker.graceful_shutdown()` on signal
- Added cleanup sequence on lifespan exit: cancel watcher, graceful shutdown, cancel ARQ consumer, close ARQ pool
- Created 5 integration tests covering wakeup enqueue, disabled path, shutdown, consumer startup, and dispatch settings

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Moved settings = get_settings() before RunWorker construction**
- **Found during:** Task 1
- **Issue:** The plan references `settings.dispatch.poll_interval` and `settings.dispatch.shutdown_timeout` in RunWorker construction, but `settings = get_settings()` was defined later (Phase 13 section)
- **Fix:** Moved `settings = get_settings()` call before the worker construction block
- **Files modified:** src/zeroth/service/bootstrap.py
- **Commit:** ae42125

**2. [Rule 1 - Bug] Used contextlib.suppress for ARQ pool close**
- **Found during:** Task 2
- **Issue:** ruff SIM105 flagged try/except/pass pattern for pool close
- **Fix:** Used `contextlib.suppress(Exception)` instead
- **Files modified:** src/zeroth/service/app.py
- **Commit:** d085fce

## Decisions Made

1. **arq>=0.27 constraint**: Updated from 0.26 (Plan 01) to match plan specification; 0.27 is the version installed by uv
2. **Settings hoisting**: Moved `get_settings()` earlier in bootstrap_service to make dispatch settings available before RunWorker construction -- no functional change since the singleton is already loaded
3. **Mock-based integration tests**: Used MagicMock/AsyncMock for lifespan tests to avoid the pre-existing sync/async LeaseManager issue

## Known Issues (Pre-existing)

- LeaseManager from Plan 01 uses synchronous `with self.database.transaction()` but the database is now async (`AsyncSQLiteDatabase`), causing failures in tests/dispatch/test_lease.py and tests/service/ tests that bootstrap a real worker. This is a pre-existing issue from Plans 01/02, not introduced by Plan 03.

## Known Stubs

None -- all wiring is functional and connected to real modules.

## Verification

- `uv run pytest tests/dispatch/test_integration.py -v` -- 5/5 passed
- `uv run pytest tests/dispatch/test_arq_wakeup.py -v` -- 6/6 passed
- `uv run ruff check src/zeroth/service/ src/zeroth/dispatch/ src/zeroth/config/` -- all clean
- `uv run python -c "import arq; print(arq.VERSION)"` -- 0.27.0
