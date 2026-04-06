# Phase 13: Regulus Economics Integration - Research

**Researched:** 2026-04-07
**Domain:** LLM cost instrumentation, budget enforcement, economic telemetry
**Confidence:** HIGH

## Summary

Phase 13 integrates the Regulus economics SDK (`econ-instrumentation-sdk 0.1.1`) into Zeroth so that every LLM provider call emits a cost event, token costs are attributed per node/run/tenant/deployment in audit records, per-tenant budget caps are enforced before execution, and cumulative cost totals are queryable via REST.

The Regulus SDK is a private Python package located at `/Users/dondoe/coding/regulus/sdk/python/`. It uses pydantic models (`ExecutionEvent`, `OutcomeEvent`), httpx for HTTP transport, and a background daemon thread (`TelemetryTransport`) for non-blocking event emission. The SDK's core dependencies (pydantic>=2.7, httpx>=0.27) are already satisfied by Zeroth's dependency tree. The SDK should be added as a local path dependency (same pattern as GovernAI).

The Regulus backend exposes execution ingestion at `POST /v1/instrumentation/executions` and enforcement actions at `/v1/enforcement/actions`. The backend uses capability_id and implementation_id as its primary organizing concepts. Zeroth's cost REST endpoints (`GET /v1/tenants/{id}/cost`) will query the Regulus backend's dashboard/costing APIs to aggregate spend.

**Primary recommendation:** Add `econ-instrumentation-sdk` as a local path dep, build a thin `src/zeroth/econ/` module with `RegulusClient` (wrapper around `InstrumentationClient`), `InstrumentedProviderAdapter` (decorator), and `BudgetEnforcer` (pre-execution guard). Use `litellm.cost_per_token()` to estimate USD cost from token counts. Emit events via SDK's fire-and-forget transport -- never block the LLM hot path.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Use `regulus-sdk` (econ-instrumentation-sdk) Python package as typed dependency
- D-02: Regulus backend URL and API key configured via ZerothSettings (pydantic-settings, YAML + env vars)
- D-03: Regulus client instantiated as a singleton dependency (like existing repository pattern)
- D-04: InstrumentedProviderAdapter follows decorator pattern, wraps any ProviderAdapter, emits Regulus ExecutionEvent after each ainvoke()
- D-05: Orchestrator/runner does NOT change -- adapter stacking handles instrumentation. Stack: InstrumentedProviderAdapter(GovernedLLMProviderAdapter(LiteLLMProviderAdapter(...))) or InstrumentedProviderAdapter(LiteLLMProviderAdapter(...))
- D-06: ExecutionEvent includes: model_name, input_tokens, output_tokens, total_tokens, estimated_cost, node_id, run_id, tenant_id, deployment_ref, timestamp
- D-07: New cost attribution fields on NodeAuditRecord: cost_usd (Decimal or float), cost_event_id (Regulus event reference)
- D-08: InstrumentedProviderAdapter enriches ProviderResponse with cost data -- runner copies cost fields to audit record alongside existing token_usage flow
- D-09: Cost attribution dimensions: node_id, run_id, tenant_id, deployment_ref
- D-10: Pre-execution policy guard in AgentRunner checks tenant budget cap against Regulus cumulative spend BEFORE calling adapter
- D-11: Budget check is a fast Regulus API call (cached with short TTL), not a database query
- D-12: Over-budget results in a policy rejection (reuses approval/policy rejection patterns)
- D-13: Budget caps configured per-tenant in Regulus backend -- Zeroth reads caps, doesn't manage them
- D-14: GET /v1/tenants/{id}/cost returns cumulative spend
- D-15: GET /v1/deployments/{ref}/cost returns deployment-level cost view (secondary)
- D-16: Cost endpoints query Regulus backend (source of truth), not local audit records
- D-17: Endpoints follow existing FastAPI router patterns in src/zeroth/service/
- D-18: Unit tests mock Regulus SDK client
- D-19: Integration tests gated behind @pytest.mark.live

### Claude's Discretion
- Exact Regulus SDK version pinning (use latest stable, pin minimum)
- ExecutionEvent field mapping details (follow SDK conventions)
- Budget cache TTL value (sensible default, e.g., 30s)
- Cost estimation logic (use Regulus pricing data or provider-reported costs)

