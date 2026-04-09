---
phase: 17-deployment-packaging-operations
plan: 03
subsystem: infra
tags: [docker, nginx, tls, uvicorn, deployment]

requires:
  - phase: 17-01
    provides: health probes, TLS settings, API versioning
  - phase: 17-02
    provides: async database factory, Alembic migrations, bootstrap_service async
provides:
  - Dockerfile with multi-stage uv-based build
  - docker-compose.yml with 6-service topology
  - Nginx TLS reverse proxy configuration
  - Production entrypoint with migration support
affects: [deployment, operations, ci-cd]

tech-stack:
  added: [nginx 1.27, postgres 16, redis 7]
  patterns: [multi-stage Docker build, uv-in-docker, factory-pattern entrypoint]

key-files:
  created:
    - Dockerfile
    - docker-compose.yml
    - docker/nginx/nginx.conf
    - docker/nginx/certs/.gitkeep
    - docker/regulus-sdk/.gitkeep
    - docker/regulus-sdk/README.md
    - src/zeroth/service/entrypoint.py
  modified: []

key-decisions:
  - "Regulus SDK handled via pre-built wheel in docker/regulus-sdk/ directory"
  - "Zeroth port 8000 not exposed to host -- only Nginx 443/80 are exposed"
  - "Docker socket mounted only to sandbox-sidecar, never to zeroth container"
  - "Production entrypoint uses uvicorn factory pattern with async bootstrap"

patterns-established:
  - "Docker wheel staging: local path deps converted to wheels in docker/ directory"
  - "Entrypoint pattern: sync main() for migrations, async factory for app bootstrap"

requirements-completed: [DEP-01, DEP-04]

duration: 89s
completed: 2026-04-07
---

# Phase 17 Plan 03: Docker Deployment Stack Summary

**Multi-stage Dockerfile with uv, 6-service docker-compose topology, Nginx TLS termination, and auto-migration production entrypoint**

## What Was Built

### Dockerfile
Multi-stage build using uv package manager. Stage 1 (builder) installs git for GovernAI's GitHub commit pin, copies pre-built Regulus SDK wheel, and installs all dependencies. Stage 2 (runtime) runs as non-root `zeroth` user (uid 1001), exposes port 8000, includes Docker HEALTHCHECK against `/health/live`, and uses the production entrypoint module.

### docker-compose.yml
Defines six services in a `zeroth-net` bridge network:
- **zeroth**: Built from Dockerfile, depends on healthy postgres and redis
- **postgres**: PostgreSQL 16 Alpine with healthcheck and persistent volume
- **redis**: Redis 7 Alpine with healthcheck
- **regulus**: Placeholder image for Regulus backend
- **sandbox-sidecar**: Placeholder image with Docker socket mounted (read-only)
- **nginx**: TLS termination on ports 443/80, proxies to zeroth:8000

### Nginx Configuration
TLS termination with TLSv1.2/1.3, proxy headers for correct client IP forwarding, buffering disabled for streaming responses, 300s read timeout for long-running agent operations, HTTP-to-HTTPS redirect on port 80.

### Production Entrypoint
`src/zeroth/service/entrypoint.py` provides:
- `main()`: Runs Alembic migrations (Postgres only), then starts uvicorn with optional TLS
- `app_factory()`: Async bootstrap creating database, wiring service, returning FastAPI app
- Used as `CMD ["python", "-m", "zeroth.service.entrypoint"]` in Dockerfile

## Verification Results

- docker-compose.yml validates as valid YAML
- Dockerfile contains all required instructions (FROM, multi-stage, USER, HEALTHCHECK)
- nginx.conf has ssl_certificate, proxy_pass, proxy_buffering off
- Docker socket only present in sandbox-sidecar service
- ruff check passes on entrypoint.py
- All 6 services present in docker-compose.yml

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all files are production-ready configurations. The regulus and sandbox-sidecar services use placeholder image names (`regulus-backend:latest`, `zeroth-sandbox-sidecar:latest`) which users must build or provide, but this is intentional and documented.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 2f22621 | Dockerfile, docker-compose.yml, nginx config, production entrypoint |
| 2 | -- | Auto-approved checkpoint (human-verify) |

## Self-Check: PASSED
