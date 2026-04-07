# Phase 17: Deployment Packaging & Operations - Research

**Researched:** 2026-04-07
**Domain:** Docker containerization, API versioning, health probes, TLS termination
**Confidence:** HIGH

## Summary

Phase 17 packages the Zeroth platform into a production-ready Docker Compose stack with five services (Zeroth app, Postgres, Redis, Regulus backend, sandbox sidecar), adds versioned API routes under `/v1/`, implements readiness/liveness health probes, auto-generates OpenAPI documentation, and supports TLS via Nginx reverse proxy.

The codebase is well-prepared for this phase. The FastAPI app factory (`service/app.py`) already has a clean `create_app()` pattern with closure-based route registration that can be refactored to use `APIRouter` for versioning. The `ZerothSettings` pydantic-settings model is extensible for TLS configuration. All infrastructure dependencies (Postgres via psycopg3 + psycopg_pool, Redis via redis-py, ARQ) are already wired and tested.

The primary challenge is the Regulus SDK (`econ-instrumentation-sdk`) which uses a local file path in `pyproject.toml`. The Dockerfile must pre-build the SDK wheel and COPY it into the build context, or use a wheelhouse volume mount.

**Primary recommendation:** Use a multi-stage Dockerfile with `uv` for fast dependency resolution, mount Regulus SDK as a pre-built wheel, and keep the API versioning simple with a single `APIRouter(prefix="/v1")` that all existing route registration functions mount onto.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Single multi-stage Dockerfile for the Zeroth application image. Companion services use upstream images in docker-compose.
- **D-02:** GovernAI installed from GitHub commit pin in Dockerfile (same `git+https://...@7452de4` as pyproject.toml).
- **D-03:** Regulus SDK resolved via a local wheelhouse volume or COPY from adjacent build context during Docker build.
- **D-04:** Docker-compose brings up all services with a single `docker compose up` -- no manual setup steps.
- **D-05:** Sandbox sidecar runs alongside API container in docker-compose, with Docker socket mounted only to the sidecar.
- **D-06:** All existing route registrations mounted under APIRouter with `prefix="/v1/"`. Existing unversioned paths remain active as aliases.
- **D-07:** Health endpoints (`/health`, `/health/ready`, `/health/live`) remain at root level, not versioned.
- **D-08:** Authentication middleware applies to both versioned and unversioned routes identically.
- **D-09:** FastAPI's native OpenAPI at `/openapi.json` documents all routes. No additional tooling.
- **D-10:** `/health/ready` checks: database SELECT 1, Redis PING, optional Regulus reachability.
- **D-11:** `/health/live` returns 200 if process is running -- no dependency checks.
- **D-12:** Existing `/health` endpoint unchanged for backward compatibility.
- **D-13:** Readiness response includes per-dependency status with ok/degraded/unhealthy. Regulus is optional.
- **D-14:** Primary TLS: Nginx reverse proxy in docker-compose handles TLS termination.
- **D-15:** Secondary TLS: optional `ZEROTH_TLS_CERTFILE` and `ZEROTH_TLS_KEYFILE` settings for direct uvicorn SSL.
- **D-16:** Docker-compose exposes port 443 (Nginx). Port 8000 (uvicorn) is internal.

### Claude's Discretion
- Multi-stage Dockerfile layer structure (build vs runtime stages, caching strategy)
- Docker-compose service names and network topology
- Exact APIRouter wiring approach (single router vs per-module sub-routers under v1)
- Health check timeout values and retry logic
- Nginx configuration details (proxy_pass, headers, buffer sizes)
- OpenAPI metadata (title, description, version string, contact info)
- Whether to add a `docker-compose.dev.yml` override for development