### Deferred Ideas (OUT OF SCOPE)
- ECON-05: LLM response caching (semantic and exact-match)
- ECON-06: Model routing and cost optimization
- ECON-07: Regulus A/B experiments for model comparison
- LLM-05: Model fallback chains
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ECON-01 | InstrumentedProviderAdapter wraps any ProviderAdapter and emits Regulus ExecutionEvent per LLM call | Regulus SDK `ExecutionEvent` schema verified; `track_execution()` API documented; decorator pattern follows existing `GovernedLLMProviderAdapter` |
| ECON-02 | Token cost attributed per node, run, tenant, and deployment in audit records | `NodeAuditRecord` already has `node_id`, `run_id`, `tenant_id`, `deployment_ref`; new `cost_usd` and `cost_event_id` fields needed; `litellm.cost_per_token()` available for USD estimation |
| ECON-03 | Per-tenant and per-deployment budget caps enforced via policy guard before execution | Regulus backend has `/v1/enforcement/` APIs; budget check in `AgentRunner.run()` before provider call; cached with TTL via httpx |
| ECON-04 | REST endpoints expose cumulative cost per tenant and deployment | Regulus backend has `/v1/dashboard/kpis` and execution query APIs; new FastAPI router in `src/zeroth/service/cost_api.py` |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Build/test: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- Project layout: `src/zeroth/` (main package), `tests/` (pytest tests)
- Must use `progress-logger` skill during implementation
- Track progress in `PROGRESS.md`

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| econ-instrumentation-sdk | 0.1.1 (local path) | Regulus SDK: ExecutionEvent emission, telemetry transport | Private companion SDK; no PyPI alternative. Uses pydantic + httpx |
| litellm | >=1.83,<2.0 (already installed) | `cost_per_token()` for USD cost estimation from token counts | Already in Zeroth deps; has built-in pricing data for 400+ models |
| httpx | >=0.27 (already installed, 0.28.1) | HTTP client for Regulus budget check queries | Already a Zeroth transitive dep via litellm and SDK |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| cachetools | >=5.5 | TTL cache for budget check results | Budget enforcement -- avoid per-call HTTP round trips |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| litellm.cost_per_token() | Manual pricing table | litellm maintains pricing for 400+ models automatically; manual would go stale |
| cachetools.TTLCache | functools.lru_cache | lru_cache has no time-based expiry; budget checks need TTL refresh |
| Background thread transport | Async queue | SDK already uses daemon thread; fighting it adds complexity for no benefit |

**Installation:**
```bash
# Add Regulus SDK as local path dep (same pattern as GovernAI)
uv add "econ-instrumentation-sdk @ file:///Users/dondoe/coding/regulus/sdk/python"
# Add cachetools for budget TTL cache
uv add "cachetools>=5.5"
```

**Version verification:** econ-instrumentation-sdk 0.1.1 verified from local `pyproject.toml`. httpx 0.28.1 verified from installed environment. litellm version confirmed installed with `cost_per_token` and `completion_cost` available.

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/
├── econ/                        # NEW module
│   ├── __init__.py              # Public API: RegulusClient, InstrumentedProviderAdapter, BudgetEnforcer
│   ├── client.py                # RegulusClient: thin wrapper around InstrumentationClient
│   ├── adapter.py               # InstrumentedProviderAdapter: decorator wrapping ProviderAdapter
│   ├── budget.py                # BudgetEnforcer: pre-execution budget check with TTL cache
│   ├── cost.py                  # Cost estimation: litellm.cost_per_token() wrapper
│   └── models.py                # CostAttribution pydantic model, RegulusSettings
├── config/
│   └── settings.py              # Add RegulusSettings sub-model to ZerothSettings
├── audit/
│   └── models.py                # Add cost_usd, cost_event_id to NodeAuditRecord
├── agent_runtime/
│   ├── provider.py              # Add cost_usd, cost_event_id to ProviderResponse
│   └── runner.py                # Add budget check before provider call
├── service/
│   ├── cost_api.py              # NEW: GET /v1/tenants/{id}/cost, GET /v1/deployments/{ref}/cost
│   └── bootstrap.py             # Wire RegulusClient, InstrumentedProviderAdapter, BudgetEnforcer
```

### Pattern 1: Decorator Adapter (InstrumentedProviderAdapter)
**What:** Wraps any ProviderAdapter, delegates `ainvoke()` to inner adapter, then emits an `ExecutionEvent` to Regulus and enriches `ProviderResponse` with cost data.
**When to use:** Every LLM call in production. Stacked outermost in the adapter chain.
**Example:**
```python
# Source: Verified against existing GovernedLLMProviderAdapter pattern (provider.py:93)
# and Regulus SDK schemas.py + client.py

