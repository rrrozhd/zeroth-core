# Phase 18: Cross-Phase Integration Wiring - Research

**Researched:** 2026-04-08
**Domain:** Bootstrap wiring, Pydantic settings, FastAPI route mounting
**Confidence:** HIGH

## Summary

Phase 18 is a gap-closure phase that fixes five integration wiring issues left over from worktree-based parallel execution of Phases 13, 14, and 16. All the implementation code already exists -- InstrumentedProviderAdapter, DispatchSettings, ARQ pool creation, memory connector factory, cost API routes -- but the shared files (`settings.py`, `bootstrap.py`, `cost_api.py`) were not fully updated during worktree merges.

The changes are surgical: add one dataclass field, add one settings class, change one function call argument, wrap providers in a decorator, and remove hardcoded path prefixes. No new modules, no new dependencies, no architectural changes. The existing test for dispatch settings (`tests/dispatch/test_integration.py::test_worker_uses_dispatch_settings`) already validates the DispatchSettings shape but currently fails because the field is missing from ZerothSettings.

**Primary recommendation:** Execute all five wiring fixes in a single plan with 5-6 focused tasks, each touching 1-2 files. Changes are independent enough to verify individually but small enough for a single plan.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ECON-01 | InstrumentedProviderAdapter wraps any ProviderAdapter and emits Regulus ExecutionEvent per LLM call | Adapter exists at `econ/adapter.py`; bootstrap must wrap agent runners' providers with it when Regulus is enabled |
| ECON-02 | Token cost attributed per node, run, tenant, and deployment in audit records | Depends on ECON-01 wiring; once adapter wraps providers, cost_usd and cost_event_id flow into ProviderResponse |
| ECON-04 | REST endpoints expose cumulative cost per tenant and deployment | Routes in `service/cost_api.py` have hardcoded `/v1/` prefix; remove prefix since v1_router already provides `/v1/` |
| MEM-01 | Redis-backed key-value memory connector replacing in-memory dict | RedisKVMemoryConnector exists; bootstrap passes stub settings instead of real ZerothSettings to factory |
| MEM-06 | Zeroth memory connectors bridged to GovernAI ScopedMemoryConnector and AuditingMemoryConnector | Code works; REQUIREMENTS.md traceability table still marks as Pending |
| OPS-04 | Multi-worker horizontal scaling with shared Postgres lease store | DispatchSettings class was created in Phase 16 worktree but not merged into ZerothSettings |
| OPS-05 | ARQ (Redis queue) wakeup notifications supplementing existing lease poller | arq_pool field not on ServiceBootstrap; create_arq_pool never called in bootstrap_service() |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Build/test: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- Project layout: `src/zeroth/` main package, `tests/` pytest tests
- Progress logging mandatory via `progress-logger` skill
- Context efficiency: read only what is needed per task

## Architecture Patterns

### Files Requiring Modification

```
src/zeroth/
  config/
    settings.py          # Add DispatchSettings class + dispatch field on ZerothSettings
  service/
    bootstrap.py         # Wire arq_pool, InstrumentedProviderAdapter, real settings to memory factory
    cost_api.py          # Remove /v1/ prefix from route decorators
.planning/
  REQUIREMENTS.md        # Update stale traceability markers
```

### Pattern 1: Nested Pydantic Settings (DispatchSettings)

**What:** Add a new `DispatchSettings(BaseModel)` class and a `dispatch` field on `ZerothSettings`, following the exact pattern used by all other settings subsections.

**Current pattern in settings.py:**
```python
class WebhookSettings(BaseModel):
    enabled: bool = True
    # ...

class ZerothSettings(BaseSettings):
    webhook: WebhookSettings = Field(default_factory=WebhookSettings)
```

**DispatchSettings to add** (from Phase 16 worktree, verified by existing test at `tests/dispatch/test_integration.py:193-221`):
```python
class DispatchSettings(BaseModel):
    """Dispatch and horizontal scaling configuration."""
    arq_enabled: bool = False
    shutdown_timeout: float = 30.0
    poll_interval: float = 0.5
```

Add `dispatch: DispatchSettings = Field(default_factory=DispatchSettings)` to ZerothSettings, after `approval_sla` and before `tls`.

**Confidence:** HIGH -- existing test validates the exact field names and defaults; env var `ZEROTH_DISPATCH__ARQ_ENABLED` in docker-compose.yml confirms the nesting delimiter pattern.

### Pattern 2: ARQ Pool Bootstrap Wiring

