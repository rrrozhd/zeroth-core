# Phase 35: Resilient HTTP Client - Research

**Researched:** 2026-04-12
**Domain:** Async HTTP client resilience patterns (retry, circuit breaking, connection pooling, governance)
**Confidence:** HIGH

## Summary

This phase builds a `ResilientHttpClient` wrapper around `httpx.AsyncClient` that adds retry with exponential backoff, per-endpoint circuit breaking, connection pooling, automatic auth header resolution, governance capability gating, audit logging, and per-endpoint rate limiting. The entire implementation uses only the existing `httpx` dependency (already installed at 0.28.1) and in-memory data structures -- no new packages are needed.

The codebase already contains all the integration points: `PolicyGuard.evaluate()` for capability checking, `Capability` enum with `NETWORK_READ`/`NETWORK_WRITE`/`EXTERNAL_API_CALL` already defined, `SecretProvider(Protocol)` for auth resolution, `NodeAuditRecord` with `execution_metadata` dict for logging HTTP call details, and `ZerothSettings` for configuration. The existing `webhooks/delivery.py` provides a proven retry+backoff pattern and `guardrails/rate_limit.py` provides a token bucket implementation (database-backed), though the HTTP client needs an in-memory variant for low-latency enforcement.

**Primary recommendation:** Create `zeroth.core.http` package with 5 files (models.py, circuit_breaker.py, client.py, errors.py, __init__.py), add `HttpClientSettings` to `ZerothSettings`, wire into `ServiceBootstrap`, and expose to agent tools and execution units via orchestrator injection.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `ResilientHttpClient` wraps `httpx.AsyncClient` -- not a new HTTP implementation. Adds retry, circuit breaking, and governance layers on top.
- **D-02:** Client is placed in a new `zeroth.core.http` package (models.py, client.py, circuit_breaker.py, errors.py, __init__.py).
- **D-03:** Custom circuit breaker implementation (~60 LOC) per PROJECT.md decision -- no external dependency (aiobreaker/pybreaker unmaintained).
- **D-04:** `HttpClientSettings` added to `ZerothSettings` with sensible defaults (max_retries=3, retry_backoff_base=0.5, circuit_breaker_threshold=5, circuit_breaker_reset_timeout=30, pool_max_connections=100, pool_max_keepalive=20).
- **D-05:** Exponential backoff with jitter: `delay = backoff_base * (2 ** attempt) + random(0, 0.1)`. Retryable status codes configurable, default: {408, 429, 500, 502, 503, 504}.
- **D-06:** Per-endpoint circuit breaker: tracks failure count per (host, port) tuple. States: CLOSED -> OPEN -> HALF_OPEN. Thread-safe via asyncio.Lock.
- **D-07:** Circuit breaker state is in-memory only -- no persistence. Resets on process restart.
- **D-08:** Single shared `httpx.AsyncClient` per `ResilientHttpClient` instance with configurable pool limits. Per-tenant pooling by creating separate client instances per tenant at bootstrap.
- **D-09:** Auth header resolution: accepts optional `SecretResolver`. Before each request, if endpoint config specifies a secret key, resolver injects auth header (Bearer token, API key, or custom header).
- **D-10:** Every HTTP call checks capabilities via `PolicyGuard.evaluate()` before execution: GET/HEAD -> NETWORK_READ, POST/PUT/PATCH/DELETE -> NETWORK_WRITE, all -> EXTERNAL_API_CALL.
- **D-11:** Every HTTP call produces an audit record: URL (with query params redacted), method, status code, latency_ms, response_size_bytes. Uses existing `NodeAuditRecord` pattern.
- **D-12:** Rate limiting per endpoint: token bucket algorithm with configurable rate and burst. Shared across requests to the same endpoint.

### Claude's Discretion
- Whether to use `httpx.Limits` directly or wrap with custom pool management
- Rate limiter implementation details (asyncio.Semaphore vs custom token bucket)
- Error class hierarchy structure
- Test fixture strategy for HTTP mocking (respx vs httpx mock transport)

