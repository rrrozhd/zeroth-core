# Zeroth Codebase Stack

## Overview

Zeroth is currently a Python 3.12 backend/service codebase. The repo is organized as a Python package under `src/zeroth` with pytest-based verification in `tests/`. There is no checked-in frontend application, package manager lockfile for JavaScript, or Vite/Vue/React workspace yet.

## Languages And Runtime

- Python 3.12+ is required by [`pyproject.toml`](`pyproject.toml`)
- The package name is `zeroth`
- Packaging uses Hatchling via [`pyproject.toml`](`pyproject.toml`)
- Local development uses `uv` with lockfile [`uv.lock`](`uv.lock`)

## Primary Frameworks And Libraries

- FastAPI powers the HTTP service surface in [`src/zeroth/service/app.py`](`src/zeroth/service/app.py`)
- Pydantic v2 is used for API models, domain models, and validation throughout `src/zeroth/*`
- PyJWT with crypto extras is used for bearer auth support in [`src/zeroth/service/auth.py`](`src/zeroth/service/auth.py`)
- `httpx` is present as a dependency for HTTP-facing integrations/tests
- `redis` is present as a dependency; there is also Redis-related storage code in [`src/zeroth/storage/redis.py`](`src/zeroth/storage/redis.py`)
- `uvicorn` is present for serving the FastAPI app
- `cryptography` is used indirectly through Fernet encryption in [`src/zeroth/storage/sqlite.py`](`src/zeroth/storage/sqlite.py`)

## Local Path Dependency

- The project depends on a local editable-style path for GovernAI:
  - `governai @ file:///Users/dondoe/coding/governai`
- This is defined in [`pyproject.toml`](`pyproject.toml`)
- This makes the repo environment-sensitive and can block reproducible setup on another machine unless that path exists or the dependency is replaced

## Tooling

- Testing: pytest, configured in [`pyproject.toml`](`pyproject.toml`)
- Async test support: `pytest-asyncio`
- Linting/format import ordering: Ruff
- No separate mypy, pre-commit, or frontend build tool configuration was found in repo root during mapping

## Persistence And Storage Technologies

- SQLite is the primary persisted storage backend through [`src/zeroth/storage/sqlite.py`](`src/zeroth/storage/sqlite.py`)
- JSON storage helpers exist in [`src/zeroth/storage/json.py`](`src/zeroth/storage/json.py`)
- Redis support exists in [`src/zeroth/storage/redis.py`](`src/zeroth/storage/redis.py`) and Phase 9 mentions Redis-related durable control behavior
- SQLite wrapper enables WAL mode, foreign keys, and schema version tracking

## Security And Identity Stack

- Static API key auth and bearer token verification are implemented in [`src/zeroth/service/auth.py`](`src/zeroth/service/auth.py`)
- Identity/role models live in [`src/zeroth/identity/models.py`](`src/zeroth/identity/models.py`)
- Service authorization gates live in [`src/zeroth/service/authorization.py`](`src/zeroth/service/authorization.py`)
- Secret resolution/redaction logic lives under `src/zeroth/secrets/`

## Runtime And Domain Packages

The package layout shows a strongly modular backend:

- `graph` — graph models, validation, serialization, storage
- `contracts` — contract registry and errors
- `mappings` — payload mapping execution and validation
- `runs` — run persistence and lifecycle state
- `agent_runtime` — agent invocation, prompts, tools, thread store
- `execution_units` — executable-unit manifests, adapters, runner, sandbox
- `conditions` — branching and condition evaluation
- `memory` — memory connectors and registry
- `approvals` — human approval models, repo, service
- `audit` — audit records, evidence, verifier, timeline
- `deployments` — deployment snapshots and provenance
- `dispatch` — leases and durable workers
- `guardrails` — quotas, dead letters, rate limiting
- `observability` — metrics, correlation, queue gauges
- `service` — FastAPI-facing HTTP APIs

## Missing Or Not Yet Present

- No frontend app directory such as `apps/studio`, `web/`, or `frontend/`
- No TypeScript config, Vite config, or Node package manifest
- No monorepo package management or workspace tooling
- No Studio-specific backend package yet such as `src/zeroth/studio/`

## Practical Summary

This is a mature Python backend with strong domain decomposition and test coverage, but it is still backend-only in repository shape. Any Studio UI phase will be additive: introducing a frontend stack and likely a new backend package for authoring/gateway concerns rather than extending an existing web client.