### Deferred Ideas (OUT OF SCOPE)
None.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DEP-01 | Dockerfile and docker-compose for Zeroth, Postgres, Redis, and Regulus backend | Multi-stage Dockerfile with uv, docker-compose with 5 services. Regulus SDK wheel strategy documented. |
| DEP-02 | API routes prefixed with /v1/ with version negotiation headers | FastAPI APIRouter with prefix="/v1", dual-mount pattern for backward compatibility |
| DEP-03 | OpenAPI spec auto-generated from FastAPI route definitions | FastAPI native OpenAPI generation at /openapi.json -- zero additional tooling |
| DEP-04 | TLS/HTTPS support via reverse proxy or uvicorn SSL configuration | Nginx container for TLS termination + optional uvicorn --ssl-certfile/--ssl-keyfile |
| OPS-03 | Readiness and liveness health probes with dependency checks | /health/ready with DB, Redis, Regulus checks; /health/live process-only |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| FastAPI | >=0.115 (already in pyproject.toml) | Web framework, OpenAPI generation | Already the project framework; native OpenAPI support |
| uvicorn | >=0.30 (already in pyproject.toml) | ASGI server | Already the project server; supports --ssl-certfile natively |
| uv | latest (build-time only) | Python package manager in Dockerfile | 10-100x faster than pip for Docker builds; deterministic lockfile |
| Docker / Docker Compose | v2 (compose v2 syntax) | Container orchestration | Standard for multi-service deployment |
| Nginx | 1.27-alpine (Docker image) | TLS termination, reverse proxy | Lightweight, proven, standard for HTTPS termination |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| psycopg3 / psycopg_pool | >=3.3 (already installed) | Postgres health check via SELECT 1 | Readiness probe DB check |
| redis | >=5.0 (already installed) | Redis health check via PING | Readiness probe Redis check |
| httpx | >=0.27 (already installed) | Regulus backend reachability check | Readiness probe Regulus check |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Nginx for TLS | Traefik | Traefik has auto-cert but heavier config; Nginx simpler for static certs |
| uv in Dockerfile | pip | pip is 10-100x slower; uv uses the existing uv.lock for reproducibility |
| Single v1 APIRouter | Per-module sub-routers | Per-module adds complexity; single router is simpler and sufficient |

## Architecture Patterns

### Recommended Project Structure
```
Dockerfile                    # Multi-stage build for zeroth app
docker-compose.yml            # All 5 services
docker/
  nginx/
    nginx.conf               # Reverse proxy + TLS config
    certs/                    # Bind-mount directory for TLS certs
  regulus-sdk/
    *.whl                     # Pre-built Regulus SDK wheel
src/zeroth/
  service/
    app.py                    # Modified: v1 router + health probes
    health.py                 # NEW: readiness/liveness probe logic
  config/
    settings.py               # Extended: TLS settings
```

### Pattern 1: API Versioning via Single APIRouter

**What:** Create a single `APIRouter(prefix="/v1")` and refactor all `register_*_routes()` functions to accept `FastAPI | APIRouter` instead of just `FastAPI`. Mount the router on the app, then also register routes directly on the app for backward compatibility.

**When to use:** When all routes share the same version and you need backward-compatible aliases.

**Implementation approach:**

The existing pattern is:
```python
def register_run_routes(app: FastAPI) -> None:
    @app.post("/runs", ...)
    async def create_run(...): ...
```

The refactored pattern:
```python
from fastapi import APIRouter, FastAPI

def register_run_routes(parent: FastAPI | APIRouter) -> None:
    @parent.post("/runs", ...)
    async def create_run(...): ...
```

Then in `create_app()`:
```python
v1_router = APIRouter(prefix="/v1", tags=["v1"])

# Register on v1 router (primary)
register_run_routes(v1_router)
register_approval_routes(v1_router)
# ... all route modules

# Mount v1 router
app.include_router(v1_router)

# Also register directly on app for backward compatibility (aliases)
register_run_routes(app)
register_approval_routes(app)
# ... all route modules
```

This works because FastAPI's `APIRouter` supports the same `@router.get()`, `@router.post()` decorator API as `FastAPI` itself. The type union `FastAPI | APIRouter` is safe because both inherit from Starlette's `Router`.

**Important:** The duplicate registration means each endpoint handler function is defined twice. FastAPI handles this fine -- routes are matched by path, and the first match wins. The `/v1/` prefixed routes appear first in the OpenAPI spec since `include_router` is called before the direct registrations.

**Alternative (simpler, recommended):** Instead of calling register functions twice, create the routes on the v1 router only, then loop through the v1 router's routes and add them to the app directly:

```python
v1_router = APIRouter(prefix="/v1")
register_run_routes(v1_router)
# ... all modules

app.include_router(v1_router)

# Add unversioned aliases
for route in v1_router.routes:
    app.routes.append(route)
```

This avoids duplicate function definitions and keeps a single source of truth. However, the route objects share state, which is fine for read-only route definitions.

**Recommended approach:** The simplest and cleanest is to change `register_*_routes` to accept `APIRouter`, create one v1 router, register everything on it, include it on the app, and then separately call the same registration functions on `app` for backward compat. The decorator approach is idiomatic FastAPI and each closure captures its own scope correctly.

### Pattern 2: Multi-Stage Dockerfile with uv

**What:** Two-stage Docker build: builder installs dependencies via `uv sync`, runtime copies only the `.venv` and source.

**When to use:** Always for Python production images with uv.

**Key structure:**
```dockerfile
# Stage 1: Builder
FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy

WORKDIR /app

# Install dependencies first (cached layer)
COPY pyproject.toml uv.lock ./
# Copy pre-built Regulus SDK wheel
COPY docker/regulus-sdk/ /wheels/
RUN uv sync --locked --no-install-project --no-dev

# Copy source and install project
COPY src/ src/
RUN uv sync --locked --no-dev

# Stage 2: Runtime
FROM python:3.12-slim
WORKDIR /app

# Create non-root user
RUN useradd --create-home --uid 1001 zeroth
USER zeroth

COPY --from=builder --chown=zeroth:zeroth /app/.venv /app/.venv
COPY --from=builder --chown=zeroth:zeroth /app/src /app/src
COPY alembic/ alembic/
COPY alembic.ini .

ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000

CMD ["uvicorn", "zeroth.service.bootstrap:create_app_from_settings", "--host", "0.0.0.0", "--port", "8000", "--proxy-headers"]
```

**Critical: Regulus SDK handling.** The `pyproject.toml` references `econ-instrumentation-sdk @ file:///Users/dondoe/coding/regulus/sdk/python` which will not resolve inside Docker. Strategy:

1. Pre-build the wheel: `cd ../regulus/sdk/python && python -m build --wheel --outdir ../../../zeroth/docker/regulus-sdk/`
2. Modify pyproject.toml for Docker: override the Regulus dependency to point at the local wheel file during build. Or better: use `uv pip install /wheels/*.whl` as a separate step after `uv sync` (with Regulus removed from pyproject.toml's lock resolution).

**Recommended:** The cleanest approach is to build the Regulus wheel before `docker build`, COPY it in, and use a `--find-links /wheels/` override or install it explicitly in the Dockerfile. The pyproject.toml should keep its local path for development; the Dockerfile handles the override.

### Pattern 3: Health Probe Implementation

**What:** Separate readiness and liveness endpoints with structured responses.

**When to use:** Container orchestration (Docker Compose, Kubernetes).

**Structure:**
```python
# service/health.py
from pydantic import BaseModel, Field

class DependencyCheck(BaseModel):
    status: str  # "ok" | "unavailable" | "error"
    latency_ms: float | None = None
    error: str | None = None

class ReadinessResponse(BaseModel):
    status: str  # "ok" | "degraded" | "unhealthy"
    checks: dict[str, DependencyCheck]

class LivenessResponse(BaseModel):
    status: str = "ok"
```

**Health check logic:**
- **Database:** `SELECT 1` via AsyncDatabase.transaction() with a timeout
- **Redis:** `redis.ping()` with a timeout
- **Regulus:** HTTP GET to Regulus base URL health endpoint with httpx timeout
- Regulus failure = "degraded" (optional dependency), DB/Redis failure = "unhealthy"

### Pattern 4: Docker Compose Service Topology

**What:** Five services in a single compose file with internal networking.

```yaml
services:
  zeroth:
    build: .
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    environment:
      ZEROTH_DATABASE__BACKEND: postgres
      ZEROTH_DATABASE__POSTGRES_DSN: postgresql://zeroth:zeroth@postgres:5432/zeroth
      ZEROTH_REDIS__HOST: redis
      ZEROTH_SANDBOX__BACKEND: sidecar
      ZEROTH_SANDBOX__SIDECAR_URL: http://sandbox-sidecar:8001
    networks: [zeroth-net]

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: zeroth
      POSTGRES_PASSWORD: zeroth
      POSTGRES_DB: zeroth
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zeroth"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [zeroth-net]

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [zeroth-net]

  regulus:
    image: regulus-backend:latest  # or build from adjacent context
    networks: [zeroth-net]

  sandbox-sidecar:
    image: zeroth-sandbox-sidecar:latest  # or build from sidecar Dockerfile
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [zeroth-net]

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./docker/nginx/certs:/etc/nginx/certs:ro
    depends_on: [zeroth]
    networks: [zeroth-net]

volumes:
  pgdata:

networks:
  zeroth-net:
```

### Anti-Patterns to Avoid
- **Mounting Docker socket to the API container:** Security violation per SBX-02. Only the sandbox sidecar gets socket access.
- **Installing dev dependencies in production image:** Use `--no-dev` with uv sync.
- **Hardcoding connection strings:** All service URLs must come from environment variables, matching the existing `ZEROTH_` prefix convention.
- **Running as root in container:** Use a non-root user in the runtime stage.
- **Skipping --proxy-headers on uvicorn:** When behind Nginx, uvicorn must trust forwarded headers for correct client IP and scheme detection.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| OpenAPI spec generation | Custom schema introspection | FastAPI native `/openapi.json` | FastAPI generates correct OpenAPI 3.1 from route type hints automatically |
| TLS termination | Direct uvicorn SSL in production | Nginx reverse proxy | Nginx handles cert rotation, HTTP/2, connection pooling; uvicorn SSL is for simple/dev cases only |
| Docker health checks | Custom polling scripts | Docker Compose `healthcheck` + `depends_on: condition` | Native orchestration support, no custom wiring |
| API versioning library | fastapi-versionizer or custom middleware | FastAPI `APIRouter(prefix="/v1")` | Built-in, zero dependencies, well-documented |

## Common Pitfalls

### Pitfall 1: Regulus SDK Local Path Breaks Docker Build
**What goes wrong:** `uv sync` fails because `file:///Users/dondoe/coding/regulus/sdk/python` does not exist in the Docker build context.
**Why it happens:** pyproject.toml has a local filesystem path dependency.
**How to avoid:** Pre-build the Regulus SDK as a wheel, COPY it into the Docker build context, and override the dependency resolution during Docker build. Use a separate `pyproject.docker.toml` or a build-time `--find-links` flag.
**Warning signs:** `uv sync` error during `docker build` mentioning file not found.

### Pitfall 2: GovernAI Git Dependency Requires Git in Builder
**What goes wrong:** `uv sync` fails because the builder image does not have `git` installed, and GovernAI is pinned to a GitHub commit.
**Why it happens:** `python:3.12-slim` does not include git by default.
**How to avoid:** Add `RUN apt-get update && apt-get install -y --no-install-recommends git && rm -rf /var/lib/apt/lists/*` in the builder stage before `uv sync`.
**Warning signs:** `uv sync` error mentioning git command not found.

### Pitfall 3: Duplicate Route Names in OpenAPI
**What goes wrong:** When registering the same endpoint function on both `/v1/runs` and `/runs`, FastAPI may generate duplicate operation IDs in the OpenAPI spec, causing schema validation warnings.
**Why it happens:** FastAPI auto-generates operation IDs from function names. Two routes with the same handler function get the same operation ID.
**How to avoid:** Either use `include_in_schema=False` for the unversioned aliases, or explicitly set unique `operation_id` parameters on the alias routes.
**Warning signs:** OpenAPI spec validation warnings about duplicate operationId.

### Pitfall 4: Alembic Migrations Need Database at Startup
**What goes wrong:** The Zeroth container starts before Postgres is ready, and Alembic migrations fail.
**Why it happens:** Docker Compose `depends_on` only waits for container start, not database readiness, unless `condition: service_healthy` is used.
**How to avoid:** Use `depends_on` with `condition: service_healthy` and configure Postgres health checks in docker-compose.yml.
**Warning signs:** Connection refused errors in container logs during startup.

### Pitfall 5: Nginx Buffering Large Responses
**What goes wrong:** Nginx buffers entire response before sending to client, causing memory issues or timeouts for large audit/run responses.
**Why it happens:** Default Nginx proxy buffering.
**How to avoid:** Set `proxy_buffering off;` or tune `proxy_buffer_size` and `proxy_buffers` in the Nginx config.
**Warning signs:** 502 errors or slow responses for large payloads.

### Pitfall 6: Authentication Middleware Blocking Health Probes
**What goes wrong:** The existing authentication middleware in `app.py` intercepts all requests, including health probe requests from Docker/load balancers.
**Why it happens:** The middleware runs for every request with no path exclusion.
**How to avoid:** Add path exclusions in the authentication middleware for `/health`, `/health/ready`, and `/health/live`. The existing middleware already runs on all routes; it needs a check like `if request.url.path.startswith("/health"): return await call_next(request)`.
**Warning signs:** Health probes return 401 Unauthorized.

## Code Examples

### Health Probe Endpoints (service/health.py)
```python
# Source: Informed by existing app.py pattern and D-10/D-11/D-13 decisions
import asyncio
from typing import Any

from pydantic import BaseModel, ConfigDict, Field

class DependencyStatus(BaseModel):
    model_config = ConfigDict(extra="forbid")
    status: str  # "ok" | "unavailable" | "error"
    latency_ms: float | None = None
    detail: str | None = None

class ReadinessResponse(BaseModel):
    model_config = ConfigDict(extra="forbid")
    status: str  # "ok" | "degraded" | "unhealthy"
    checks: dict[str, DependencyStatus] = Field(default_factory=dict)

class LivenessResponse(BaseModel):
    model_config = ConfigDict(extra="forbid")
    status: str = "ok"

async def check_database(db) -> DependencyStatus:
    """Check database connectivity via SELECT 1."""
    import time
    start = time.monotonic()
    try:
        async with db.transaction() as conn:
            await conn.fetch_one("SELECT 1")
        elapsed = (time.monotonic() - start) * 1000
        return DependencyStatus(status="ok", latency_ms=round(elapsed, 1))
    except Exception as exc:
        elapsed = (time.monotonic() - start) * 1000
        return DependencyStatus(
            status="error", latency_ms=round(elapsed, 1), detail=str(exc)
        )

async def check_redis(redis_client) -> DependencyStatus:
    """Check Redis connectivity via PING."""
    import time
    start = time.monotonic()
    try:
        await redis_client.ping()
        elapsed = (time.monotonic() - start) * 1000
        return DependencyStatus(status="ok", latency_ms=round(elapsed, 1))
    except Exception as exc:
        elapsed = (time.monotonic() - start) * 1000
        return DependencyStatus(
            status="error", latency_ms=round(elapsed, 1), detail=str(exc)
        )
```

### APIRouter Versioning Wiring (in create_app)
```python
# Source: FastAPI APIRouter docs + existing app.py pattern
from fastapi import APIRouter

v1_router = APIRouter(prefix="/v1", tags=["v1"])

# Register all routes on v1 router
register_run_routes(v1_router)
register_approval_routes(v1_router)
register_audit_routes(v1_router)
register_contract_routes(v1_router)
register_cost_routes(v1_router)
register_webhook_routes(v1_router)
register_admin_routes(v1_router)

# Mount versioned router
app.include_router(v1_router)

# Backward-compatible aliases (unversioned, excluded from OpenAPI)
compat_router = APIRouter(include_in_schema=False)
register_run_routes(compat_router)
register_approval_routes(compat_router)
# ... etc
app.include_router(compat_router)
```

### Nginx TLS Config
```nginx
# Source: Standard Nginx reverse proxy pattern for FastAPI
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://zeroth:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| pip install in Dockerfile | uv sync with lockfile | 2024-2025 | 10-100x faster builds, deterministic resolution |
| Gunicorn + UvicornWorker | Uvicorn with --workers | uvicorn 0.30+ (2024) | Simpler config, built-in multi-process |
| docker-compose (v1) | docker compose (v2, built-in) | 2023 | No separate install, better performance |
| Manual OpenAPI YAML | FastAPI auto-generation | Since FastAPI inception | Zero maintenance, always in sync with code |

## Open Questions

1. **Regulus Backend Docker Image**
   - What we know: Regulus backend is a companion service that needs to run in docker-compose. The SDK is at `/Users/dondoe/coding/regulus/sdk/python`.
   - What's unclear: Is there an existing Regulus backend Docker image, or does it need to be built from the adjacent repo? What image name/tag to use?
   - Recommendation: Use a placeholder image name (`regulus-backend:latest`) in docker-compose and document that users must build/provide it. Or add a build context pointing to the adjacent `regulus/` directory if it has a Dockerfile.

2. **Sandbox Sidecar Docker Image**
   - What we know: The sidecar must run as a separate container with Docker socket access (per SBX-02).
   - What's unclear: Is there an existing sidecar implementation with its own Dockerfile, or does Phase 17 need to create one?
   - Recommendation: If no sidecar image exists, create a minimal sidecar Dockerfile in `docker/sandbox-sidecar/` that runs the sidecar HTTP API on port 8001.

3. **Uvicorn Entrypoint**
   - What we know: The bootstrap module creates the app. Current pyproject.toml has no `[project.scripts]` entry for the server.
   - What's unclear: The exact import path for the app factory that uvicorn should call.
   - Recommendation: Create a thin entry module or use `uvicorn zeroth.service.bootstrap:create_app_from_settings --factory` pattern.

4. **Alembic in Container**
   - What we know: Alembic migrations run at startup via `run_migrations()`.
   - What's unclear: Whether the alembic config and migration directory are properly included in the Docker image.
   - Recommendation: Ensure `alembic.ini` and `alembic/` directory are COPYed into the runtime stage.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Docker | Container build & compose | Not in PATH | -- | Must be installed on target machine |
| uv | Dockerfile builder stage | Not in PATH (host) | -- | Copied from ghcr.io/astral-sh/uv:latest in Dockerfile |
| Python 3.12 | Application runtime | Via Docker image | 3.12-slim | -- |
| Nginx | TLS termination | Via Docker image | 1.27-alpine | -- |
| Postgres | Data storage | Via Docker image | 16-alpine | -- |
| Redis | Caching, ARQ queues | Via Docker image | 7-alpine | -- |

**Missing dependencies with no fallback:**
- Docker must be available on the deployment target to run `docker compose up`. This is expected and not a blocker for code implementation (Dockerfile and compose files can be written without Docker installed).

**Missing dependencies with fallback:**
- uv is not installed on the host but is only needed inside the Docker builder stage (copied from official image). No host installation required.

## Project Constraints (from CLAUDE.md)

- **Build/test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` main package, `tests/` for pytest tests
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Context efficiency:** Read only what is needed for the current task
- **Tracking:** `PROGRESS.md` is the single source of truth

## Sources

### Primary (HIGH confidence)
- Project source code: `src/zeroth/service/app.py`, `src/zeroth/config/settings.py`, `pyproject.toml` -- direct inspection
- [FastAPI APIRouter reference](https://fastapi.tiangolo.com/reference/apirouter/) -- official docs
- [uv Docker integration guide](https://docs.astral.sh/uv/guides/integration/docker/) -- official docs
- [Optimal Python uv Dockerfile](https://depot.dev/docs/container-builds/how-to-guides/optimal-dockerfiles/python-uv-dockerfile) -- verified pattern

### Secondary (MEDIUM confidence)
- [FastAPI Docker deployment](https://fastapi.tiangolo.com/deployment/docker/) -- official docs
- [Nginx reverse proxy for FastAPI](https://www.restack.io/p/fastapi-answer-nginx-reverse-proxy) -- community pattern, widely verified
- [FastAPI production deployment 2025 guide](https://blog.greeden.me/en/2025/09/02/the-definitive-guide-to-fastapi-production-deployment-with-dockeryour-one-stop-reference-for-uvicorn-gunicorn-nginx-https-health-checks-and-observability-2025-edition/) -- comprehensive community guide

### Tertiary (LOW confidence)
- Regulus backend Docker image availability -- unverified, needs investigation during implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already in pyproject.toml, Docker/Nginx are industry standard
- Architecture: HIGH - patterns derived from direct codebase inspection and official docs
- Pitfalls: HIGH - identified from actual codebase analysis (local path deps, auth middleware, git requirement)

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable domain, patterns unlikely to change)