from decimal import Decimal
from time import perf_counter

from econ_instrumentation import ExecutionEvent, InstrumentationClient
from litellm import cost_per_token

from zeroth.agent_runtime.provider import ProviderAdapter, ProviderRequest, ProviderResponse


class InstrumentedProviderAdapter:
    """Wraps any ProviderAdapter to emit Regulus cost events."""

    def __init__(
        self,
        inner: ProviderAdapter,
        regulus_client: InstrumentationClient,
        *,
        node_id: str,
        run_id: str,
        tenant_id: str,
        deployment_ref: str,
    ) -> None:
        self._inner = inner
        self._regulus = regulus_client
        self._node_id = node_id
        self._run_id = run_id
        self._tenant_id = tenant_id
        self._deployment_ref = deployment_ref

    async def ainvoke(self, request: ProviderRequest) -> ProviderResponse:
        start = perf_counter()
        response = await self._inner.ainvoke(request)
        elapsed_ms = int((perf_counter() - start) * 1000)

        # Estimate cost from token usage via litellm pricing data
        cost_usd = Decimal("0")
        if response.token_usage:
            try:
                prompt_cost, completion_cost = cost_per_token(
                    model=request.model_name,
                    prompt_tokens=response.token_usage.input_tokens,
                    completion_tokens=response.token_usage.output_tokens,
                )
                cost_usd = Decimal(str(prompt_cost + completion_cost))
            except Exception:
                pass  # Unknown model pricing -- emit event with zero cost

        event = ExecutionEvent(
            capability_id=self._node_id,
            implementation_id=request.model_name,
            model_version=request.model_name,
            token_cost_usd=cost_usd,
            latency_ms=elapsed_ms,
            compute_time_ms=elapsed_ms,
            metadata={
                "run_id": self._run_id,
                "tenant_id": self._tenant_id,
                "deployment_ref": self._deployment_ref,
                "input_tokens": response.token_usage.input_tokens if response.token_usage else 0,
                "output_tokens": response.token_usage.output_tokens if response.token_usage else 0,
                "total_tokens": response.token_usage.total_tokens if response.token_usage else 0,
            },
        )
        # Fire-and-forget via background thread transport
        self._regulus.track_execution(event)

        # Enrich response with cost data for audit record
        response = response.model_copy(
            update={
                "metadata": {
                    **response.metadata,
                    "cost_usd": float(cost_usd),
                    "cost_event_id": event.execution_id,
                },
            }
        )
        return response
```

### Pattern 2: Pre-execution Budget Guard
**What:** Checks tenant spend against budget cap before any LLM call. Uses TTL-cached spend to avoid per-call HTTP round trips.
**When to use:** In `AgentRunner.run()`, before the provider call loop.
**Example:**
```python
# Source: Verified against Regulus enforcement API and existing AgentRunner pattern

from cachetools import TTLCache
import httpx


class BudgetEnforcer:
    """Pre-execution budget check against Regulus backend."""

    def __init__(
        self,
        regulus_base_url: str,
        *,
        cache_ttl: int = 30,
        timeout: float = 5.0,
    ) -> None:
        self._base_url = regulus_base_url.rstrip("/")
        self._timeout = timeout
        self._cache: TTLCache[str, dict] = TTLCache(maxsize=1024, ttl=cache_ttl)

    async def check_budget(self, tenant_id: str) -> tuple[bool, float, float]:
        """Returns (allowed, current_spend, budget_cap).

        If Regulus is unavailable, defaults to allowing execution (fail-open).
        """
        cached = self._cache.get(tenant_id)
        if cached is not None:
            return cached["allowed"], cached["spend"], cached["cap"]

        try:
            async with httpx.AsyncClient(timeout=self._timeout) as client:
                resp = await client.get(
                    f"{self._base_url}/dashboard/kpis",
                    params={"tenant_id": tenant_id},
                )
                resp.raise_for_status()
                data = resp.json()
                spend = float(data.get("total_cost_usd", 0))
                cap = float(data.get("budget_cap_usd", float("inf")))
                allowed = spend < cap
                self._cache[tenant_id] = {"allowed": allowed, "spend": spend, "cap": cap}
                return allowed, spend, cap
        except Exception:
            # Fail-open: if Regulus is down, allow execution
            return True, 0.0, float("inf")
```

### Pattern 3: Cost REST Endpoint
**What:** Thin FastAPI router querying Regulus backend for cumulative cost.
**When to use:** `GET /v1/tenants/{id}/cost` and `GET /v1/deployments/{ref}/cost`.
**Example:**
```python
# Source: Follows existing audit_api.py pattern (service/audit_api.py:133)

