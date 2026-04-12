---
phase: 35-resilient-http-client
plan: 01
subsystem: http-client
tags: [http, resilience, circuit-breaker, retry, rate-limiting]
dependency_graph:
  requires: [httpx, zeroth.core.config, zeroth.core.policy, zeroth.core.secrets]
  provides: [zeroth.core.http]
  affects: [zeroth.core.config.settings]
tech_stack:
  added: []
  patterns: [circuit-breaker, token-bucket-rate-limiter, exponential-backoff-with-jitter, lazy-import-for-circular-avoidance]
key_files:
  created:
    - src/zeroth/core/http/__init__.py
    - src/zeroth/core/http/models.py
    - src/zeroth/core/http/errors.py
    - src/zeroth/core/http/circuit_breaker.py
    - src/zeroth/core/http/client.py
    - tests/http/__init__.py
    - tests/http/test_models.py
    - tests/http/test_circuit_breaker.py
    - tests/http/test_client.py
  modified:
    - src/zeroth/core/config/settings.py
    - src/zeroth/core/http/__init__.py
decisions:
  - Lazy import of ResilientHttpClient in __init__.py via __getattr__ to break circular import chain (settings -> http -> client -> policy -> graph -> storage -> settings)
  - Lazy import of Capability inside _check_capabilities method for same circular import reason
  - DELETE included in _IDEMPOTENT_METHODS for retry purposes (safe to retry since it is idempotent per HTTP spec)
metrics:
  duration_seconds: 614
  completed: "2026-04-12T22:55:19Z"
  tasks_completed: 2
  tasks_total: 2
  tests_added: 48
  files_created: 9
  files_modified: 1
---

# Phase 35 Plan 01: Resilient HTTP Client Core Package Summary

Resilient HTTP client wrapping httpx.AsyncClient with retry (exponential backoff + jitter), per-endpoint circuit breaking (CLOSED/OPEN/HALF_OPEN state machine), in-memory token-bucket rate limiting, capability gating, SecretProvider auth injection, and call-record auditing -- all with zero new dependencies.

## What Was Built

### Models (`src/zeroth/core/http/models.py`)
- `HttpClientSettings` -- global config with D-04 defaults (max_retries=3, backoff_base=0.5, circuit threshold=5, pool 100/20, timeout 30s, rate 10/burst 20), `extra="forbid"`
- `EndpointConfig` -- per-endpoint overrides with all-optional fields for max_retries, retryable_status_codes, timeout, secret_key, auth_type, auth_header_name, rate_limit_rate/burst
- `AuthType` -- StrEnum: bearer, api_key, custom_header
- `HttpCallRecord` -- audit record with redacted URL, method, status, latency, size, retry count, circuit breaker state, error
- `redact_url()` -- strips query params and fragments from URLs for safe audit logging

### Errors (`src/zeroth/core/http/errors.py`)
- `HttpClientError` (base), `CircuitOpenError`, `HttpRetryExhaustedError`, `HttpRateLimitError`

### Circuit Breaker & Rate Limiter (`src/zeroth/core/http/circuit_breaker.py`)
- `CircuitState` StrEnum (closed, open, half_open)
- `CircuitBreaker` -- async state machine with asyncio.Lock; dynamic OPEN->HALF_OPEN transition based on elapsed time
- `CircuitBreakerRegistry` -- per-endpoint breaker cache (keyed by host:port)
- `InMemoryTokenBucket` -- refill-on-acquire rate limiter with configurable rate and burst

### Client (`src/zeroth/core/http/client.py`)
- `ResilientHttpClient` -- full pipeline: rate limit -> capability check -> auth resolution -> circuit breaker -> retry loop with backoff -> audit record
- D-05 backoff formula: `backoff_base * 2^attempt + random(0, 0.1)`, capped at retry_max_delay
- Non-idempotent methods (POST/PUT/PATCH) do NOT retry on status codes by default; only retry when EndpointConfig explicitly sets retryable_status_codes
- Convenience methods: get, post, put, patch, delete
- `drain_call_records()` returns and clears accumulated HttpCallRecord list
- `aclose()` delegates to httpx.AsyncClient

### Settings Integration
- `HttpClientSettings` wired into `ZerothSettings` as `http_client` field in `src/zeroth/core/config/settings.py`

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Circular import chain: settings -> http -> client -> policy -> graph -> storage -> settings**
- **Found during:** Task 2
- **Issue:** Importing `Capability` from `zeroth.core.policy.models` at module level in `client.py`, and importing `ResilientHttpClient` from `client.py` in `__init__.py`, created a circular import chain that prevented the entire package from loading
- **Fix:** (a) Used `__getattr__` lazy-import pattern in `__init__.py` for `ResilientHttpClient`. (b) Moved `Capability` import inside `_check_capabilities()` method behind an early-return guard.
- **Files modified:** `src/zeroth/core/http/__init__.py`, `src/zeroth/core/http/client.py`
- **Commit:** 85de9be

**2. [Rule 1 - Bug] Retry exhaustion on retryable status did not raise**
- **Found during:** Task 2
- **Issue:** When the last retry attempt returned a retryable status code, the code fell through to return the response instead of raising `HttpRetryExhaustedError`, because the `attempt < max_retries` check was false on the last attempt
- **Fix:** Added an explicit check after the `continue` branch: if status is retryable but no more attempts remain, break out of the loop to the exhaustion handler
- **Files modified:** `src/zeroth/core/http/client.py`
- **Commit:** 85de9be

**3. [Rule 1 - Bug] Test used deprecated `asyncio.coroutine` (removed in Python 3.12)**
- **Found during:** Task 2
- **Issue:** `test_aclose` used `asyncio.coroutine(lambda: None)()` which no longer exists in Python 3.12
- **Fix:** Replaced with `unittest.mock.AsyncMock()`
- **Files modified:** `tests/http/test_client.py`
- **Commit:** 85de9be

## Commits

| Task | Commit | Message |
|------|--------|---------|
| 1 (RED) | 06fdaf7 | test(35-01): add failing tests for models, errors, circuit breaker, and rate limiter |
| 1 (GREEN) | ac78398 | feat(35-01): implement models, errors, circuit breaker, and rate limiter |
| 2 (RED) | 07d1da0 | test(35-01): add failing tests for ResilientHttpClient and settings integration |
| 2 (GREEN) | 85de9be | feat(35-01): implement ResilientHttpClient and wire HttpClientSettings into ZerothSettings |

## Test Results

48 tests added, all passing:
- `tests/http/test_models.py` -- 13 tests (settings defaults, extra=forbid, endpoint config, auth type, call record, redact_url)
- `tests/http/test_circuit_breaker.py` -- 14 tests (state transitions, registry, token bucket)
- `tests/http/test_client.py` -- 21 tests (basic requests, retry, backoff, circuit breaker, rate limiter, capability gating, auth injection, call records, aclose, settings integration)

All tests use `httpx.MockTransport` -- no network calls.

## Threat Mitigations Verified

| Threat ID | Mitigation | Status |
|-----------|------------|--------|
| T-35-01 | redact_url() strips query params; auth headers never in call records | Verified via test_drain_returns_records_and_clears |
| T-35-02 | Secrets resolved at call time via SecretProvider; never stored in config models | Verified via auth injection tests |
| T-35-03 | Circuit breaker + jitter backoff + rate limiter prevent retry storms | Verified via circuit breaker and rate limiter integration tests |
| T-35-04 | httpx.Limits caps connections; rate limiter caps request rate | Verified via settings defaults test |
| T-35-05 | Capability check before every request; missing cap raises HttpClientError | Verified via capability gating tests |

## Self-Check: PASSED
