# Phase 17: Deployment Packaging & Operations - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 17 packages Zeroth as a reproducible container image with docker-compose orchestration (Zeroth, Postgres, Redis, Regulus backend, sandbox sidecar), adds versioned API routes under `/v1/`, auto-generates OpenAPI documentation, implements readiness/liveness health probes with dependency checks, and supports TLS via reverse proxy or direct uvicorn SSL configuration.

</domain>

<decisions>
## Implementation Decisions

### Container Topology
- **D-01:** Single multi-stage Dockerfile for the Zeroth application image. Companion services (Postgres, Redis, Regulus backend, sandbox sidecar) use their own upstream images in docker-compose.
- **D-02:** GovernAI dependency installed from GitHub commit pin in Dockerfile (same `git+https://...@7452de4` as pyproject.toml). No special handling needed beyond pip/uv install.
- **D-03:** Regulus SDK (`econ-instrumentation-sdk`) resolved via a local wheelhouse volume or COPY from adjacent build context during Docker build, since it references a local file path in pyproject.toml. The Dockerfile must handle this path dependency.
- **D-04:** Docker-compose brings up all services with a single `docker compose up` — no manual setup steps. Environment variables configure connection strings between services.
- **D-05:** Sandbox sidecar container runs alongside the API container in docker-compose, with Docker socket mounted only to the sidecar (per Phase 14 SBX-02 decision).

### API Versioning Strategy
- **D-06:** All existing route registrations (run, approval, audit, contracts, cost, webhook, admin) mounted under an APIRouter with `prefix="/v1/"`. Existing unversioned paths remain active as aliases so no existing tests break.
- **D-07:** The `/health` and `/health/ready` and `/health/live` endpoints remain at the root level (not versioned) — standard practice for infrastructure probes and load balancer checks.
- **D-08:** The authentication middleware applies to both versioned and unversioned routes identically.
- **D-09:** FastAPI's auto-generated OpenAPI spec at `/openapi.json` documents all `/v1/` routes with correct schemas and authentication requirements. No additional OpenAPI tooling needed — FastAPI provides this natively.

### Health Probe Depth
- **D-10:** `/health/ready` (readiness probe) checks: database connection (SELECT 1), Redis PING, and optional Regulus backend reachability. Returns HTTP 200 with structured JSON only when all required dependencies are healthy. Returns structured error with per-dependency status on failure.
- **D-11:** `/health/live` (liveness probe) returns HTTP 200 if the process is running — no dependency checks. This prevents container restarts when a dependency is temporarily down.
- **D-12:** The existing `/health` endpoint (deployment metadata) remains unchanged for backward compatibility. New readiness/liveness endpoints are additive.
- **D-13:** Readiness probe response includes per-dependency status: `{"status": "ok"|"degraded"|"unhealthy", "checks": {"database": "ok", "redis": "ok", "regulus": "ok"|"unavailable"}}`. Regulus is optional — its unavailability does not make the probe fail, but is reported.

### TLS Approach
- **D-14:** Primary TLS strategy: reverse proxy (Nginx) in docker-compose handles TLS termination. Uvicorn runs plain HTTP behind it. The compose file includes an Nginx service with a bind-mounted certificate directory.
- **D-15:** Secondary TLS support: optional `ZEROTH_TLS_CERTFILE` and `ZEROTH_TLS_KEYFILE` settings allow direct uvicorn SSL for simple deployments without a reverse proxy. When set, uvicorn starts with `--ssl-certfile` and `--ssl-keyfile`.
- **D-16:** Docker-compose exposes port 443 (via Nginx) by default. Port 8000 (uvicorn) is internal to the Docker network.

### Claude's Discretion
- Multi-stage Dockerfile layer structure (build vs runtime stages, caching strategy)
- Docker-compose service names and network topology
- Exact APIRouter wiring approach (single router vs per-module sub-routers under v1)
- Health check timeout values and retry logic
- Nginx configuration details (proxy_pass, headers, buffer sizes)
- OpenAPI metadata (title, description, version string, contact info)
- Whether to add a `docker-compose.dev.yml` override for development

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Service Layer
- `src/zeroth/service/app.py` — FastAPI app factory, lifespan, auth middleware, existing `/health` endpoint, route registration
- `src/zeroth/service/bootstrap.py` — ServiceBootstrap wiring all dependencies
- `src/zeroth/service/run_api.py` — Run route registration pattern (closure-based)
- `src/zeroth/service/auth.py` — Authentication config and middleware

### Configuration
- `src/zeroth/config/settings.py` — ZerothSettings with DatabaseSettings, RedisSettings, RegulusSettings (extend for TLS settings)
- `pyproject.toml` — Dependencies including GovernAI git pin, Regulus local path, all infrastructure deps

### Infrastructure (from prior phases)
- `src/zeroth/dispatch/worker.py` — RunWorker for background task lifecycle in container
- `src/zeroth/dispatch/arq_wakeup.py` — ARQ wakeup module (Phase 16) requiring Redis in docker-compose
- `src/zeroth/storage/redis.py` — RedisConfig for connection management
- `src/zeroth/db/` — Async Database protocol, SQLite and Postgres implementations

### Sandbox Sidecar
- `src/zeroth/execution_units/sandbox.py` — Sandbox sidecar client (Phase 14) — sidecar must be in docker-compose

### Observability
- `src/zeroth/observability/metrics.py` — MetricsCollector with Prometheus-format output (expose via /metrics endpoint if needed)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **FastAPI app factory** (`service/app.py`): Already creates the app with lifespan, middleware, and route registration — extend with APIRouter for v1 prefix and new health endpoints
- **HealthResponse model** (`service/app.py`): Existing health response — keep as-is, add new ReadinessResponse and LivenessResponse models
- **ZerothSettings** (`config/settings.py`): Pydantic-settings model — extend with TLS settings
- **Route registration pattern**: `register_*_routes(app)` closures — wrap under APIRouter for versioning

### Established Patterns
- **Closure-based route registration**: All API modules use `register_*_routes(app: FastAPI)` — can be refactored to register on an APIRouter instead
- **Bootstrap via app.state**: Dependencies accessed via `request.app.state.bootstrap` — unchanged by versioning
- **Pydantic settings with env vars**: `ZEROTH_` prefix convention — extend for `ZEROTH_TLS_CERTFILE`, `ZEROTH_TLS_KEYFILE`
- **Background task lifecycle**: Lifespan context manager handles start/stop — Docker SIGTERM triggers existing graceful shutdown (Phase 16)

### Integration Points
- **Route registration in app.py**: Where v1 APIRouter would be mounted
- **create_app()**: Entry point for adding versioned router and health endpoints
- **pyproject.toml**: Must handle Regulus SDK local path for Docker build
- **uvicorn startup**: Currently in pyproject.toml scripts or direct invocation — needs Dockerfile CMD

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Key constraints:
- `docker compose up` must work with zero manual setup
- Existing tests must pass with unversioned route aliases
- Sandbox sidecar must never expose Docker socket to the API container

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 17-deployment-packaging-operations*
*Context gathered: 2026-04-07*