### Deferred Ideas (OUT OF SCOPE)
- HTTP response caching -- explicitly out of scope per FUTURE-05 in REQUIREMENTS.md
- Distributed circuit breaker state (Redis-backed) -- in-memory sufficient for v4.0

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| HTTP-01 | Platform-provided async HTTP client available to agent tools and executable units, configurable per-node or per-tool | `ResilientHttpClient` class wrapping httpx.AsyncClient; `HttpClientSettings` in ZerothSettings; wired via ServiceBootstrap; exposed to orchestrator for injection |
| HTTP-02 | Configurable retry with exponential backoff and jitter; retryable status codes configurable (default: 408, 429, 5xx) | Retry loop in `client.py` with D-05 formula; matches existing `next_retry_delay` pattern from webhooks/delivery.py |
| HTTP-03 | Per-endpoint circuit breaker with configurable failure threshold and reset timeout | Custom ~60 LOC circuit breaker in `circuit_breaker.py` with CLOSED/OPEN/HALF_OPEN states keyed by (host, port); asyncio.Lock for thread safety |
| HTTP-04 | Shared or per-tenant connection pools with configurable limits | `httpx.Limits` directly on the wrapped `httpx.AsyncClient`; per-tenant achieved via separate `ResilientHttpClient` instances |
| HTTP-05 | External HTTP calls gated by capabilities, logged in audit records, subject to rate limiting | Capability check via `PolicyGuard.evaluate()` with NETWORK_READ/NETWORK_WRITE/EXTERNAL_API_CALL; audit via execution_metadata dict pattern; in-memory token bucket rate limiter |
| HTTP-06 | HTTP client resolves auth headers/tokens from SecretResolver automatically based on configuration | Optional `SecretProvider` on client; endpoint config specifies secret_key and auth_type (bearer/api_key/custom_header); resolver called pre-request |

</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| httpx | 0.28.1 | Async HTTP client foundation | Already in project deps, used by webhooks, sidecar, health, cost API, LiteLLM [VERIFIED: uv pip show] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| httpx.MockTransport | 0.28.1 (built-in) | Test HTTP mocking without external dep | Testing the resilient client -- avoids adding respx as a new dep [VERIFIED: `help(httpx.MockTransport)` in runtime] |
| httpx.Limits | 0.28.1 (built-in) | Connection pool configuration | Configuring max_connections and max_keepalive_connections on AsyncClient [VERIFIED: `help(httpx.Limits)` in runtime] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| httpx.MockTransport | respx | respx adds an external dep not currently in project; MockTransport is sufficient for testing controlled request/response pairs |
| Custom circuit breaker | aiobreaker/pybreaker | Both unmaintained per PROJECT.md decision; custom ~60 LOC is simpler and avoids dep risk [VERIFIED: CONTEXT.md D-03] |
| In-memory token bucket | Database-backed TokenBucketRateLimiter | Existing rate_limit.py is database-backed for durability; HTTP rate limiting needs sub-ms latency, so in-memory is correct |

**Installation:**
```bash
# No new packages needed. httpx already installed.
uv sync
```

**Version verification:**
- httpx 0.28.1 installed [VERIFIED: `uv pip show httpx` output 0.28.1, required `>=0.27` in pyproject.toml]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/http/
    __init__.py          # Public API re-exports
    models.py            # HttpClientSettings, EndpointConfig, HttpCallRecord (Pydantic)
    circuit_breaker.py   # CircuitBreaker, CircuitBreakerRegistry, CircuitState
    client.py            # ResilientHttpClient (main class)
    errors.py            # Error hierarchy
