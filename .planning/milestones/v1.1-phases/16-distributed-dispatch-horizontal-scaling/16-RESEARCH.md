# Phase 16: Distributed Dispatch & Horizontal Scaling - Research

**Researched:** 2026-04-07
**Domain:** Multi-worker dispatch, Postgres lease contention, ARQ wakeup notifications
**Confidence:** HIGH

## Summary

Phase 16 makes the existing lease-based dispatch system work across multiple worker processes sharing the same Postgres database. The core change is adding a `SELECT ... FOR UPDATE SKIP LOCKED` claiming path for Postgres (alongside the existing SQLite timestamp-expiry path), and integrating ARQ as an optional wakeup notification layer to reduce poll latency.

The existing codebase is well-structured for this. `LeaseManager` already encapsulates all claim/renew/release logic. `RunWorker` already uses semaphore-bounded concurrency and a poll loop. The `AsyncPostgresDatabase` already converts `?` placeholders to `%s` for psycopg. The main work is: (1) a backend-conditional claiming query in LeaseManager, (2) ARQ wakeup integration at the run submission point, (3) graceful shutdown with lease release on SIGTERM, and (4) an Alembic migration for any needed indexes.

**Primary recommendation:** Add a Postgres-specific `claim_pending` path using `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` in a single atomic query, wire ARQ `enqueue_job` into `create_run` with try/except fallback, and enhance the app lifespan with SIGTERM-aware shutdown that releases leases before exiting.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Postgres lease claiming uses `SELECT ... FOR UPDATE SKIP LOCKED` in a transaction -- workers skip rows already being claimed by another worker. Zero contention, no retries needed.
- **D-02:** SQLite lease path keeps the current timestamp-expiry `UPDATE WHERE lease_expires_at < now()` pattern unchanged -- single-writer makes it safe for dev/test.
- **D-03:** LeaseManager gains a backend-aware claiming strategy: Postgres path uses SKIP LOCKED, SQLite path retains current logic. Both paths return the same result type.
- **D-04:** Run submission enqueues a minimal ARQ job (just run_id) as a "wake up and claim from DB" signal. ARQ handles distribution -- exactly one worker gets the wakeup notification.
- **D-05:** ARQ is optional with fallback -- if Redis/ARQ is unavailable, workers fall back to pure poll-based dispatch (current behavior). Config flag: `ZEROTH_ARQ_ENABLED` (default false).
- **D-06:** The ARQ job does NOT carry run payload or orchestrate execution -- it only signals the worker to check the lease store. Postgres lease remains the authoritative queue.
- **D-07:** Workers run both: ARQ consumer (for wakeup signals) AND poll loop (as fallback and catch-all for missed notifications).
- **D-08:** Keep current uuid-per-instance worker ID pattern. Lease renewals serve as implicit heartbeat.
- **D-09:** Operational visibility is log-based only -- structured logging for worker lifecycle events.
- **D-10:** No worker dashboard or admin API in this phase.
- **D-11:** On SIGTERM, worker stops claiming new runs, waits for in-flight runs to complete (up to 30s configurable timeout), then actively releases remaining leases back to PENDING status.
- **D-12:** If the process is killed hard (SIGKILL/crash), expiry-based reclamation kicks in as safety net -- other workers reclaim after lease expires (60s default).
- **D-13:** Shutdown timeout is configurable via settings (default 30s).

### Claude's Discretion
- Exact ARQ job schema and task naming conventions
- Whether to use a dedicated ARQ worker class or integrate into the existing RunWorker
- Alembic migration details for any index changes needed for SKIP LOCKED performance
- ARQ connection pool configuration and error handling specifics
- Structured log format for worker lifecycle events

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| OPS-04 | Multi-worker horizontal scaling with shared Postgres lease store | SKIP LOCKED claiming pattern, LeaseManager backend-conditional strategy, graceful shutdown with lease release |
| OPS-05 | ARQ (Redis queue) wakeup notifications supplementing existing lease poller | ARQ 0.27.0 enqueue_job + programmatic Worker integration, optional fallback when Redis unavailable |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| arq | 0.27.0 | Async Redis job queue for wakeup notifications | Only mature async-native Redis queue for Python; used by FastAPI ecosystem |
| psycopg | >=3.3 (already installed) | Postgres async driver with FOR UPDATE SKIP LOCKED | Already in project; native support for advisory locks and row-level locking |
| redis | >=5.0.0 (already installed) | Redis client (ARQ dependency) | Already in project for GovernAI runtime stores |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| psycopg-pool | >=3.2 (already installed) | Connection pooling for Postgres | Already wired via AsyncPostgresDatabase |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| ARQ | Postgres LISTEN/NOTIFY | Eliminates Redis dependency but adds complexity; LISTEN/NOTIFY can lose messages under load if client disconnects. ARQ is simpler and Redis is already in the stack. |
| ARQ | Raw Redis BRPOP | Lower level, no retry/serialization built in. ARQ provides structured job handling. |