**What:** When `settings.dispatch.arq_enabled` is True, call `create_arq_pool(settings.redis)` in `bootstrap_service()` and store result on ServiceBootstrap.

**Changes needed:**
1. Add `arq_pool: object | None = None` field to `ServiceBootstrap` dataclass (line ~126)
2. In `bootstrap_service()`, after the Regulus wiring block (~line 250), add ARQ pool creation:
```python
# Phase 16: ARQ wakeup pool.
arq_pool = None
if settings.dispatch.arq_enabled:
    try:
        from zeroth.dispatch.arq_wakeup import create_arq_pool
        arq_pool = await create_arq_pool(settings.redis)
    except ImportError:
        pass
```
3. Pass `arq_pool=arq_pool` to ServiceBootstrap constructor

**Downstream consumers already handle this gracefully:**
- `app.py` lifespan (line 89): `arq_pool = getattr(app.state.bootstrap, "arq_pool", None)` -- already uses getattr with None default
- `run_api.py` (line 128): same getattr pattern
- `approval_api.py` (line 147): same getattr pattern

**Confidence:** HIGH -- all consumer code already exists and uses defensive getattr.

### Pattern 3: InstrumentedProviderAdapter Wrapping

**What:** When Regulus is enabled and a regulus_client + cost_estimator are available, wrap each AgentRunner's provider adapter with InstrumentedProviderAdapter in bootstrap.

**Key challenge:** InstrumentedProviderAdapter is per-call scoped (needs node_id, run_id, tenant_id, deployment_ref), not per-bootstrap scoped. The adapter constructor requires runtime context that is not available at bootstrap time.

**Two approaches:**
1. **Bootstrap-time wrapping** -- store regulus_client and cost_estimator on ServiceBootstrap; let the orchestrator/AgentRunner create InstrumentedProviderAdapter per-call with runtime context
2. **Factory pattern** -- provide a factory function that the orchestrator calls per-node-execution

**Analysis of InstrumentedProviderAdapter constructor:**
```python
def __init__(self, inner, regulus_client, cost_estimator, *, node_id, run_id, tenant_id, deployment_ref):
```
This requires per-execution context (node_id, run_id, tenant_id). Therefore, wrapping cannot happen at bootstrap time. Instead, bootstrap should ensure the regulus_client and cost_estimator are available, and the orchestrator or AgentRunner should wrap at call time.

**Recommended approach:** Add `cost_estimator: CostEstimator | None = None` field to ServiceBootstrap. The regulus_client field already exists. Then in the orchestrator or AgentRunner execution path, when both regulus_client and cost_estimator are present, wrap the provider adapter with InstrumentedProviderAdapter before calling ainvoke. This matches the audit pattern where context is added per-execution.

**Alternative simpler approach:** If the orchestrator already receives the bootstrap object or its components, add a helper that creates InstrumentedProviderAdapter at execution time. The bootstrap just needs to ensure cost_estimator is created when Regulus is enabled.

**Confidence:** MEDIUM -- the wiring path from bootstrap to per-call wrapping needs careful design. The adapter itself is verified working.

### Pattern 4: Memory Factory Real Settings

**What:** Replace `_BootstrapMemorySettings()` with real `ZerothSettings` instance in the `register_memory_connectors()` call.

**Current (broken):**
```python
register_memory_connectors(memory_registry, _BootstrapMemorySettings())
```

**Fixed:**
```python
register_memory_connectors(memory_registry, settings)
```

The `settings` variable (ZerothSettings instance) is already available in `bootstrap_service()` at line 231. The `register_memory_connectors` function uses duck-typed `settings: Any` and accesses `.memory`, `.pgvector`, `.chroma`, `.elasticsearch` attributes -- all of which exist on ZerothSettings.

**Additionally:** Pass `redis_client` and `pg_conninfo` when available:
- Redis client: needs creation from `settings.redis` (redis.asyncio)
- pg_conninfo: available from `settings.database.postgres_dsn` when backend is postgres

**Confidence:** HIGH -- the factory already accepts these parameters; ZerothSettings has the exact attribute shape expected.

### Pattern 5: Cost API Double Prefix Fix

**What:** Remove hardcoded `/v1/` from route decorator paths in `cost_api.py`.

**Current (broken):**
```python
@app.get("/v1/tenants/{tenant_id}/cost", ...)
@app.get("/v1/deployments/{deployment_ref}/cost", ...)
```

When registered on `v1_router` (which has `prefix="/v1"`), paths become `/v1/v1/tenants/...`.

**Fixed:**
```python
@app.get("/tenants/{tenant_id}/cost", ...)
@app.get("/deployments/{deployment_ref}/cost", ...)
```