```

### Pattern 1: Layered Request Pipeline
**What:** Each HTTP request passes through a pipeline: rate limit check -> capability gate -> auth resolution -> circuit breaker check -> actual HTTP call with retry -> audit record creation
**When to use:** Every call through `ResilientHttpClient.request()`
**Example:**
```python
# Source: Synthesized from existing codebase patterns [ASSUMED]
async def request(
    self,
    method: str,
    url: str,
    *,
    endpoint_config: EndpointConfig | None = None,
    # standard httpx kwargs
    **kwargs: Any,
) -> httpx.Response:
    config = endpoint_config or EndpointConfig()
    endpoint_key = self._endpoint_key(url)

    # 1. Rate limit check
    if not await self._rate_limiter.acquire(endpoint_key, config.rate_limit):
        raise HttpRateLimitError(f"rate limit exceeded for {endpoint_key}")

    # 2. Capability gate
    self._check_capabilities(method)

    # 3. Auth resolution
    headers = dict(kwargs.pop("headers", {}) or {})
    if config.secret_key and self._secret_provider:
        headers.update(self._resolve_auth(config))

    # 4. Circuit breaker check
    breaker = self._circuit_breakers.get(endpoint_key)
    if breaker.is_open:
        raise CircuitOpenError(endpoint_key)

    # 5. Retry loop with backoff
    start = time.monotonic()
    last_exc: Exception | None = None
    for attempt in range(config.max_retries + 1):
        try:
            response = await self._client.request(method, url, headers=headers, **kwargs)
            if response.status_code in config.retryable_status_codes and attempt < config.max_retries:
                delay = self._backoff_delay(attempt, config)
                await asyncio.sleep(delay)
                continue
            breaker.record_success()
            # 6. Audit
            self._record_audit(method, url, response, start)
            return response
        except (httpx.TimeoutException, httpx.ConnectError) as exc:
            breaker.record_failure()
            last_exc = exc
            if attempt < config.max_retries:
                delay = self._backoff_delay(attempt, config)
                await asyncio.sleep(delay)

    # All retries exhausted
    self._record_audit(method, url, None, start, error=str(last_exc))
    raise HttpRetryExhaustedError(str(last_exc)) from last_exc
```

### Pattern 2: Circuit Breaker State Machine
**What:** Per-endpoint (host:port) circuit breaker with three states: CLOSED (passing requests), OPEN (rejecting requests), HALF_OPEN (allowing one probe request)
**When to use:** Automatic -- managed internally by `ResilientHttpClient`
**Example:**
```python
# Source: Standard circuit breaker pattern + asyncio.Lock for safety [ASSUMED]
import asyncio
import time
from enum import StrEnum

class CircuitState(StrEnum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, reset_timeout: float = 30.0) -> None:
        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._failure_threshold = failure_threshold
        self._reset_timeout = reset_timeout
        self._last_failure_time: float = 0.0
        self._lock = asyncio.Lock()

    @property
    def state(self) -> CircuitState:
        if self._state == CircuitState.OPEN:
            if time.monotonic() - self._last_failure_time >= self._reset_timeout:
                return CircuitState.HALF_OPEN
        return self._state

    async def record_success(self) -> None:
        async with self._lock:
            self._failure_count = 0
            self._state = CircuitState.CLOSED

    async def record_failure(self) -> None:
        async with self._lock:
            self._failure_count += 1
            self._last_failure_time = time.monotonic()
            if self._failure_count >= self._failure_threshold:
                self._state = CircuitState.OPEN

    async def check(self) -> None:
        state = self.state
        if state == CircuitState.OPEN:
            raise CircuitOpenError(...)
        if state == CircuitState.HALF_OPEN:
            # Allow one probe -- will transition on success/failure
            pass
```

### Pattern 3: In-Memory Token Bucket Rate Limiter
**What:** Per-endpoint rate limiting using token bucket algorithm with configurable rate and burst capacity, entirely in-memory (no database)
**When to use:** Enforced before every HTTP call to the same endpoint
**Example:**
```python
# Source: Adapted from guardrails/rate_limit.py pattern but in-memory [ASSUMED]
import asyncio
import time

class InMemoryTokenBucket:
    def __init__(self, rate: float, burst: int) -> None:
        self._rate = rate        # tokens per second
        self._burst = burst      # max tokens
        self._tokens = float(burst)
        self._last_refill = time.monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self) -> bool:
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self._last_refill
            self._tokens = min(self._burst, self._tokens + elapsed * self._rate)
            self._last_refill = now
            if self._tokens >= 1.0:
                self._tokens -= 1.0
                return True
            return False