**Installation:**
```bash
# Add to pyproject.toml dependencies
uv add "arq>=0.27"
```

**Version verification:** ARQ 0.27.0 confirmed on PyPI (latest as of 2026-04-07). Requires Python >=3.9. Compatible with redis-py >=5.0.

## Architecture Patterns

### Recommended Changes to Existing Structure
```
src/zeroth/
├── dispatch/
│   ├── __init__.py          # Add ARQ consumer exports
│   ├── lease.py             # Add claim_pending_pg() or backend-conditional logic
│   ├── worker.py            # Add ARQ wakeup handler, enhanced shutdown
│   └── arq_wakeup.py        # NEW: ARQ job definition, pool factory, wakeup consumer
├── config/
│   └── settings.py          # Add DispatchSettings (arq_enabled, shutdown_timeout)
├── service/
│   ├── app.py               # Enhanced lifespan: SIGTERM handling, ARQ consumer task
│   ├── bootstrap.py         # Wire ARQ pool and consumer
│   └── run_api.py           # Inject ARQ wakeup enqueue after run creation
└── migrations/
    └── versions/
        └── 004_add_skip_locked_index.py  # NEW: index for SKIP LOCKED performance
```

### Pattern 1: Backend-Conditional Lease Claiming
**What:** LeaseManager detects whether the database is Postgres or SQLite and uses the appropriate claiming query.
**When to use:** Every call to `claim_pending()`.
**Example:**
```python
# In lease.py
async def claim_pending(self, deployment_ref: str, worker_id: str) -> str | None:
    if self._is_postgres():
        return await self._claim_pending_pg(deployment_ref, worker_id)
    return await self._claim_pending_sqlite(deployment_ref, worker_id)

async def _claim_pending_pg(self, deployment_ref: str, worker_id: str) -> str | None:
    """Atomic claim using SELECT ... FOR UPDATE SKIP LOCKED."""
    now = _utc_now()
    expires_at = now + timedelta(seconds=self.lease_duration_seconds)
    async with self.database.transaction() as conn:
        # Single atomic query: select + lock + skip already-locked rows
        row = await conn.fetch_one(
            """
            SELECT run_id FROM runs
            WHERE deployment_ref = ?
              AND status = ?
              AND (lease_worker_id IS NULL OR lease_expires_at < ?)
            ORDER BY started_at ASC
            FOR UPDATE SKIP LOCKED
            LIMIT 1
            """,
            (deployment_ref, RunStatus.PENDING.value, now.isoformat()),
        )
        if row is None:
            return None
        run_id = row["run_id"]
        await conn.execute(
            """
            UPDATE runs
            SET lease_worker_id = ?,
                lease_acquired_at = ?,
                lease_expires_at = ?
            WHERE run_id = ?
            """,
            (worker_id, now.isoformat(), expires_at.isoformat(), run_id),
        )
    return run_id
```

**Key difference from SQLite path:** The `FOR UPDATE SKIP LOCKED` on the SELECT means two concurrent workers will never see the same row. No verify step needed -- the lock is acquired at SELECT time, not UPDATE time.

### Pattern 2: ARQ Wakeup Signal (Fire-and-Forget)
**What:** After creating a run, enqueue a lightweight ARQ job that tells any listening worker to check the lease store.
**When to use:** In `create_run` endpoint and `schedule_continuation`.
**Example:**
```python
# In arq_wakeup.py
from arq import create_pool
from arq.connections import RedisSettings as ArqRedisSettings

async def wakeup_worker(ctx: dict, run_id: str) -> None:
    """ARQ job handler -- does nothing itself.
    The worker's ARQ consumer triggers a claim attempt."""
    pass  # The act of receiving the job IS the wakeup signal

async def enqueue_wakeup(arq_pool, run_id: str) -> None:
    """Best-effort wakeup enqueue. Never raises."""
    try:
        await arq_pool.enqueue_job("wakeup_worker", run_id, _job_id=f"wakeup:{run_id}")
    except Exception:
        logger.debug("ARQ wakeup enqueue failed for %s, poll fallback active", run_id)
```

