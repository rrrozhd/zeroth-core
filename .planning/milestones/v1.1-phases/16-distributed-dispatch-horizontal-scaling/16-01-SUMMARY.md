---
phase: 16-distributed-dispatch-horizontal-scaling
plan: 01
subsystem: dispatch
tags: [postgres, skip-locked, leasing, horizontal-scaling, alembic]

# Dependency graph
requires:
  - phase: 11-config-postgres-storage
    provides: AsyncPostgresDatabase, AsyncDatabase protocol, Alembic migration infrastructure
  - phase: 15-webhooks-approval-sla
    provides: Migration 003 (down_revision chain)
provides:
  - DispatchSettings with arq_enabled, shutdown_timeout, poll_interval
  - Backend-conditional LeaseManager with Postgres SKIP LOCKED and SQLite timestamp-expiry paths
  - Alembic migration 004 with ix_runs_pending_claim partial index
affects: [16-02, 16-03, worker, deployment]

# Tech tracking
tech-stack:
  added: []
  patterns: [backend-conditional dispatch with isinstance detection, try/except ImportError guard for optional dependencies]

key-files:
  created:
    - src/zeroth/config/settings.py (DispatchSettings added)
    - src/zeroth/migrations/versions/004_add_skip_locked_index.py
  modified:
    - src/zeroth/dispatch/lease.py

key-decisions:
  - "Synchronous claim_pending interface preserved for backward compatibility; Postgres path bridges async via thread pool when inside running event loop"
  - "Conditional import with _HAS_PG guard allows deployment without psycopg installed"

patterns-established:
  - "Backend-conditional dispatch: isinstance(db, AsyncPostgresDatabase) for Postgres-specific SQL"
  - "try/except ImportError with _HAS_PG sentinel for optional Postgres dependency"

requirements-completed: [OPS-04]

# Metrics
duration: 292s
completed: 2026-04-07
---

# Phase 16 Plan 01: SKIP LOCKED Dispatch and DispatchSettings Summary

**Backend-conditional LeaseManager with Postgres FOR UPDATE SKIP LOCKED claiming, DispatchSettings config, and partial index migration for multi-worker horizontal scaling**

## Performance

- **Duration:** 292s (~5 min)
- **Started:** 2026-04-07T16:49:46Z
- **Completed:** 2026-04-07T16:54:38Z
- **Tasks:** 2
- **Files modified:** 5

## Accomplishments
- Added DispatchSettings to ZerothSettings with arq_enabled, shutdown_timeout, and poll_interval fields
- Made LeaseManager backend-aware: Postgres uses SELECT FOR UPDATE SKIP LOCKED (no verify step), SQLite retains existing timestamp-expiry UPDATE pattern
- Created Alembic migration 004 adding ix_runs_pending_claim partial index on runs(deployment_ref, status, started_at) WHERE status='pending'
- Added comprehensive tests covering backend detection, SQLite fallback, and mocked Postgres claiming path

## Task Commits

Each task was committed atomically:

1. **Task 1: Add DispatchSettings and backend-conditional SKIP LOCKED claiming** - `0fab2d9` (feat)
2. **Task 2: Add Alembic migration for SKIP LOCKED partial index and write tests** - `9d0d12f` (feat)

## Files Created/Modified
- `src/zeroth/config/__init__.py` - Package init for config module
- `src/zeroth/config/settings.py` - Added DispatchSettings class with arq_enabled, shutdown_timeout, poll_interval; added dispatch field to ZerothSettings
- `src/zeroth/dispatch/lease.py` - Backend-conditional claim_pending dispatching to _claim_pending_pg (SKIP LOCKED) or _claim_pending_sqlite (timestamp-expiry); _is_postgres() detection; conditional AsyncPostgresDatabase import
- `src/zeroth/migrations/versions/004_add_skip_locked_index.py` - Alembic migration creating partial index ix_runs_pending_claim
- `tests/dispatch/test_lease.py` - Added 7 new tests for backend detection, SQLite fallback, and Postgres mocked claiming path

## Decisions Made
- Preserved synchronous claim_pending interface for backward compatibility with existing RunWorker; Postgres async path bridged via ThreadPoolExecutor when inside a running event loop
- Conditional import of AsyncPostgresDatabase with _HAS_PG sentinel allows deployment without psycopg installed
- Postgres tests use @pytest.mark.skipif(not _HAS_PG) to gracefully skip when psycopg unavailable
- patch.object targets LeaseManager class (not instance) due to slots=True preventing instance-level patching

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed mock patching for slots=True dataclass**
- **Found during:** Task 2 (test writing)
- **Issue:** LeaseManager uses slots=True which prevents instance-level mock patching; tests failed with "read-only" AttributeError
- **Fix:** Changed patch.object to target LeaseManager class instead of instance
- **Files modified:** tests/dispatch/test_lease.py
- **Verification:** All 16 tests collected, 12 passed, 4 skipped (psycopg not installed)
- **Committed in:** 9d0d12f (Task 2 commit)

**2. [Rule 3 - Blocking] Created config package directory**
- **Found during:** Task 1 (settings modification)
- **Issue:** Worktree did not have src/zeroth/config/ directory (exists on main but not in worktree)
- **Fix:** Created config/__init__.py and settings.py with DispatchSettings added
- **Files modified:** src/zeroth/config/__init__.py, src/zeroth/config/settings.py
- **Verification:** grep confirms all expected classes and fields present
- **Committed in:** 0fab2d9 (Task 1 commit)

---

**Total deviations:** 2 auto-fixed (1 bug, 1 blocking)
**Impact on plan:** Both auto-fixes necessary for correctness in parallel worktree execution. No scope creep.

## Issues Encountered
- Worktree based on old commit missing config/ and migrations/ directories; files created from scratch matching main repo structure
- Python import verification had to use PYTHONPATH override since worktree not installed as editable package

## Known Stubs
None - all data paths are wired to real implementations.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- LeaseManager is now backend-aware, ready for 16-02 (ARQ wakeup integration) and 16-03 (worker scaling)
- ix_runs_pending_claim partial index ready for Postgres deployments
- DispatchSettings provides configuration hooks for ARQ and shutdown behavior

## Self-Check: PASSED

- All 6 files verified present on disk
- Commits 0fab2d9 and 9d0d12f verified in git log
- 16 tests collected (12 passed, 4 skipped due to missing psycopg)

---
*Phase: 16-distributed-dispatch-horizontal-scaling*
*Completed: 2026-04-07*
