# Phase 35: Resilient HTTP Client - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Provide a platform-managed async HTTP client (wrapping httpx) that agent tools and executable units use for external HTTP calls. The client handles retry with backoff, per-endpoint circuit breaking, connection pooling, automatic auth header resolution from SecretResolver, capability gating (NETWORK_READ/WRITE/EXTERNAL_API_CALL), and audit logging of every call.

</domain>

<decisions>
## Implementation Decisions

### Client Architecture
- **D-01:** `ResilientHttpClient` wraps `httpx.AsyncClient` — not a new HTTP implementation. Adds retry, circuit breaking, and governance layers on top.
- **D-02:** Client is placed in a new `zeroth.core.http` package (models.py, client.py, circuit_breaker.py, errors.py, __init__.py).
- **D-03:** Custom circuit breaker implementation (~60 LOC) per PROJECT.md decision — no external dependency (aiobreaker/pybreaker unmaintained).
- **D-04:** `HttpClientSettings` added to `ZerothSettings` with sensible defaults (max_retries=3, retry_backoff_base=0.5, circuit_breaker_threshold=5, circuit_breaker_reset_timeout=30, pool_max_connections=100, pool_max_keepalive=20).

### Retry & Circuit Breaking
- **D-05:** Exponential backoff with jitter: `delay = backoff_base * (2 ** attempt) + random(0, 0.1)`. Retryable status codes configurable, default: {408, 429, 500, 502, 503, 504}.
- **D-06:** Per-endpoint circuit breaker: tracks failure count per (host, port) tuple. States: CLOSED (normal) -> OPEN (after threshold failures) -> HALF_OPEN (after reset timeout, lets 1 request through). Thread-safe via asyncio.Lock.
- **D-07:** Circuit breaker state is in-memory only — no persistence needed. Resets on process restart.

### Connection Pooling & Auth
- **D-08:** Single shared `httpx.AsyncClient` per `ResilientHttpClient` instance with configurable pool limits. Per-tenant pooling is achieved by creating separate client instances per tenant at bootstrap.
- **D-09:** Auth header resolution: `ResilientHttpClient` accepts an optional `SecretResolver`. Before each request, if endpoint config specifies a secret key, the resolver injects the auth header (Bearer token, API key, or custom header).

### Governance Integration
- **D-10:** Every HTTP call checks capabilities via `PolicyGuard.check_capability()` before execution: GET/HEAD -> NETWORK_READ, POST/PUT/PATCH/DELETE -> NETWORK_WRITE, all -> EXTERNAL_API_CALL.
- **D-11:** Every HTTP call produces an audit record: URL (with query params redacted), method, status code, latency_ms, response_size_bytes. Uses the existing `NodeAuditRecord` pattern.
- **D-12:** Rate limiting per endpoint: token bucket algorithm with configurable rate and burst. Shared across requests to the same endpoint.

### Claude's Discretion
- Whether to use `httpx.Limits` directly or wrap with custom pool management
- Rate limiter implementation details (asyncio.Semaphore vs custom token bucket)
- Error class hierarchy structure
- Test fixture strategy for HTTP mocking (respx vs httpx mock transport)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### HTTP & Networking
- `src/zeroth/core/webhooks/delivery.py` — Existing httpx usage for webhook delivery (retry patterns)
- `src/zeroth/core/execution_units/sidecar_client.py` — Existing httpx usage for sidecar communication

### Governance
- `src/zeroth/core/policy/models.py` — Capability enum (NETWORK_READ, NETWORK_WRITE, EXTERNAL_API_CALL already defined)
- `src/zeroth/core/policy/guard.py` — PolicyGuard.check_capability() — governance gating

### Audit
- `src/zeroth/core/audit/models.py` — NodeAuditRecord pattern for audit logging

### Secrets
- `src/zeroth/core/secrets/provider.py` — SecretProvider(Protocol) — auth header resolution

### Settings
- `src/zeroth/core/config/settings.py` — ZerothSettings with sub-models

### Bootstrap
- `src/zeroth/core/service/bootstrap.py` — Service initialization (where client gets created)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `httpx` already in dependencies (used by webhooks, sidecar, health, cost API)
- `PolicyGuard` for capability checking
- `SecretProvider(Protocol)` for auth resolution
- `NodeAuditRecord` for audit logging pattern
- Capability enum already has NETWORK_READ, NETWORK_WRITE, EXTERNAL_API_CALL

### Established Patterns
- Protocol for interfaces (same as ArtifactStore, ThreadStateStore)
- Settings sub-models in ZerothSettings
- Package structure: models.py, errors.py, __init__.py
- ConfigDict(extra="forbid") on all Pydantic models

### Integration Points
- `service/bootstrap.py` — Construct ResilientHttpClient at startup
- `agent_runtime/tools.py` — Make client available to agent tools
- `execution_units/runner.py` — Make client available to execution units
- `policy/guard.py` — Capability gating before HTTP calls
- `audit/` — Logging HTTP call records

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Implementation well-constrained by HTTP-01 through HTTP-06.

</specifics>

<deferred>
## Deferred Ideas

- HTTP response caching — explicitly out of scope per FUTURE-05 in REQUIREMENTS.md
- Distributed circuit breaker state (Redis-backed) — in-memory sufficient for v4.0

</deferred>

---

*Phase: 35-resilient-http-client*
*Context gathered: 2026-04-13*
