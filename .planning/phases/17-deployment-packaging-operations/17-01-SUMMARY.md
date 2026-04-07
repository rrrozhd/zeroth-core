---
phase: 17-deployment-packaging-operations
plan: 01
subsystem: service/health, config/settings
tags: [health-probes, tls, readiness, liveness, auth-bypass]
dependency_graph:
  requires: [config-settings, async-database, bootstrap]
  provides: [health-probes, tls-settings, auth-bypass-health]
  affects: [service/app, service/bootstrap, auth-middleware]
tech_stack:
  added: [httpx-health-check, redis-asyncio-ping]
  patterns: [dependency-status-check, readiness-degraded-model, auth-path-exclusion]
key_files:
  created:
    - src/zeroth/service/health.py
  modified:
    - src/zeroth/config/settings.py
    - src/zeroth/service/app.py
    - src/zeroth/service/bootstrap.py
    - tests/test_health_probes.py
    - tests/service/test_auth_api.py
    - tests/service/test_bearer_auth.py
decisions:
  - "Health endpoints bypass auth middleware via path prefix check"
  - "Regulus unavailability produces 'degraded' not 'unhealthy' status"
  - "Redis connectivity checked via from_url + ping pattern"
  - "Database reference stored on ServiceBootstrap for health probe access"
metrics:
  duration: 457s
  completed: "2026-04-07T17:58:47Z"
  tasks: 1
  files: 7
---

# Phase 17 Plan 01: Health Probes and TLS Settings Summary

Readiness and liveness health probes with per-dependency status checking (database, Redis, Regulus) plus TLS certfile/keyfile configuration via environment variables.

## What Was Built

### Health Probe Module (`src/zeroth/service/health.py`)

- **DependencyStatus** model: status (ok/unavailable/error), latency_ms, detail
- **ReadinessResponse** model: overall status (ok/degraded/unhealthy) with per-dependency checks dict
- **LivenessResponse** model: always returns status="ok"
- **check_database()**: Executes SELECT 1 via AsyncDatabase.transaction(), measures latency
- **check_redis()**: Creates redis.asyncio client from URL, calls ping(), measures latency
- **check_regulus()**: HTTP GET to base_url/health via httpx.AsyncClient, returns unavailable if not configured
- **determine_readiness_status()**: DB/Redis failure = unhealthy, Regulus-only failure = degraded, all ok = ok
- **register_health_routes()**: Registers /health/ready and /health/live on FastAPI app

### TLS Settings (`src/zeroth/config/settings.py`)

- **TLSSettings** class with certfile and keyfile fields (both default to None)
- Added to ZerothSettings as `tls: TLSSettings` field
- Configurable via ZEROTH_TLS__CERTFILE and ZEROTH_TLS__KEYFILE environment variables

### Auth Middleware Bypass (`src/zeroth/service/app.py`)

- Health endpoints registered BEFORE auth middleware definition
- Auth middleware checks `request.url.path.startswith("/health")` and skips authentication
- Prevents 401 responses on load balancer health probes

### Bootstrap Wiring (`src/zeroth/service/bootstrap.py`)

- Added `database: AsyncDatabase | None = None` field to ServiceBootstrap dataclass
- Wired database reference in bootstrap_service() return statement

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated existing auth tests for health bypass behavior**
- **Found during:** Task 1 (GREEN phase)
- **Issue:** Existing tests in test_auth_api.py and test_bearer_auth.py expected /health to require authentication (401). Health auth bypass made these tests fail.
- **Fix:** Updated test_service_health_requires_authentication to test_service_health_bypasses_authentication (expects 200). Updated bearer auth rejection tests to use /runs endpoint instead of /health, and added test_health_bypasses_auth_even_with_bad_bearer_token.
- **Files modified:** tests/service/test_auth_api.py, tests/service/test_bearer_auth.py
- **Commit:** dd632bc

**2. [Rule 2 - Missing] Redis URL construction for health check**
- **Found during:** Task 1 implementation
- **Issue:** Plan referenced RedisSettings object but health check needed actual Redis URL for redis.asyncio.from_url()
- **Fix:** Health readiness endpoint constructs Redis URL from ZerothSettings redis fields (scheme, auth, host, port, db)
- **Files modified:** src/zeroth/service/health.py

## Decisions Made

1. **Health endpoints bypass auth via path prefix** -- All paths starting with /health skip the authenticate_request middleware. This covers /health, /health/ready, and /health/live.
2. **Regulus unavailability = degraded, not unhealthy** -- Regulus is optional; only DB and Redis failures cause unhealthy status.
3. **Redis check uses from_url pattern** -- Constructs Redis URL from settings rather than passing RedisSettings object, since redis.asyncio needs a URL string.
4. **Database stored on ServiceBootstrap** -- Added optional database field to ServiceBootstrap dataclass so health probes can access DB without going through repositories.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| RED | b19780f | Failing tests for health probes, TLS settings, auth bypass |
| GREEN | dd632bc | Implementation of health probes, TLS settings, auth bypass |

## Known Stubs

None -- all functionality is fully wired.

## Self-Check: PASSED