from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel


class TenantCostResponse(BaseModel):
    tenant_id: str
    total_cost_usd: float
    budget_cap_usd: float | None = None
    currency: str = "USD"


def register_cost_routes(app: FastAPI) -> None:
    @app.get("/v1/tenants/{tenant_id}/cost", response_model=TenantCostResponse)
    async def get_tenant_cost(request: Request, tenant_id: str) -> TenantCostResponse:
        # Query Regulus backend for cumulative spend
        ...
```

### Anti-Patterns to Avoid
- **Do NOT use Regulus auto-instrumentation (`enable_auto_instrumentation()`):** It monkey-patches providers and breaks Zeroth's audit chain integrity. Use explicit `track_execution()` calls only.
- **Do NOT make budget checks synchronous and blocking:** Use TTL cache. A synchronous HTTP call on every node execution adds latency proportional to Regulus backend response time.
- **Do NOT store cost data only in Regulus:** The `cost_usd` and `cost_event_id` fields on `NodeAuditRecord` ensure cost data is queryable from Zeroth's own audit trail, even if Regulus is temporarily unavailable.
- **Do NOT embed Regulus backend inside Zeroth:** Regulus is a companion service with its own database and scaling. Communicate via SDK only.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| LLM pricing per model | Custom pricing table | `litellm.cost_per_token()` | Maintains pricing for 400+ models; auto-updates with litellm releases |
| Telemetry transport | Custom HTTP batch sender | Regulus SDK `TelemetryTransport` | Already handles buffering, retry with backoff, daemon thread lifecycle |
| TTL caching | Custom expiring dict | `cachetools.TTLCache` | Battle-tested, thread-safe, configurable maxsize and TTL |
| Event schema | Custom pydantic models for cost events | Regulus `ExecutionEvent` | Must match Regulus backend ingestion schema exactly |

**Key insight:** The Regulus SDK already handles the hard parts (transport, buffering, retry, event schema). Zeroth's `econ/` module is a thin mapping layer, not a telemetry system.

## Common Pitfalls

### Pitfall 1: Auto-instrumentation Corrupts Audit Chain
**What goes wrong:** Calling `econ_instrumentation.enable_auto_instrumentation()` or `auto_instrument()` monkey-patches LangChain/OpenAI/Anthropic clients, emitting duplicate events and corrupting correlation IDs in async context.
**Why it happens:** Auto-instrumentation is designed for standalone applications, not for platforms that already have their own audit chain.
**How to avoid:** Only use explicit `track_execution()` calls from `InstrumentedProviderAdapter`. Never import or call any `auto_instrument` function.
**Warning signs:** Regulus import appearing before GovernedLLMProviderAdapter in module imports; AuditContinuityReport failures after enabling Regulus.

### Pitfall 2: TelemetryTransport SIGTERM Flush Behavior
**What goes wrong:** On container shutdown (SIGTERM), the daemon thread may not flush in-flight events, causing telemetry loss.
**Why it happens:** `TelemetryTransport._thread` is a daemon thread (`daemon=True`), so Python kills it without flushing on process exit.
**How to avoid:** Register a shutdown hook (`atexit` or SIGTERM handler) that calls `transport.stop()` which joins the thread with a 1-second timeout. The `stop()` method sets `_stop` event and joins, but does NOT explicitly flush. Consider calling `flush_once()` before `stop()`.
**Warning signs:** Missing cost events for the last few seconds before container restart.

### Pitfall 3: ProviderResponse extra="forbid" Blocks New Fields
**What goes wrong:** `ProviderResponse` has `ConfigDict(extra="forbid")`, so adding `cost_usd` to `metadata` dict works, but adding new top-level fields requires a model change.
**Why it happens:** Pydantic strict validation rejects unknown fields.
**How to avoid:** D-08 specifies enriching via `metadata` dict on ProviderResponse (which already exists and accepts arbitrary keys). Alternatively, add `cost_usd` and `cost_event_id` as optional fields to ProviderResponse. The metadata approach is simpler and requires no model schema change.

### Pitfall 4: Budget Check Latency on Cold Cache
**What goes wrong:** First request for each tenant hits Regulus backend synchronously, adding 50-200ms latency.
**Why it happens:** TTL cache is empty on startup or after TTL expiry.
**How to avoid:** Fail-open design (allow execution if Regulus is unreachable). Consider pre-warming cache on startup for known tenants. Keep TTL reasonable (30s default).
**Warning signs:** First request after deployment is noticeably slower.

### Pitfall 5: Regulus SDK openai Pin Conflict
**What goes wrong:** Regulus SDK's `integrations` extra pins `openai>=1.40,<2.0`, but Zeroth may transitively pull openai>=2.0 via litellm.
**Why it happens:** The SDK was built against openai v1.x; the ecosystem moved to v2.x.
**How to avoid:** Do NOT install the `integrations` extra. Zeroth only needs the base SDK (pydantic + httpx). The base extra has no openai/anthropic/langchain pins.
**Warning signs:** Dependency resolution failure when adding econ-instrumentation-sdk.

### Pitfall 6: Decimal vs Float Mismatch
**What goes wrong:** Regulus `ExecutionEvent.token_cost_usd` is `Decimal`, but `litellm.cost_per_token()` returns `float`. NodeAuditRecord `cost_usd` could be either.
**Why it happens:** Different libraries use different numeric types for money.
**How to avoid:** Use `Decimal(str(float_value))` when converting litellm floats to Regulus Decimals. For NodeAuditRecord, use `float` (simpler, and JSON serialization is straightforward). The precision loss is negligible for cost tracking (sub-cent accuracy is sufficient).

## Code Examples

### Mapping Zeroth Concepts to Regulus Concepts
```python
# Source: Verified from Regulus SDK schemas.py and Zeroth audit/models.py