**Test impact:** `tests/test_cost_api.py` creates a standalone FastAPI app (no router prefix) and tests against `/v1/tenants/t1/cost`. After the fix, tests must use `/tenants/t1/cost` since the routes no longer include the prefix. Alternatively, tests can mount on a router with `/v1/` prefix to match production.

**Confidence:** HIGH -- straightforward path fix; the bug is clearly documented in the audit.

### Anti-Patterns to Avoid

- **Do NOT create new modules** -- all code already exists; this phase only wires existing pieces together
- **Do NOT change InstrumentedProviderAdapter's interface** -- it works correctly; only the instantiation site is missing
- **Do NOT remove _BootstrapMemorySettings class** -- keep it for backward compatibility in tests that may construct bootstrap without full settings
- **Do NOT change the compat_router registration** -- cost routes must work on both v1_router (with prefix) and compat_router (without prefix); the fix is to remove the prefix from the route decorator so it works correctly in both contexts

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Settings nesting | Custom env parsing | Pydantic `env_nested_delimiter="__"` | Already configured; ZEROTH_DISPATCH__ARQ_ENABLED auto-maps |
| Redis connection pooling | Manual connection management | `redis.asyncio.from_url()` or existing RedisSettings | Standard pattern in the codebase |
| ARQ pool creation | Direct arq.create_pool | `zeroth.dispatch.arq_wakeup.create_arq_pool()` | Already wraps error handling and settings conversion |
| Cost estimation | Manual token pricing | `zeroth.econ.cost.CostEstimator` | Already wraps litellm pricing with fallback |

## Common Pitfalls

### Pitfall 1: InstrumentedProviderAdapter Requires Per-Execution Context
**What goes wrong:** Attempting to wrap providers at bootstrap time, but the adapter needs node_id, run_id, tenant_id, deployment_ref which are only known at execution time.
**Why it happens:** The adapter is designed for per-call wrapping, not singleton wrapping.
**How to avoid:** Bootstrap provides regulus_client and cost_estimator; wrapping happens in the orchestrator/AgentRunner execution path.
**Warning signs:** If you see InstrumentedProviderAdapter in ServiceBootstrap constructor, something is wrong.

### Pitfall 2: Cost API Test Path Mismatch
**What goes wrong:** After removing `/v1/` from cost_api route decorators, existing tests that hit `/v1/tenants/t1/cost` on a bare FastAPI app will get 404.
**Why it happens:** Tests use a standalone FastAPI app (no v1_router prefix), so the old hardcoded `/v1/` was actually needed for tests to pass.
**How to avoid:** Update tests to use paths without `/v1/` prefix, OR mount routes on an APIRouter with `/v1/` prefix in the test helper.
**Warning signs:** test_cost_api.py tests returning 404 after the fix.

### Pitfall 3: Settings Singleton Cache
**What goes wrong:** Tests that manipulate environment variables after settings have been loaded get stale values.
**Why it happens:** `get_settings()` caches a singleton at module level.
**How to avoid:** Existing dispatch integration test already patches `_settings_singleton` to None before constructing fresh settings. Follow the same pattern.

### Pitfall 4: Redis Client Availability for Memory Factory
**What goes wrong:** Passing real settings to memory factory but not providing a redis_client, so Redis connectors are still not registered.
**Why it happens:** The factory requires an explicit redis_client parameter; settings alone are not enough.
**How to avoid:** Create a redis.asyncio client from settings.redis when redis mode is not "disabled", and pass it to register_memory_connectors.

### Pitfall 5: Import Guards for Optional Dependencies
**What goes wrong:** Hard imports of arq, redis, chromadb cause ImportError in environments without those packages.
**Why it happens:** Not all deployments have all optional backends installed.
**How to avoid:** Follow existing pattern: `try/except ImportError` guards (used throughout dispatch, memory modules). The codebase already uses this pattern extensively.

## Code Examples

### DispatchSettings Addition (settings.py)
```python
# Add after ApprovalSLASettings class, before TLSSettings
class DispatchSettings(BaseModel):
    """Dispatch and horizontal scaling configuration."""
    arq_enabled: bool = False
    shutdown_timeout: float = 30.0
    poll_interval: float = 0.5

# In ZerothSettings class, add field:
dispatch: DispatchSettings = Field(default_factory=DispatchSettings)
```

### ARQ Pool in Bootstrap (bootstrap.py)
```python
# After regulus wiring block
arq_pool = None
if settings.dispatch.arq_enabled:
    try:
        from zeroth.dispatch.arq_wakeup import create_arq_pool
        arq_pool = await create_arq_pool(settings.redis)
    except ImportError:
        pass
```

