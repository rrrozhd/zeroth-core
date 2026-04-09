---
phase: 18-cross-phase-integration-wiring
verified: 2026-04-08T09:15:00Z
status: passed
score: 6/6 must-haves verified
re_verification: false
---

# Phase 18: Cross-Phase Integration Wiring Verification Report

**Phase Goal:** Close all worktree merge gaps by wiring DispatchSettings, ARQ pool, InstrumentedProviderAdapter, and memory factory into the production bootstrap path, and fix the cost API double-prefix bug.
**Verified:** 2026-04-08T09:15:00Z
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `ZerothSettings` includes `DispatchSettings` as a nested field, and `ZEROTH_DISPATCH__ARQ_ENABLED` env var is respected | VERIFIED | `settings.py` line 116-121 defines `DispatchSettings` with `arq_enabled: bool = False`; line 158 adds `dispatch: DispatchSettings` field to `ZerothSettings`. Runtime check confirms `ZerothSettings().dispatch.arq_enabled is False`. |
| 2 | `bootstrap_service()` creates an ARQ pool when dispatch ARQ is enabled and stores it on `ServiceBootstrap` | VERIFIED | `bootstrap.py` line 270 checks `settings.dispatch.arq_enabled`; line 274 calls `await create_arq_pool(settings.redis)`; line 380 passes `arq_pool=arq_pool` to `ServiceBootstrap` constructor. |
| 3 | `bootstrap_service()` wraps provider adapters with `InstrumentedProviderAdapter` so cost events are emitted on every LLM call | VERIFIED | `runtime.py` lines 251-268 wrap `runner.provider` with `InstrumentedProviderAdapter` when `self.regulus_client` and `self.cost_estimator` are set; `bootstrap.py` lines 264-266 wire these fields onto the orchestrator. Provider restored in `finally` block (line 282). |
| 4 | `bootstrap_service()` passes real `ZerothSettings` (not stub) to the memory connector factory, enabling external backends | VERIFIED | `bootstrap.py` line 296 calls `register_memory_connectors(memory_registry, settings, redis_client=redis_client, pg_conninfo=pg_conninfo)` where `settings` is `get_settings()` (line 235). `_BootstrapMemorySettings()` is no longer used in the registration call. |
| 5 | `GET /v1/tenants/{id}/cost` routes respond correctly without double `/v1/v1/` prefix | VERIFIED | `cost_api.py` line 38 uses `"/tenants/{tenant_id}/cost"` (no `/v1/` prefix); `app.py` line 249+258 registers on `APIRouter(prefix="/v1")`. No `/v1/` found in `cost_api.py` route decorators. All 6 cost API tests pass. |
| 6 | REQUIREMENTS.md traceability reflects accurate completion status for all v1.1 requirements | VERIFIED | `grep -c '\- \[ \]' REQUIREMENTS.md` returns 0. All 28 requirements marked `[x]`. Traceability table shows Complete for all Phase 18 requirements (ECON-01, ECON-02, ECON-04, MEM-01, OPS-04, OPS-05). |

