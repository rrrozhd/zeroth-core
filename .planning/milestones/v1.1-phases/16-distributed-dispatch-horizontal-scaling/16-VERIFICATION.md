---
phase: 16-distributed-dispatch-horizontal-scaling
verified: 2026-04-07T20:00:00Z
status: passed
score: 3/3 must-haves verified
gaps: []
human_verification:
  - test: "Start two worker processes against a shared Postgres database and submit 10 runs; verify each run executes exactly once"
    expected: "All 10 runs complete with no duplicates across workers"
    why_human: "Requires a live multi-process Postgres deployment to validate true horizontal scaling"
  - test: "Submit a run with arq_enabled=true and observe wall-clock time to first processing step vs poll_interval"
    expected: "Worker begins processing well before the 500ms poll interval elapses"
    why_human: "Requires running Redis/ARQ infrastructure to measure latency improvement"
  - test: "Send SIGTERM to a worker while it has in-flight runs, then verify another worker reclaims them"
    expected: "In-flight runs are released to PENDING and picked up by another worker"
    why_human: "Requires multi-process deployment with signal delivery"
---

# Phase 16: Distributed Dispatch & Horizontal Scaling Verification Report

**Phase Goal:** Multiple worker processes share a Postgres lease store for run ownership, and an ARQ-backed wakeup notification reduces lease poll latency without replacing the database as the authoritative queue.
**Verified:** 2026-04-07T20:00:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Two or more worker processes started against the same Postgres instance each claim disjoint sets of pending runs with no duplicate execution | VERIFIED | `LeaseManager._claim_pending_pg` uses `SELECT ... FOR UPDATE SKIP LOCKED` (lease.py:140) ensuring concurrent workers never see the same row. No verify step needed. Partial index `ix_runs_pending_claim` in migration 004 optimizes the query. |
| 2 | Submitting a run triggers an ARQ wakeup notification that causes a worker to begin processing sooner than the configured poll interval | VERIFIED | `run_api.py:128-132` calls `enqueue_wakeup(arq_pool, persisted.run_id)` after `run_repository.create`. `approval_api.py:149-151` does the same after `schedule_continuation`. `RunWorker.handle_wakeup` (worker.py:279-307) triggers immediate `claim_pending` without waiting for poll. ARQ consumer wired in `app.py:88-101`. |
| 3 | Killing a worker mid-run causes another worker to reclaim the run after the lease expires, with no manual intervention | VERIFIED | `RunWorker.graceful_shutdown` (worker.py:313-339) releases in-flight leases back to PENDING on SIGTERM. `_release_to_pending` (worker.py:349-368) reverts status. For hard-kill scenarios, existing `claim_orphaned` (lease.py:160-212) reclaims RUNNING runs with expired leases on worker startup. Signal handlers in `app.py:103-114`. |