### Pattern 3: Programmatic ARQ Consumer in Existing Event Loop
**What:** Instead of running `arq` CLI, embed the ARQ worker as an asyncio task in the FastAPI lifespan.
**When to use:** Always -- the Zeroth worker process IS the FastAPI app.
**Example:**
```python
# In arq_wakeup.py
from arq.worker import Worker as ArqWorker

async def run_arq_consumer(
    redis_settings: ArqRedisSettings,
    on_wakeup: Callable[[str], Awaitable[None]],
) -> None:
    """Run ARQ consumer as a background task. Calls on_wakeup for each signal."""
    async def handle_wakeup(ctx: dict, run_id: str) -> None:
        await on_wakeup(run_id)

    worker = ArqWorker(
        functions=[handle_wakeup],
        redis_settings=redis_settings,
        burst=False,
        max_jobs=10,
    )
    await worker.async_run()  # Runs until cancelled
```

### Pattern 4: Graceful Shutdown with Lease Release
**What:** On SIGTERM, stop the poll loop and ARQ consumer, wait for in-flight tasks, then release remaining leases.
**When to use:** App lifespan shutdown.
**Example:**
```python
# Enhanced lifespan shutdown
async def graceful_shutdown(worker: RunWorker, timeout: float = 30.0) -> None:
    """Wait for in-flight tasks then release all leases."""
    # Wait for active tasks to complete (with timeout)
    if worker._active_tasks:
        done, pending = await asyncio.wait(
            worker._active_tasks, timeout=timeout
        )
        # For any tasks still running after timeout, release their leases
        for task in pending:
            run_id = _extract_run_id(task)
            if run_id:
                await worker.lease_manager.release_lease(run_id, worker.worker_id)
                # Revert run to PENDING so another worker can claim it
                await _revert_to_pending(worker.run_repository, run_id)
            task.cancel()
```

### Anti-Patterns to Avoid
- **Carrying payload in ARQ jobs:** ARQ is a wakeup signal ONLY. The DB lease is the authoritative queue. If ARQ carried the payload, you'd have two sources of truth.
- **Removing the poll loop when ARQ is active:** The poll loop is the safety net. ARQ notifications can be lost (Redis restart, network blip). Poll catches everything.
- **Using `FOR UPDATE SKIP LOCKED` with SQLite:** SQLite does not support SKIP LOCKED. The existing timestamp-expiry pattern is correct for single-writer SQLite.
- **Blocking shutdown on all tasks:** A configurable timeout (D-13) prevents a hung run from blocking deploys indefinitely.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Redis job queue | Custom BRPOP/LPUSH protocol | ARQ | Serialization, retries, connection management, dedup via _job_id |
| Postgres row-level locking | Application-level mutex/advisory locks | FOR UPDATE SKIP LOCKED | Built into Postgres, zero contention, battle-tested |
| Worker process coordination | Shared-memory / file locks | Postgres lease store (existing) | Already built, works across hosts, crash recovery via expiry |
| Async signal handling | Raw signal.signal() | asyncio loop.add_signal_handler() | Signal handlers in asyncio must be loop-aware to avoid deadlocks |

**Key insight:** The entire distributed dispatch system is already 80% built. The LeaseManager + RunWorker pattern scales to multi-worker with only the SKIP LOCKED query change. ARQ is additive optimization, not architectural change.

## Common Pitfalls

### Pitfall 1: Placeholder Conversion for FOR UPDATE SKIP LOCKED
**What goes wrong:** The `_sqlite_to_psycopg` converter in `async_postgres.py` converts `?` to `%s`. The `FOR UPDATE SKIP LOCKED` clause contains no `?` so it passes through safely. BUT: if someone adds a `?` in a comment or string literal near `FOR UPDATE`, it would be incorrectly converted.
**Why it happens:** Regex-based placeholder conversion is fragile.
**How to avoid:** The existing queries use `?` only for parameters, never in literals. Keep this pattern. The converter works fine for all current and planned queries.
**Warning signs:** psycopg errors about incorrect number of parameters.

### Pitfall 2: SKIP LOCKED Returns No Rows When All Are Locked
**What goes wrong:** If all PENDING runs are currently being claimed by other workers (locked in their transactions), SKIP LOCKED returns empty result set. Worker thinks there's no work.
**Why it happens:** By design -- SKIP LOCKED skips locked rows rather than waiting.
**How to avoid:** This is correct behavior. The worker goes back to polling. The poll interval (0.5s default) ensures it will retry quickly. ARQ wakeup also triggers re-checking.
**Warning signs:** None -- this is expected behavior under high contention.