### Memory Factory with Real Settings (bootstrap.py)
```python
# Replace: register_memory_connectors(memory_registry, _BootstrapMemorySettings())
# With:
redis_client = None
if settings.redis.mode != "disabled":
    try:
        import redis.asyncio as aioredis
        redis_url = f"redis://{settings.redis.host}:{settings.redis.port}/{settings.redis.db}"
        if settings.redis.password:
            redis_url = f"redis://:{settings.redis.password.get_secret_value()}@{settings.redis.host}:{settings.redis.port}/{settings.redis.db}"
        redis_client = aioredis.from_url(redis_url)
    except ImportError:
        pass

pg_conninfo = None
if settings.database.backend == "postgres" and settings.database.postgres_dsn:
    pg_conninfo = settings.database.postgres_dsn.get_secret_value()

register_memory_connectors(memory_registry, settings, redis_client=redis_client, pg_conninfo=pg_conninfo)
```

### Cost API Route Fix (cost_api.py)
```python
# Before (broken):
@app.get("/v1/tenants/{tenant_id}/cost", response_model=TenantCostResponse)
@app.get("/v1/deployments/{deployment_ref}/cost", response_model=DeploymentCostResponse)

# After (fixed):
@app.get("/tenants/{tenant_id}/cost", response_model=TenantCostResponse)
@app.get("/deployments/{deployment_ref}/cost", response_model=DeploymentCostResponse)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| _BootstrapMemorySettings stub | Real ZerothSettings | Phase 18 | External memory backends become functional |
| No cost event emission | InstrumentedProviderAdapter wrapping | Phase 18 | Every LLM call emits Regulus cost event |
| Dead ZEROTH_DISPATCH__ARQ_ENABLED | DispatchSettings on ZerothSettings | Phase 18 | ARQ wakeup actually activates |

## Open Questions

1. **InstrumentedProviderAdapter wiring location**
   - What we know: The adapter needs per-execution context (node_id, run_id, etc.) that is not available at bootstrap time.
   - What's unclear: Whether wrapping should happen in RuntimeOrchestrator, AgentRunner, or a new intermediary.
   - Recommendation: Check how AgentRunner calls ProviderAdapter.ainvoke() -- the wrapping should happen at that call site. Add cost_estimator to ServiceBootstrap so the runner can access it. The orchestrator already receives the bootstrap components.

2. **Redis client lifecycle management**
   - What we know: The memory factory needs a redis.asyncio client; bootstrap creates it.
   - What's unclear: Whether the redis client should be closed in lifespan shutdown.
   - Recommendation: Store redis_client on ServiceBootstrap and close in lifespan shutdown, following the same pattern as webhook_http_client.

## Sources

### Primary (HIGH confidence)
- `src/zeroth/config/settings.py` -- current ZerothSettings structure (no dispatch field)
- `src/zeroth/service/bootstrap.py` -- current bootstrap_service() with stub memory settings, no ARQ pool, no adapter wrapping
- `src/zeroth/service/cost_api.py` -- hardcoded `/v1/` in route decorators confirmed
- `src/zeroth/dispatch/arq_wakeup.py` -- create_arq_pool() API confirmed
- `src/zeroth/econ/adapter.py` -- InstrumentedProviderAdapter constructor signature confirmed
- `src/zeroth/memory/factory.py` -- register_memory_connectors() signature confirmed
- `tests/dispatch/test_integration.py` -- DispatchSettings shape confirmed (arq_enabled, shutdown_timeout, poll_interval)
- `.planning/v1.1-MILESTONE-AUDIT.md` -- all five gaps documented with root cause analysis

### Secondary (MEDIUM confidence)
- Phase 16 plan files -- DispatchSettings design intent
- Phase 13 plan files -- InstrumentedProviderAdapter design intent

## Metadata

**Confidence breakdown:**
- Settings merge (DispatchSettings): HIGH -- exact shape verified by existing test
- Bootstrap ARQ wiring: HIGH -- all downstream code already written with defensive getattr
- Memory factory wiring: HIGH -- duck-typed settings interface matches ZerothSettings
- Cost API fix: HIGH -- straightforward path prefix removal
- InstrumentedProviderAdapter wiring: MEDIUM -- wiring site needs investigation at plan time
- REQUIREMENTS.md update: HIGH -- simple checkbox changes

**Research date:** 2026-04-08
**Valid until:** 2026-05-08 (stable -- no external dependency changes expected)