```

### Pattern 4: Capability Gating by HTTP Method
**What:** Map HTTP methods to required capabilities before executing any request
**When to use:** Every request through the resilient client
**Example:**
```python
# Source: Existing Capability enum in policy/models.py [VERIFIED: policy/models.py]
from zeroth.core.policy.models import Capability

_METHOD_CAPABILITIES: dict[str, set[Capability]] = {
    "GET":     {Capability.NETWORK_READ, Capability.EXTERNAL_API_CALL},
    "HEAD":    {Capability.NETWORK_READ, Capability.EXTERNAL_API_CALL},
    "OPTIONS": {Capability.NETWORK_READ, Capability.EXTERNAL_API_CALL},
    "POST":    {Capability.NETWORK_WRITE, Capability.EXTERNAL_API_CALL},
    "PUT":     {Capability.NETWORK_WRITE, Capability.EXTERNAL_API_CALL},
    "PATCH":   {Capability.NETWORK_WRITE, Capability.EXTERNAL_API_CALL},
    "DELETE":  {Capability.NETWORK_WRITE, Capability.EXTERNAL_API_CALL},
}
```

### Anti-Patterns to Avoid
- **Don't retry non-idempotent POST/PUT without explicit configuration:** Retrying write requests can cause duplicates. Only retry on transport-level errors (connection reset, timeout) by default -- NOT on 5xx for write methods unless the caller opts in.
- **Don't share circuit breaker state across (host, port) tuples:** A failing endpoint on one host should not open the circuit for a healthy endpoint on another host. Key breakers by `(host, port)`.
- **Don't log full request/response bodies in audit:** Only log URL (query-redacted), method, status, latency, size. Body logging would be a security and storage problem.
- **Don't use `asyncio.Semaphore` for rate limiting:** Semaphore limits concurrency (number of in-flight requests) but does not limit request rate (requests per second). Token bucket is the correct algorithm for rate limiting.

## Project Constraints (from CLAUDE.md)

- Build/test commands: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- All source goes under `src/zeroth/core/`
- Tests go under `tests/`
- Progress logging is mandatory via `progress-logger` skill during implementation
- Backward compatibility: existing tests must continue passing

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP connection pooling | Custom pool manager | `httpx.Limits` on `httpx.AsyncClient` | httpx already manages connection pools via httpcore; configuring `Limits(max_connections=N, max_keepalive_connections=M)` handles pool sizing, keepalive, and cleanup [VERIFIED: httpx.Limits API inspection] |
| HTTP timeout handling | Manual asyncio.wait_for wrappers | `httpx.Timeout` on `httpx.AsyncClient` | httpx has separate connect/read/write/pool timeouts built in [VERIFIED: httpx.Timeout class inspection] |
| Mock HTTP for tests | Custom request interceptor | `httpx.MockTransport` | Built into httpx, works with both sync and async clients, no external dep needed [VERIFIED: httpx.MockTransport API inspection] |
| TLS/SSL management | Custom certificate handling | httpx default transport | httpx uses certifi and handles TLS transparently |

**Key insight:** httpx already handles the hard networking problems (connection pooling, TLS, timeouts, HTTP/2). The resilient client only adds application-level concerns: retry logic, circuit breaking, auth resolution, capability gating, audit logging, and rate limiting. These are all pure Python logic that wraps httpx calls.

## Common Pitfalls

### Pitfall 1: Circuit Breaker Race Condition on HALF_OPEN
**What goes wrong:** Multiple concurrent requests see HALF_OPEN state and all attempt to probe, defeating the purpose of allowing only one test request.
**Why it happens:** State check and state transition are not atomic without locking.
**How to avoid:** Use `asyncio.Lock` in the circuit breaker. When entering HALF_OPEN, atomically transition to HALF_OPEN and allow only one request through. Record the result to either close or re-open.
**Warning signs:** Circuit breaker oscillates rapidly between OPEN and HALF_OPEN under load.

### Pitfall 2: Retry Storm Amplification
**What goes wrong:** When an upstream service is struggling, retries from many clients amplify the load, making recovery harder.
**Why it happens:** All clients retry at the same intervals without jitter, creating synchronized retry waves.
**How to avoid:** D-05's formula includes jitter (`+ random(0, 0.1)`). Also, the circuit breaker (D-06) stops retries entirely once the failure threshold is hit, which is the primary defense against retry storms.
**Warning signs:** Upstream latency spikes correlate with client retry bursts.

### Pitfall 3: Secret Leakage in Audit Logs
**What goes wrong:** Auth headers containing Bearer tokens or API keys appear in audit records.
**Why it happens:** Audit logging captures headers without filtering.
**How to avoid:** Never log auth headers. The audit record (D-11) captures URL (query-redacted), method, status, latency, response size -- explicitly NOT headers. The URL redaction must strip query parameters (which may contain API keys).
**Warning signs:** Bearer tokens appearing in audit record searches.

### Pitfall 4: PolicyGuard API Mismatch
**What goes wrong:** CONTEXT.md references `PolicyGuard.check_capability()` but the actual API is `PolicyGuard.evaluate(graph, node, run, input_payload) -> EnforcementResult`.
**Why it happens:** The CONTEXT.md was written with a simplified mental model.
**How to avoid:** The HTTP client needs a simpler capability check interface. It should accept an `EnforcementResult` (or a set of effective capabilities) rather than calling PolicyGuard directly, because PolicyGuard needs graph/node/run context that the HTTP client does not have. The orchestrator should pass the enforcement result to the HTTP client as context. [VERIFIED: PolicyGuard.evaluate() signature in policy/guard.py]
**Warning signs:** Import cycles or awkward parameter passing if the HTTP client depends directly on PolicyGuard.

### Pitfall 5: Forgetting to Close httpx.AsyncClient
**What goes wrong:** Connection pool leaks, resource exhaustion, "unclosed client session" warnings.
**Why it happens:** `httpx.AsyncClient` maintains a connection pool that must be explicitly closed.
**How to avoid:** `ResilientHttpClient` must implement `async def aclose()` that calls `self._client.aclose()`. Bootstrap code must ensure cleanup on shutdown. Pattern already established in `SandboxSidecarClient.close()` [VERIFIED: sidecar_client.py line 52].
**Warning signs:** ResourceWarning about unclosed connections in test output.

### Pitfall 6: Retrying Non-Idempotent Requests
**What goes wrong:** POST requests are retried on 5xx, causing duplicate side effects.
**Why it happens:** Default retryable status codes include 500/502/503/504 regardless of method.
**How to avoid:** Consider making the default retryable set method-aware: for POST/PUT/PATCH/DELETE, only retry on transport-level errors (connection failure, timeout) unless explicitly configured. The per-endpoint config should allow overriding this.
**Warning signs:** Duplicate records created in external APIs.

## Code Examples

### HttpClientSettings Model
```python
# Source: Pattern from existing settings sub-models in config/settings.py [VERIFIED: settings.py]
from pydantic import BaseModel, ConfigDict