### Pitfall 3: ARQ Worker.async_run() Blocks Until Cancelled
**What goes wrong:** `worker.async_run()` runs a loop internally. If not wrapped in a task with proper cancellation, it can prevent shutdown.
**Why it happens:** ARQ's async_run is designed to be the "main" coroutine.
**How to avoid:** Run it as an `asyncio.create_task()` in the lifespan. Cancel the task during shutdown, and suppress CancelledError (same pattern as poll_loop).
**Warning signs:** Process hangs on shutdown.

### Pitfall 4: ARQ enqueue_job With Duplicate _job_id
**What goes wrong:** If the same `_job_id` is used for an already-queued-but-not-yet-processed job, ARQ may silently ignore the enqueue.
**Why it happens:** ARQ deduplicates by `_job_id`.
**How to avoid:** Use `_job_id=f"wakeup:{run_id}"` -- each run gets its own wakeup signal. Even if a duplicate is silently dropped, the poll loop ensures the run gets claimed.
**Warning signs:** Wakeup appears to not fire, but run still gets claimed via poll (correct fallback behavior).

### Pitfall 5: Shutdown Race Between Lease Release and Task Cancellation
**What goes wrong:** During shutdown, if you cancel a task AND release its lease simultaneously, the cancelled task's finally block may also try to release the lease.
**Why it happens:** The `_execute_leased_run` finally block already calls `release_lease`. If shutdown also calls it, that's fine -- release_lease is idempotent (WHERE run_id = ? AND lease_worker_id = ?).
**How to avoid:** The existing release_lease is safe to call multiple times. But for reverting to PENDING (D-11), only the shutdown handler should do that -- not the normal execution path.
**Warning signs:** Run stuck in RUNNING after shutdown (lease released but status not reverted).

### Pitfall 6: LeaseManager Backend Detection
**What goes wrong:** LeaseManager needs to know if it's talking to Postgres or SQLite. Passing a flag is fragile; isinstance check is cleaner.
**Why it happens:** LeaseManager takes `AsyncDatabase` protocol, not a concrete class.
**How to avoid:** Use `isinstance(self.database, AsyncPostgresDatabase)` -- the protocol is runtime_checkable so this works. Alternatively, accept a `backend: str` parameter at construction time (mirrors settings.database.backend).
**Warning signs:** Wrong claiming strategy used for the active backend.

## Code Examples

### SKIP LOCKED Claiming Query (Postgres)
```python
# Single-statement atomic claim -- no verify step needed
# Source: Postgres documentation + established job queue patterns
row = await conn.fetch_one(
    """
    SELECT run_id FROM runs
    WHERE deployment_ref = ?
      AND status = ?
      AND (lease_worker_id IS NULL OR lease_expires_at < ?)
    ORDER BY started_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1
    """,
    (deployment_ref, RunStatus.PENDING.value, now.isoformat()),
)
```

### ARQ Pool From Existing RedisSettings
```python
# Reuse ZerothSettings.redis for ARQ connection
from arq.connections import RedisSettings as ArqRedisSettings

def arq_settings_from_zeroth(redis: RedisSettings) -> ArqRedisSettings:
    """Convert ZerothSettings.redis to ARQ RedisSettings."""
    password = redis.password.get_secret_value() if redis.password else None
    return ArqRedisSettings(
        host=redis.host,
        port=redis.port,
        database=redis.db,
        password=password,
        ssl=redis.tls,
    )
```

### Graceful Shutdown Signal Handler
```python
# In app lifespan or entry point
import signal

shutdown_event = asyncio.Event()

def _handle_sigterm():
    shutdown_event.set()

loop = asyncio.get_running_loop()
loop.add_signal_handler(signal.SIGTERM, _handle_sigterm)
loop.add_signal_handler(signal.SIGINT, _handle_sigterm)
```

