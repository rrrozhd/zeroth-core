# Phase 16: Distributed Dispatch & Horizontal Scaling - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-07
**Phase:** 16-distributed-dispatch-horizontal-scaling
**Areas discussed:** Lease query strategy, ARQ wakeup pattern, Worker identity, Shutdown behavior

---

## Lease Query Strategy

### Q1: How should Postgres lease claiming work under concurrent workers?

| Option | Description | Selected |
|--------|-------------|----------|
| FOR UPDATE SKIP LOCKED | Workers skip rows already being claimed. Standard Postgres job queue pattern — zero contention, no retries. | ✓ |
| Advisory locks | pg_try_advisory_xact_lock on run_id. More explicit but adds complexity. | |
| Keep expiry pattern | Adapt current timestamp-expiry UPDATE for Postgres. Simpler migration but clock-dependent. | |

**User's choice:** FOR UPDATE SKIP LOCKED (Recommended)

### Q2: Should the SQLite lease path also be updated?

| Option | Description | Selected |
|--------|-------------|----------|
| Keep SQLite as-is | Single-writer makes current pattern safe. Avoids diverging code for dev-only backend. | ✓ |
| Unified query logic | Rewrite both backends for consistency. SQLite doesn't support SKIP LOCKED — needs shim. | |

**User's choice:** Keep SQLite as-is (Recommended)

---

## ARQ Wakeup Pattern

### Q1: How should ARQ wakeup notifications reach workers?

| Option | Description | Selected |
|--------|-------------|----------|
| Enqueue thin job | Minimal ARQ job (just run_id) as "wake up and claim from DB" signal. ARQ handles distribution. Falls back to poll if down. | ✓ |
| Redis pub/sub channel | Publish to channel on submission; workers subscribe. No delivery guarantee. | |
| Redis list (BLPOP) | Push to list; workers BLPOP with timeout. Guaranteed to one worker but loses ARQ features. | |

**User's choice:** Enqueue thin job (Recommended)

### Q2: Should ARQ be required or optional?

| Option | Description | Selected |
|--------|-------------|----------|
| Optional with fallback | If Redis/ARQ unavailable, fall back to pure poll-based dispatch. Config flag ZEROTH_ARQ_ENABLED. | ✓ |
| Required when Postgres | Fail-fast if Redis unavailable in production. Adds hard dependency. | |
| Always optional | Purely an optimization. Never fail if missing. | |

**User's choice:** Optional with fallback (Recommended)

---

## Worker Identity

### Q1: How should worker identity and health be tracked?

| Option | Description | Selected |
|--------|-------------|----------|
| UUID + lease as heartbeat | Keep current uuid pattern. Lease renewals = implicit heartbeat. Add lightweight ops visibility. | ✓ |
| Heartbeat registration table | Workers register and heartbeat to Postgres table. Better visibility, more infra. | |
| UUID only, no tracking | Minimal — worker_id in lease columns only. Debug via log correlation. | |

**User's choice:** UUID + lease as heartbeat (Recommended)

### Q2: What form should ops visibility take?

| Option | Description | Selected |
|--------|-------------|----------|
| Log-based only | Structured logging of worker ID, start time, hostname, claimed runs. No new tables. | ✓ |
| Workers table | Postgres table updated on lease renewal. Queryable via admin API. | |
| Metrics only | Prometheus gauges for worker count and active runs per worker. | |

**User's choice:** Log-based only (Recommended)

---

## Shutdown Behavior

### Q1: How should workers handle leased runs on graceful shutdown?

| Option | Description | Selected |
|--------|-------------|----------|
| Active release + expiry fallback | Stop claiming, wait for in-flight, release remaining leases. Expiry catches hard kills. | ✓ |
| Expiry-only reclamation | Worker just stops. Leases expire after 60s. Simpler but slower recovery. | |
| Immediate requeue | Set all owned runs to PENDING immediately. Fast but risks partial execution. | |

**User's choice:** Active release + expiry fallback (Recommended)

### Q2: Graceful shutdown timeout?

| Option | Description | Selected |
|--------|-------------|----------|
| 30 seconds | Wait up to 30s for in-flight completion, then release. Configurable via settings. | ✓ |
| Match lease duration (60s) | Maximizes completion chance but slows deploys. | |
| 10 seconds | Fast shutdown, more runs requeued mid-flight. | |

**User's choice:** 30 seconds (Recommended)

---

## Claude's Discretion

- Exact ARQ job schema and task naming conventions
- Whether to use dedicated ARQ worker class or integrate into RunWorker
- Alembic migration details for SKIP LOCKED index performance
- ARQ connection pool configuration and error handling
- Structured log format for worker lifecycle events

## Deferred Ideas

None — discussion stayed within phase scope.