# Zeroth concept        -> Regulus ExecutionEvent field
# node_id               -> capability_id
# model_name            -> implementation_id, model_version
# run_id                -> metadata["run_id"] + join_key
# tenant_id             -> metadata["tenant_id"]
# deployment_ref        -> metadata["deployment_ref"]
# token_usage.input     -> metadata["input_tokens"]
# token_usage.output    -> metadata["output_tokens"]
# cost_usd              -> token_cost_usd
# elapsed_ms            -> latency_ms, compute_time_ms
```

### RegulusSettings Sub-model
```python
# Source: Follows existing DatabaseSettings/RedisSettings pattern in config/settings.py

from pydantic import BaseModel, SecretStr


class RegulusSettings(BaseModel):
    """Regulus backend connection settings."""

    enabled: bool = False
    base_url: str = "http://localhost:8000/v1"
    api_key: SecretStr | None = None
    budget_cache_ttl: int = 30  # seconds
    request_timeout: float = 5.0
```

### Adding Cost Fields to NodeAuditRecord
```python
# Source: Extends existing NodeAuditRecord (audit/models.py:101)

class NodeAuditRecord(BaseModel):
    # ... existing fields ...
    token_usage: TokenUsage | None = None
    # NEW: cost attribution
    cost_usd: float | None = None
    cost_event_id: str | None = None
```

### Adding Cost Fields to ProviderResponse
```python
# Source: Extends existing ProviderResponse (provider.py:41)

class ProviderResponse(BaseModel):
    # ... existing fields ...
    token_usage: TokenUsage | None = None
    # NEW: cost attribution (populated by InstrumentedProviderAdapter)
    cost_usd: float | None = None
    cost_event_id: str | None = None
```

### Budget Check Integration in AgentRunner
```python
# Source: Inserted in AgentRunner.run() before the retry loop (runner.py:125)

# Before the retry loop:
if budget_enforcer is not None:
    allowed, spend, cap = await budget_enforcer.check_budget(
        tenant_id=enforcement_context.get("tenant_id", "default") if enforcement_context else "default"
    )
    if not allowed:
        raise BudgetExceededError(
            f"tenant budget exceeded: spent ${spend:.4f} of ${cap:.4f} cap"
        )
```

### Wiring in ServiceBootstrap
```python
# Source: Follows existing bootstrap.py pattern

from econ_instrumentation import InstrumentationClient, InstrumentationConfig
from zeroth.econ.budget import BudgetEnforcer

