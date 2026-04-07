# Phase 16: Distributed Dispatch & Horizontal Scaling - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 16 makes the existing lease-based run dispatch work across multiple worker processes sharing a Postgres lease store, and adds ARQ-backed wakeup notifications to reduce poll latency without replacing the database as the authoritative queue. The SQLite backend retains its current single-process dispatch behavior for dev/test.

</domain>

<decisions>
## Implementation Decisions

### Lease Query Strategy
- **D-01:** Postgres lease claiming uses `SELECT ... FOR UPDATE SKIP LOCKED` in a transaction — workers skip rows already being claimed by another worker. Zero contention, no retries needed.
- **D-02:** SQLite lease path keeps the current timestamp-expiry `UPDATE WHERE lease_expires_at < now()` pattern unchanged — single-writer makes it safe for dev/test.
- **D-03:** LeaseManager gains a backend-aware claiming strategy: Postgres path uses SKIP LOCKED, SQLite path retains current logic. Both paths return the same result type.

### ARQ Wakeup Pattern
- **D-04:** Run submission enqueues a minimal ARQ job (just run_id) as a "wake up and claim from DB" signal. ARQ handles distribution — exactly one worker gets the wakeup notification.
- **D-05:** ARQ is optional with fallback — if Redis/ARQ is unavailable, workers fall back to pure poll-based dispatch (current behavior). Config flag: `ZEROTH_ARQ_ENABLED` (default false).
- **D-06:** The ARQ job does NOT carry run payload or orchestrate execution — it only signals the worker to check the lease store. Postgres lease remains the authoritative queue.
- **D-07:** Workers run both: ARQ consumer (for wakeup signals) AND poll loop (as fallback and catch-all for missed notifications).

### Worker Identity & Visibility
- **D-08:** Keep current uuid-per-instance worker ID pattern. Lease renewals serve as implicit heartbeat — if a worker stops renewing, its leases expire and runs get reclaimed.
- **D-09:** Operational visibility is log-based only — workers log their ID, start time, hostname, and claimed runs via structured logging. No new worker registration table needed.
- **D-10:** No worker dashboard or admin API for worker tracking in this phase — log aggregation / metrics dashboards are the ops surface.

### Graceful Shutdown
- **D-11:** On SIGTERM, worker stops claiming new runs, waits for in-flight runs to complete (up to 30s configurable timeout), then actively releases remaining leases back to PENDING status.
- **D-12:** If the process is killed hard (SIGKILL/crash), expiry-based reclamation kicks in as safety net — other workers reclaim after lease expires (60s default).
- **D-13:** Shutdown timeout is configurable via settings (default 30s). Balances completion chance with deploy speed.

### Claude's Discretion
- Exact ARQ job schema and task naming conventions
- Whether to use a dedicated ARQ worker class or integrate into the existing RunWorker
- Alembic migration details for any index changes needed for SKIP LOCKED performance
- ARQ connection pool configuration and error handling specifics
- Structured log format for worker lifecycle events

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Dispatch & Lease System
- `src/zeroth/dispatch/lease.py` — Current LeaseManager with atomic claim, expiry-based reclamation, orphan recovery
- `src/zeroth/dispatch/worker.py` — Current RunWorker: poll loop, semaphore(8), graceful shutdown, lease renewal
- `src/zeroth/dispatch/__init__.py` — Module exports

### Configuration & Storage
- `src/zeroth/config/settings.py` — ZerothSettings with RedisSettings, GuardrailConfig (max_concurrency, lease params)
- `src/zeroth/storage/redis.py` — RedisConfig, Redis client factories (reuse for ARQ connection)
- `src/zeroth/db/` — Async Database protocol, SQLite and Postgres implementations

### Bootstrap & Lifespan
- `src/zeroth/service/bootstrap.py` — ServiceBootstrap wiring LeaseManager, RunWorker, and all background tasks
- `src/zeroth/service/app.py` — FastAPI lifespan managing background task lifecycle

### Pattern References
- `src/zeroth/webhooks/delivery.py` — WebhookDeliveryWorker: semaphore-based bounded concurrency pattern (Phase 15)
- `src/zeroth/approvals/sla_checker.py` — ApprovalSLAChecker: background poll-loop pattern (Phase 15)

### Existing Infrastructure
- `src/zeroth/observability/queue_gauge.py` — QueueDepthGauge polling pending run count (adjust for multi-worker)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **LeaseManager** (`dispatch/lease.py`): Core lease logic — needs Postgres SKIP LOCKED path added alongside existing SQLite path
- **RunWorker** (`dispatch/worker.py`): Poll loop + semaphore pattern — needs ARQ consumer integration and shutdown enhancement
- **RedisConfig** (`storage/redis.py`): Redis connection management — reuse for ARQ connection pool
- **GuardrailConfig** (`config/settings.py`): Already has `max_concurrency` and lease duration params — extend with ARQ and shutdown settings

### Established Patterns
- **Poll-claim-execute loop**: RunWorker, WebhookDeliveryWorker, ApprovalSLAChecker all follow the same pattern — claim from DB, execute, release
- **Semaphore-based bounded concurrency**: `asyncio.Semaphore(max_concurrency)` with `_active_tasks` set and done callbacks
- **Graceful shutdown**: Cancel poll loop task, let executing tasks finish within semaphore timeout
- **Backend-conditional logic**: `ZEROTH_DB_BACKEND` flag already switches SQLite/Postgres — extend pattern for lease query strategy

### Integration Points
- **Run submission** (`service/handlers.py` or equivalent): Insert point for ARQ wakeup job enqueue
- **ServiceBootstrap**: Wire ARQ worker/consumer alongside RunWorker
- **App lifespan**: Start/stop ARQ consumer task alongside existing background tasks
- **LeaseManager**: Add `claim_pending_pg()` method or backend-conditional logic in existing `claim_pending()`

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Key constraint: ARQ is strictly a wakeup optimization, never the authoritative queue.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 16-distributed-dispatch-horizontal-scaling*
*Context gathered: 2026-04-07*