**Score:** 3/3 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/config/settings.py` | DispatchSettings with arq_enabled, shutdown_timeout, poll_interval | VERIFIED | Lines 116-121: `DispatchSettings(BaseModel)` with all three fields. Wired into `ZerothSettings` at line 151. |
| `src/zeroth/dispatch/lease.py` | Backend-conditional claiming with SKIP LOCKED for Postgres | VERIFIED | `_is_postgres()` at line 56, `_claim_pending_pg` at line 124 with `FOR UPDATE SKIP LOCKED` at line 140, `_claim_pending_sqlite` at line 78 with verify step. Conditional import with `_HAS_PG` guard. |
| `src/zeroth/migrations/versions/004_add_skip_locked_index.py` | Alembic migration adding partial index | VERIFIED | `ix_runs_pending_claim` partial index on `runs(deployment_ref, status, started_at) WHERE status = 'pending'`. Revision chain 003->004 correct. |
| `src/zeroth/dispatch/arq_wakeup.py` | ARQ pool factory, wakeup enqueue, consumer coroutine | VERIFIED | 108 lines. Exports: `arq_settings_from_zeroth`, `create_arq_pool`, `enqueue_wakeup`, `run_arq_consumer`, `WAKEUP_TASK_NAME`. `enqueue_wakeup` has try/except that catches all exceptions (never raises). |
| `src/zeroth/dispatch/worker.py` | Enhanced RunWorker with wakeup handler and graceful shutdown | VERIFIED | `handle_wakeup` at line 279, `graceful_shutdown` at line 313, `_release_to_pending` at line 349, `_extract_run_id` at line 341, `_stopping = False` in `__post_init__` at line 63, `while not self._stopping` at line 89, `shutdown_timeout: float = 30.0` at line 58. |
| `src/zeroth/service/bootstrap.py` | ARQ pool creation and wiring into ServiceBootstrap | VERIFIED | `arq_pool: object \| None = None` at line 126. Pool creation when `settings.dispatch.arq_enabled` at lines 312-319. `create_arq_pool` import. `poll_interval` and `shutdown_timeout` passed to RunWorker at lines 228-229. |
| `src/zeroth/service/app.py` | ARQ consumer task in lifespan, SIGTERM signal handler | VERIFIED | ARQ consumer at lines 88-101, SIGTERM/SIGINT handlers at lines 103-114, shutdown watcher at lines 116-125, graceful shutdown in cleanup at line 137, ARQ pool close at lines 146-148. |
| `src/zeroth/service/run_api.py` | ARQ wakeup enqueue after run creation | VERIFIED | Lines 128-132: `enqueue_wakeup(arq_pool, persisted.run_id)` with `getattr(bootstrap, "arq_pool", None)` guard. |
| `src/zeroth/dispatch/__init__.py` | ARQ exports with ImportError guard | VERIFIED | Lines 8-25: try/except ImportError wrapping ARQ function imports. |
| `pyproject.toml` | arq dependency | VERIFIED | Line 33: `"arq>=0.27"`. |
| `tests/dispatch/test_lease.py` | Tests for backend-conditional claiming | VERIFIED | 16 tests including `test_claim_pending_pg_uses_skip_locked`, `test_is_postgres_detection_*`, `test_claim_pending_sqlite_fallback`. All pass. |
| `tests/dispatch/test_arq_wakeup.py` | Tests for ARQ wakeup module | VERIFIED | 6 tests including `test_enqueue_wakeup_swallows_exception`, `test_create_arq_pool_failure_returns_none`. All pass. |
| `tests/dispatch/test_worker.py` | Tests for wakeup handler and graceful shutdown | VERIFIED | 10 tests including `test_handle_wakeup_claims_and_dispatches`, `test_graceful_shutdown_waits_for_active_tasks`, `test_stopping_flag_exits_poll_loop`. All pass. |
| `tests/dispatch/test_integration.py` | Integration tests for wakeup and shutdown flows | VERIFIED | 5 tests including `test_run_creation_enqueues_wakeup`, `test_graceful_shutdown_called_on_lifespan_exit`, `test_worker_uses_dispatch_settings`. All pass. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `lease.py` | `async_postgres.py` | isinstance check for backend detection | WIRED | Line 25: `from zeroth.storage.async_postgres import AsyncPostgresDatabase`, line 58: `isinstance(self.database, AsyncPostgresDatabase)` |
| `arq_wakeup.py` | `settings.py` | arq_settings_from_zeroth converts RedisSettings | WIRED | Lines 19-41: converts `redis_settings.host/port/db/password/tls` to `ArqRedisSettings` |
| `worker.py` | `arq_wakeup.py` | Worker uses on_wakeup callback from ARQ consumer | WIRED | `handle_wakeup` method (line 279) receives callback from `run_arq_consumer` wired in `app.py:97` |
| `run_api.py` | `arq_wakeup.py` | enqueue_wakeup call after run creation | WIRED | Lines 128-132: `enqueue_wakeup(arq_pool, persisted.run_id)` |
| `app.py` | `worker.py` | graceful_shutdown on SIGTERM | WIRED | Line 119: `worker.graceful_shutdown()` in shutdown watcher, line 137 in lifespan cleanup |
| `bootstrap.py` | `arq_wakeup.py` | create_arq_pool at bootstrap time | WIRED | Lines 315-318: `from zeroth.dispatch.arq_wakeup import create_arq_pool; arq_pool = await create_arq_pool(settings.redis)` |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| All dispatch tests pass | `uv run pytest tests/dispatch/ -v` | 37 passed (16 lease + 6 arq + 10 worker + 5 integration) | PASS |
| Ruff clean on all phase files | `uv run ruff check src/zeroth/dispatch/ src/zeroth/config/settings.py src/zeroth/service/{bootstrap,app,run_api}.py` | All checks passed | PASS |
| arq importable | Verified via test_arq_wakeup tests using `from arq.connections import RedisSettings` | arq 0.27 installed | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| OPS-04 | 16-01, 16-02, 16-03 | Multi-worker horizontal scaling with shared Postgres lease store | SATISFIED | SKIP LOCKED claiming in lease.py, graceful shutdown in worker.py, bootstrap wiring |
| OPS-05 | 16-02, 16-03 | ARQ (Redis queue) wakeup notifications supplementing existing lease poller | SATISFIED | arq_wakeup.py module, enqueue_wakeup in run_api.py and approval_api.py, ARQ consumer in app.py lifespan |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | - | - | - | No anti-patterns detected. No TODOs, FIXMEs, placeholders, or stub returns found. |

### Human Verification Required

### 1. Multi-Worker Disjoint Claiming

**Test:** Start two worker processes against the same Postgres database and submit 10 runs; verify each run executes exactly once.
**Expected:** All 10 runs complete with no duplicates across workers.
**Why human:** Requires a live multi-process Postgres deployment to validate true horizontal scaling.

### 2. ARQ Wakeup Latency Improvement

**Test:** Submit a run with `arq_enabled=true` and observe wall-clock time to first processing step vs poll_interval.
**Expected:** Worker begins processing well before the 500ms poll interval elapses.
**Why human:** Requires running Redis/ARQ infrastructure to measure latency improvement.

### 3. SIGTERM Lease Release and Reclaim

**Test:** Send SIGTERM to a worker while it has in-flight runs, then verify another worker reclaims them.
**Expected:** In-flight runs are released to PENDING and picked up by another worker.
**Why human:** Requires multi-process deployment with signal delivery.

### Gaps Summary

No gaps found. All observable truths verified, all artifacts substantive and wired, all 37 tests pass, all key links confirmed, both requirements (OPS-04, OPS-05) satisfied. The phase goal is achieved at the code level.

---

_Verified: 2026-04-07T20:00:00Z_
_Verifier: Claude (gsd-verifier)_