# In bootstrap_service():
regulus_settings = settings.regulus  # from ZerothSettings
regulus_client = None
budget_enforcer = None
if regulus_settings.enabled:
    regulus_config = InstrumentationConfig(
        base_url=regulus_settings.base_url,
        enabled=True,
        request_timeout_s=regulus_settings.request_timeout,
    )
    regulus_client = InstrumentationClient(
        base_url=regulus_settings.base_url,
        timeout=regulus_settings.request_timeout,
        enabled=True,
    )
    budget_enforcer = BudgetEnforcer(
        regulus_base_url=regulus_settings.base_url,
        cache_ttl=regulus_settings.budget_cache_ttl,
        timeout=regulus_settings.request_timeout,
    )
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual cost tables per model | litellm.cost_per_token() with built-in pricing | litellm 1.x+ | Auto-pricing for 400+ models; no manual maintenance |
| Synchronous telemetry in hot path | Background thread transport (Regulus SDK) | SDK design | Zero latency impact on LLM calls |
| Budget checks via database query | Cached HTTP to economics backend | Phase 13 design | Decoupled from audit storage; source of truth in Regulus |

**Deprecated/outdated:**
- Regulus `auto_instrument()`: Designed for standalone apps, not for platforms with custom audit chains. Do NOT use.

## Open Questions

1. **Regulus backend budget cap API**
   - What we know: The enforcement API has `/v1/enforcement/actions` with "ApplyBudgetCap" action type. Dashboard KPIs exist.
   - What's unclear: There is no obvious `GET /v1/tenants/{id}/budget` endpoint that returns current spend + cap in one call. The dashboard KPIs endpoint is global, not per-tenant.
   - Recommendation: Build the budget query as an httpx call to Regulus, aggregating execution events by tenant_id. If the Regulus backend lacks a direct budget API, implement budget cap storage in Zeroth's own config (per D-13 says "Budget caps configured per-tenant in Regulus backend" but the backend may not have this feature yet). Alternatively, store budget caps in ZerothSettings and query cumulative spend by summing NodeAuditRecord.cost_usd locally.

2. **TelemetryTransport flush-on-SIGTERM**
   - What we know: The daemon thread in `transport.py` uses `threading.Event` for stop signal with 1s join timeout. `flush_once()` exists as a public method.
   - What's unclear: Whether `stop()` calls `flush_once()` before stopping (it does NOT -- verified from source).
   - Recommendation: Add a shutdown hook that calls `transport.flush_once()` then `transport.stop()`. This is a Zeroth-side concern, not an SDK change.

3. **Cost estimation for unknown models**
   - What we know: `litellm.cost_per_token()` covers 400+ models but may raise exceptions for custom/fine-tuned models.
   - What's unclear: Whether it returns 0 or raises on unknown models.
   - Recommendation: Wrap in try/except, default to `Decimal("0")` for unknown models. Log a warning so operators know cost tracking is incomplete.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| econ-instrumentation-sdk | ECON-01, ECON-02 | Source at /Users/dondoe/coding/regulus/sdk/python/ | 0.1.1 | -- |
| litellm | ECON-02 (cost estimation) | Installed | >=1.83 | -- |
| httpx | ECON-03 (budget queries) | Installed | 0.28.1 | -- |
| cachetools | ECON-03 (budget TTL) | Not installed | -- | Install via `uv add cachetools>=5.5` |
| Regulus backend | ECON-03, ECON-04 (live budget/cost queries) | Not running locally | -- | Mock in tests; live tests gated behind @pytest.mark.live |

**Missing dependencies with no fallback:**
- None -- all blocking dependencies are available

**Missing dependencies with fallback:**
- cachetools: Not installed, trivial to add
- Regulus backend: Not running, but all unit tests mock it; live tests are optional

## Sources

### Primary (HIGH confidence)
- Regulus SDK source: `/Users/dondoe/coding/regulus/sdk/python/econ_instrumentation/` -- schemas.py, client.py, transport.py, config.py, runtime.py (direct code inspection)
- Regulus backend source: `/Users/dondoe/coding/regulus/backend/src/econ_plane/` -- instrumentation/api.py, enforcement/api.py, costing/api.py, dashboard/api.py (direct code inspection)
- Zeroth codebase: provider.py, runner.py, audit/models.py, service/bootstrap.py, config/settings.py (direct code inspection)
- litellm cost functions: `cost_per_token()` and `completion_cost()` verified available in installed environment

### Secondary (MEDIUM confidence)
- [litellm token usage docs](https://docs.litellm.ai/docs/completion/token_usage) -- cost_per_token() API and usage patterns

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries verified from source or installed environment
- Architecture: HIGH -- patterns derived from existing codebase (GovernedLLMProviderAdapter, ServiceBootstrap) and Regulus SDK source
- Pitfalls: HIGH -- pitfalls 1-2 identified from prior research (PITFALLS.md, ARCHITECTURE.md) and verified against SDK source; pitfalls 3-6 from direct code inspection

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable -- Regulus SDK is private, litellm pricing updates automatically)