class HttpClientSettings(BaseModel):
    """Configuration for the platform HTTP client subsystem."""
    model_config = ConfigDict(extra="forbid")

    max_retries: int = 3
    retry_backoff_base: float = 0.5
    retry_max_delay: float = 60.0
    retryable_status_codes: set[int] = {408, 429, 500, 502, 503, 504}
    circuit_breaker_threshold: int = 5
    circuit_breaker_reset_timeout: float = 30.0
    pool_max_connections: int = 100
    pool_max_keepalive: int = 20
    pool_keepalive_expiry: float = 5.0
    default_timeout: float = 30.0
    default_rate_limit_rate: float = 10.0  # tokens per second
    default_rate_limit_burst: int = 20     # max burst tokens
```

### EndpointConfig Model
```python
# Source: Custom per-endpoint override model [ASSUMED]
from pydantic import BaseModel, ConfigDict

class AuthType(StrEnum):
    BEARER = "bearer"
    API_KEY = "api_key"
    CUSTOM_HEADER = "custom_header"

class EndpointConfig(BaseModel):
    """Per-endpoint configuration overrides."""
    model_config = ConfigDict(extra="forbid")

    max_retries: int | None = None  # None = use global default
    retryable_status_codes: set[int] | None = None
    timeout: float | None = None
    secret_key: str | None = None
    auth_type: AuthType | None = None
    auth_header_name: str = "Authorization"  # customizable for API key endpoints
    rate_limit_rate: float | None = None
    rate_limit_burst: int | None = None