### Wakeup Enqueue in Run Creation
```python
# In run_api.py create_run handler, after persisting the run
arq_pool = getattr(bootstrap, "arq_pool", None)
if arq_pool is not None:
    try:
        await arq_pool.enqueue_job(
            "wakeup_worker",
            persisted.run_id,
            _job_id=f"wakeup:{persisted.run_id}",
        )
    except Exception:
        pass  # Poll fallback handles it
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Application-level job dedup | `FOR UPDATE SKIP LOCKED` | Postgres 9.5 (2016) | Eliminates all contention; no retries needed |
| Celery/RQ for job queues | ARQ for async Python | 2019+ | Native asyncio, no forking overhead |
| Redis as authoritative queue | DB as queue + Redis as notification | Modern pattern | Simplifies consistency, Redis is optional optimization |

**Deprecated/outdated:**
- ARQ 0.26.x: Older version referenced in STATE.md blocker note. 0.27.0 is current and confirmed on PyPI.

## Open Questions

1. **ARQ consumer integration approach**
   - What we know: ARQ's `Worker.async_run()` can run in an existing event loop. It's designed for standalone use but supports embedding.
   - What's unclear: Whether `async_run()` handles cancellation cleanly or needs wrapper logic. May need to test.
   - Recommendation: Wrap in asyncio.create_task with CancelledError suppression (same pattern as all other background tasks in this codebase).

2. **Index requirements for SKIP LOCKED performance**
   - What we know: `FOR UPDATE SKIP LOCKED` benefits from an index on the WHERE clause columns (deployment_ref, status, lease_worker_id, lease_expires_at).
   - What's unclear: Whether the existing table already has suitable indexes from the initial migration.
   - Recommendation: Check 001_initial_schema.py and add a composite index if missing. Pattern: `CREATE INDEX ix_runs_pending_claim ON runs (deployment_ref, status, started_at) WHERE lease_worker_id IS NULL`.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Postgres | Lease store (SKIP LOCKED) | Assumed (testcontainers in dev) | -- | SQLite path (single-worker dev mode) |
| Redis | ARQ wakeup notifications | Assumed (existing GovernAI dependency) | -- | Poll-only fallback (D-05) |
| ARQ | Wakeup job queue | Not yet installed | 0.27.0 (PyPI) | Must be added to pyproject.toml |

**Missing dependencies with no fallback:**
- ARQ must be added to pyproject.toml (trivial: `arq>=0.27`)

**Missing dependencies with fallback:**
- Redis unavailability at runtime falls back to poll-only dispatch (by design, D-05)
- Postgres unavailability falls back to SQLite single-worker mode (existing behavior)

## Project Constraints (from CLAUDE.md)

- **Build & test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` main package, `tests/` for pytest
- **Progress logging:** Every session MUST use `progress-logger` skill
- **Context efficiency:** Read only task-relevant files, not root PLAN.md or other phases
- **Async pattern:** All database operations use AsyncDatabase protocol (psycopg async, aiosqlite)
- **Config pattern:** Pydantic-settings with env > .env > YAML priority
- **Backend switching:** `settings.database.backend` flag ("sqlite"/"postgres") at startup
- **Placeholder convention:** Use `?` in SQL -- AsyncPostgresDatabase auto-converts to `%s`

## Sources

### Primary (HIGH confidence)
- Existing codebase: `src/zeroth/dispatch/lease.py`, `worker.py`, `storage/async_postgres.py`, `config/settings.py`, `service/bootstrap.py`, `service/app.py` -- read directly
- [ARQ official docs v0.27.0](https://arq-docs.helpmanual.io/) -- API reference for create_pool, enqueue_job, Worker, RedisSettings
- [ARQ GitHub source](https://github.com/python-arq/arq/blob/main/arq/connections.py) -- RedisSettings class, create_pool signature
- [ARQ GitHub worker.py](https://github.com/python-arq/arq/blob/main/arq/worker.py) -- Worker constructor, async_run() method
- [PyPI arq](https://pypi.org/project/arq/) -- Version 0.27.0 confirmed

### Secondary (MEDIUM confidence)
- [Postgres SKIP LOCKED patterns](https://www.inferable.ai/blog/posts/postgres-skip-locked) -- FOR UPDATE SKIP LOCKED job queue pattern
- [Netdata SKIP LOCKED guide](https://www.netdata.cloud/academy/update-skip-locked/) -- Deadlock avoidance with SKIP LOCKED
- [Neon Postgres queue guide](https://neon.com/guides/queue-system) -- Queue system using SKIP LOCKED

### Tertiary (LOW confidence)
- None -- all findings verified against official sources or codebase.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- ARQ version confirmed on PyPI, all other deps already in project
- Architecture: HIGH -- patterns derived directly from existing codebase analysis + Postgres docs
- Pitfalls: HIGH -- identified from code review of actual LeaseManager, RunWorker, and AsyncPostgresDatabase implementations

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable domain, established patterns)