**Score:** 6/6 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/config/settings.py` | DispatchSettings class and dispatch field on ZerothSettings | VERIFIED | Lines 116-121: `class DispatchSettings(BaseModel)` with `arq_enabled`, `shutdown_timeout`, `poll_interval`. Line 158: `dispatch` field on `ZerothSettings`. |
| `src/zeroth/service/bootstrap.py` | ARQ pool wiring, real settings for memory factory, cost_estimator field | VERIFIED | Lines 134-136: `cost_estimator`, `arq_pool`, `redis_client` fields on `ServiceBootstrap`. Lines 268-296: ARQ pool creation, redis client, memory registration with real settings. Lines 264-266: orchestrator cost instrumentation wiring. |
| `src/zeroth/service/cost_api.py` | Fixed route paths without /v1/ prefix | VERIFIED | Lines 38 and 61: routes use `/tenants/...` and `/deployments/...` without `/v1/` prefix. Uses `request.app.state` for APIRouter compatibility. |
| `tests/test_cost_api.py` | Updated test paths matching new route definitions | VERIFIED | Uses `APIRouter(prefix="/v1")` (lines 22-24, 76-78, 115-117). All 6 tests pass. |
| `src/zeroth/orchestrator/runtime.py` | Per-node InstrumentedProviderAdapter wrapping in _dispatch_node | VERIFIED | Lines 77-79: `regulus_client`, `cost_estimator`, `deployment_ref` fields. Lines 251-282: InstrumentedProviderAdapter wrapping with try/finally restore pattern. |
| `.planning/REQUIREMENTS.md` | Updated traceability markers | VERIFIED | All 28 v1.1 requirements checked. Traceability table shows Complete for all Phase 18 requirements. Coverage: 28/28 satisfied. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `settings.py` | `bootstrap.py` | `settings.dispatch.arq_enabled` check | WIRED | bootstrap.py line 270: `if settings.dispatch.arq_enabled:` |
| `bootstrap.py` | `dispatch/arq_wakeup.py` | `create_arq_pool(settings.redis)` | WIRED | bootstrap.py line 274: `arq_pool = await create_arq_pool(settings.redis)` |
| `bootstrap.py` | `memory/factory.py` | `register_memory_connectors(memory_registry, settings, ...)` | WIRED | bootstrap.py line 296: passes real `settings` and `redis_client` |
| `runtime.py` | `econ/adapter.py` | `InstrumentedProviderAdapter` wrapping `runner.provider` | WIRED | runtime.py lines 255-266: lazy import and instantiation in `_dispatch_node` |
| `runtime.py` | `bootstrap.py` | `self.cost_estimator` and `self.regulus_client` fields | WIRED | bootstrap.py lines 264-266: sets `orchestrator.regulus_client`, `.cost_estimator`, `.deployment_ref` |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| `cost_api.py` | `TenantCostResponse` | Regulus backend via `httpx.AsyncClient.get()` | Yes (proxies to external Regulus service) | FLOWING |
| `runtime.py` | `InstrumentedProviderAdapter` wrapping | `self.regulus_client` + `self.cost_estimator` from bootstrap | Yes (wraps real provider, emits events to Regulus) | FLOWING |
| `bootstrap.py` | `memory_registry` | `register_memory_connectors()` with real `ZerothSettings` | Yes (settings drive connector registration) | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| DispatchSettings defaults | `uv run python -c "...ZerothSettings().dispatch.arq_enabled..."` | Prints `False` | PASS |
| Cost API tests pass | `uv run pytest tests/test_cost_api.py -v` | 6/6 passed | PASS |
| RuntimeOrchestrator has cost fields | `uv run python -c "...inspect.signature(RuntimeOrchestrator)..."` | All 3 fields found | PASS |
| No /v1/ in cost_api.py routes | `grep '"/v1/' cost_api.py` | No matches (exit 1) | PASS |
| All requirements checked | `grep -c '\- \[ \]' REQUIREMENTS.md` | Returns 0 | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| ECON-01 | 18-02 | InstrumentedProviderAdapter wraps any ProviderAdapter and emits Regulus ExecutionEvent per LLM call | SATISFIED | runtime.py lines 251-268: per-node wrapping in _dispatch_node |
| ECON-02 | 18-02 | Token cost attributed per node, run, tenant, and deployment in audit records | SATISFIED | runtime.py line 258-266: InstrumentedProviderAdapter receives node_id, run_id, tenant_id, deployment_ref |
| ECON-04 | 18-01 | REST endpoints expose cumulative cost per tenant and deployment | SATISFIED | cost_api.py routes at /tenants/{id}/cost and /deployments/{ref}/cost, registered on /v1 router |
| MEM-01 | 18-01 | Redis-backed key-value memory connector replacing in-memory dict | SATISFIED | bootstrap.py line 281-288: redis_client created from settings; line 296: passed to register_memory_connectors |
| MEM-06 | 18-01 | Zeroth memory connectors bridged to GovernAI ScopedMemoryConnector and AuditingMemoryConnector | SATISFIED | Memory factory receives real settings enabling external backends (Phase 14 code + Phase 18 wiring) |
| OPS-04 | 18-01 | Multi-worker horizontal scaling with shared Postgres lease store | SATISFIED | DispatchSettings class with arq_enabled, shutdown_timeout, poll_interval in ZerothSettings |
| OPS-05 | 18-01 | ARQ (Redis queue) wakeup notifications supplementing existing lease poller | SATISFIED | bootstrap.py lines 269-276: conditional ARQ pool creation when dispatch.arq_enabled is True |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `bootstrap.py` | 36-75 | Dead code: `_BootstrapMemorySettings` and `_BootstrapMemorySubsection` classes defined but never used | Info | No functional impact; memory registration now uses real `ZerothSettings`. Could be cleaned up. |

### Human Verification Required

### 1. End-to-End Cost Event Emission

**Test:** Deploy with Regulus enabled, trigger an agent node execution, verify cost events appear in Regulus dashboard.
**Expected:** Each LLM call emits a Regulus ExecutionEvent with correct tenant_id, deployment_ref, node_id, and cost_usd.
**Why human:** Requires running Regulus backend and triggering real LLM calls through the orchestrator.

### 2. ARQ Pool Activation

**Test:** Set `ZEROTH_DISPATCH__ARQ_ENABLED=true` with Redis running, start service, verify ARQ pool connects.
**Expected:** Service starts without error; ARQ pool is created and stored on ServiceBootstrap.
**Why human:** Requires running Redis instance and observing bootstrap behavior.

### 3. External Memory Backend Registration

**Test:** Configure Postgres DSN and Redis credentials, start service, verify memory connectors register with real backends.
**Expected:** Memory registry contains Redis and pgvector connectors connected to real backends.
**Why human:** Requires running Postgres+Redis infrastructure and verifying connector connectivity.

### Gaps Summary

No gaps found. All 6 success criteria are verified through code inspection and behavioral spot-checks. All 7 requirement IDs from the phase are accounted for and satisfied. The only notable item is dead code (`_BootstrapMemorySettings` classes) that could be cleaned up in a future phase.

---

_Verified: 2026-04-08T09:15:00Z_
_Verifier: Claude (gsd-verifier)_