```

### HttpCallRecord for Audit
```python
# Source: Pattern from NodeAuditRecord.execution_metadata [VERIFIED: audit/models.py]
from pydantic import BaseModel, ConfigDict

class HttpCallRecord(BaseModel):
    """Audit record for a single HTTP call. Stored in execution_metadata."""
    model_config = ConfigDict(extra="forbid")

    url: str           # query params redacted
    method: str
    status_code: int | None = None
    latency_ms: float
    response_size_bytes: int | None = None
    retry_count: int = 0
    circuit_breaker_state: str | None = None
    error: str | None = None
```

### Bootstrap Integration
```python
# Source: Pattern from bootstrap.py Phase 34 artifact store wiring [VERIFIED: bootstrap.py]
# In ServiceBootstrap dataclass:
http_client: object | None = None  # ResilientHttpClient instance

# In bootstrap_service():
from zeroth.core.http import ResilientHttpClient
http_settings = settings.http_client  # HttpClientSettings from ZerothSettings
http_client = ResilientHttpClient(
    settings=http_settings,
    secret_provider=secret_provider,  # from existing wiring
)
orchestrator.http_client = http_client
```

### URL Redaction for Audit
```python
# Source: Common URL sanitization pattern [ASSUMED]
from urllib.parse import urlparse, urlunparse

def redact_url(url: str) -> str:
    """Strip query parameters and fragment from URL for audit logging."""
    parsed = urlparse(url)
    return urlunparse((parsed.scheme, parsed.netloc, parsed.path, "", "", ""))
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| aiobreaker / pybreaker | Custom circuit breaker | 2024 (both unmaintained since 2022) | No maintained Python async circuit breaker library exists; ~60 LOC custom code is the community standard approach [ASSUMED] |
| httpx.AsyncHTTPTransport(retries=N) | Application-level retry with backoff | httpx 0.24+ | httpx built-in retries are connection-level only (TCP retries), not HTTP status code retries; application-level retry is standard for status-code-based retry logic [VERIFIED: httpx docs reference transport retries as connection-level] |

**Deprecated/outdated:**
- `httpx.AsyncHTTPTransport(retries=N)`: This only retries on connection-level failures, NOT on HTTP 429/5xx responses. Application-level retry with backoff is required for status code based retry.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | POST/PUT/PATCH/DELETE should be retry-cautious by default to avoid duplicate side effects | Anti-Patterns / Pitfall 6 | LOW -- can be overridden via per-endpoint config; conservative default is safer |
| A2 | `random(0, 0.1)` in D-05 jitter formula means `random.uniform(0, 0.1)` | Architecture Patterns | LOW -- trivially adjustable; matches existing `random.uniform` usage in webhooks/delivery.py |
| A3 | aiobreaker and pybreaker are unmaintained as of 2024 | State of the Art | LOW -- confirmed by CONTEXT.md D-03 which states this as a project decision |

## Open Questions

1. **Capability Enforcement Interface**
   - What we know: `PolicyGuard.evaluate()` requires `(graph, node, run, input_payload)` context. The HTTP client does not have access to these at call time.
   - What's unclear: Should the client accept a pre-computed `EnforcementResult` or a simple `set[Capability]` as the "granted capabilities" for the current execution context?
   - Recommendation: Accept `effective_capabilities: set[Capability] | None` passed from the orchestrator at request time. This avoids coupling the HTTP client to graph/node/run concepts and matches the pattern used in `ExecutableUnitRunner._apply_allowed_secrets()` which also receives enforcement context as a dict. If None, skip capability gating (for use outside governance context).

2. **Audit Record Attachment**
   - What we know: `NodeAuditRecord.execution_metadata` is a `dict[str, Any]` that gets populated during node execution. HTTP call records need to be attached to the current node's audit.
   - What's unclear: Should the HTTP client accumulate call records internally and expose them, or should it accept a callback/list to append records to?
   - Recommendation: The client should accumulate `HttpCallRecord` entries in an internal list, and expose a `drain_call_records() -> list[HttpCallRecord]` method. The orchestrator calls this after node execution and attaches records to `execution_metadata["http_calls"]`. This matches the existing `tool_calls` and `memory_interactions` accumulation pattern in `NodeAuditRecord`.

3. **Per-Node vs Per-Tool Configuration**
   - What we know: HTTP-01 says "configurable per-node or per-tool". The client is instantiated at bootstrap time (per-tenant).
   - What's unclear: How to scope configuration per-node and per-tool when the client instance is shared.
   - Recommendation: The `EndpointConfig` is passed per-request by the caller (agent tool, execution unit). The client uses global defaults from `HttpClientSettings` and merges with per-request `EndpointConfig` overrides. This gives per-tool/per-node configurability without multiple client instances.

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | Auth header resolution via SecretProvider -- never hardcode tokens |
| V3 Session Management | no | HTTP client is stateless |
| V4 Access Control | yes | Capability gating via NETWORK_READ/NETWORK_WRITE/EXTERNAL_API_CALL |
| V5 Input Validation | yes | URL validation before request; EndpointConfig validated via Pydantic |
| V6 Cryptography | no | TLS handled by httpx/httpcore transparently |

### Known Threat Patterns for HTTP Client

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Secret leakage in logs | Information Disclosure | Redact query params from URLs; never log auth headers; use SecretRedactor for audit |
| SSRF (Server-Side Request Forgery) | Spoofing | Capability gating ensures only authorized nodes can make external calls; URL validation |
| Retry amplification DDoS | Denial of Service | Circuit breaker stops retries after threshold; jitter prevents synchronized retries |
| Connection exhaustion | Denial of Service | httpx.Limits caps connections; per-endpoint rate limiting caps request rate |
| Token/API key exposure | Information Disclosure | SecretProvider resolves at runtime; secrets never stored in config models |

## Sources

### Primary (HIGH confidence)
- httpx 0.28.1 installed runtime -- inspected `httpx.AsyncClient.__init__`, `httpx.Limits`, `httpx.Timeout`, `httpx.MockTransport` APIs directly [VERIFIED: Python runtime inspection]
- `src/zeroth/core/policy/models.py` -- Capability enum with NETWORK_READ, NETWORK_WRITE, EXTERNAL_API_CALL [VERIFIED: file read]
- `src/zeroth/core/policy/guard.py` -- PolicyGuard.evaluate() signature and behavior [VERIFIED: file read]
- `src/zeroth/core/webhooks/delivery.py` -- Existing retry+backoff pattern with httpx [VERIFIED: file read]
- `src/zeroth/core/config/settings.py` -- ZerothSettings sub-model pattern [VERIFIED: file read]
- `src/zeroth/core/service/bootstrap.py` -- ServiceBootstrap wiring pattern [VERIFIED: file read]
- `src/zeroth/core/secrets/provider.py` -- SecretProvider Protocol + SecretResolver [VERIFIED: file read]
- `src/zeroth/core/audit/models.py` -- NodeAuditRecord with execution_metadata [VERIFIED: file read]
- `src/zeroth/core/artifacts/` -- Recent Phase 34 package structure pattern [VERIFIED: file read]
- `src/zeroth/core/guardrails/rate_limit.py` -- TokenBucketRateLimiter pattern [VERIFIED: file read]
- `src/zeroth/core/execution_units/sidecar_client.py` -- httpx.AsyncClient wrapping pattern + close() [VERIFIED: file read]

### Secondary (MEDIUM confidence)
- [httpx Transports documentation](https://www.python-httpx.org/advanced/transports/) -- Transport retries are connection-level only
- [httpx Async Support](https://www.python-httpx.org/async/) -- AsyncClient configuration patterns

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- httpx 0.28.1 already installed and used extensively; no new dependencies
- Architecture: HIGH -- all integration points verified in codebase; patterns extracted from existing modules
- Pitfalls: HIGH -- PolicyGuard API mismatch verified; circuit breaker race condition is a well-known pattern concern

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- httpx and codebase patterns unlikely to change rapidly)
