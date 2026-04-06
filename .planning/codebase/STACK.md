# Technology Stack

**Analysis Date:** 2026-04-05

## Languages

**Primary:**
- Python >=3.12 - Backend platform (`src/zeroth/`), all core logic, API surface, tests

**Secondary:**
- TypeScript ~5.8 - Studio mockup frontend (`apps/studio-mockups/`)
- Vue SFC (`.vue`) - Studio UI components (`apps/studio-mockups/src/`)

## Runtime

**Environment:**
- Python 3.12+ (specified via `requires-python = ">=3.12"` in `pyproject.toml`)
- Ruff target version: `py312`
- Node.js (for studio mockups build tooling; no `.nvmrc` present)

**Package Manager:**
- `uv` - Python dependency management and virtual environment
- Lockfile: `uv.lock` present (revision 3)
- `npm` - Frontend dependencies for studio mockups (`apps/studio-mockups/package.json`)

## Frameworks

**Core:**
- FastAPI >=0.115 - HTTP API framework (`src/zeroth/service/app.py`)
- Pydantic >=2.10 - Data validation and settings management (used throughout all modules)
- Uvicorn >=0.30 - ASGI server for FastAPI

**Frontend:**
- Vue 3 ^3.5.13 - Studio mockup UI (`apps/studio-mockups/`)
- Vite ^5.4.18 - Frontend build tool (`apps/studio-mockups/vite.config.ts`)
- vue-tsc ^2.2.8 - Vue TypeScript type checking
- @vitejs/plugin-vue ^5.2.4 - Vite Vue plugin

**Testing:**
- pytest >=8.0 - Test runner, config in `pyproject.toml` (`testpaths = ["tests"]`)
- pytest-asyncio >=0.25 - Async test support (`asyncio_mode = "auto"`)

**Build/Dev:**
- Hatchling - Python build backend (`pyproject.toml` build-system)
- Ruff >=0.11 - Linting and formatting (configured in `pyproject.toml`)

## Key Dependencies

**Critical:**
- `governai` (local path: `file:///Users/dondoe/coding/governai`) - Core governance engine providing `GovernedLLM`, `RunStore`, `InterruptStore`, `AuditEmitter`, `Tool`, `PythonTool`, `GovernedStepSpec`, `RunState`, `RunStatus`. This is the foundational runtime that Zeroth wraps and extends.
- `fastapi >=0.115` - HTTP service layer for deployed agents
- `pydantic >=2.10` - Data models, validation, serialization across all modules
- `redis >=5.0.0` - Distributed runtime state (runs, interrupts, audit events)

**Infrastructure:**
- `httpx >=0.27` - Async HTTP client for external API calls
- `PyJWT[crypto] >=2.10` - JWT bearer token verification (`src/zeroth/service/auth.py`)
- `cryptography` (transitive via PyJWT[crypto], also used directly) - Fernet symmetric encryption for `EncryptedField` in `src/zeroth/storage/sqlite.py`
- `uvicorn >=0.30` - Production ASGI server

## Configuration

**Environment Variables:**
- `ZEROTH_REDIS_*` - Redis connection settings (HOST, PORT, MODE, PASSWORD, SSL, etc.) loaded via `RedisConfig.from_env()` in `src/zeroth/storage/redis.py`
- `ZEROTH_SERVICE_API_KEYS_JSON` - Static API key credentials (JSON string)
- `ZEROTH_SERVICE_BEARER_JSON` - JWT/OIDC bearer token config (JSON string)
- No `.env` file present in repository root

**Build Configuration:**
- `pyproject.toml` - Python project config, dependencies, Ruff settings, pytest settings
- `apps/studio-mockups/vite.config.ts` - Frontend build configuration
- `apps/studio-mockups/tsconfig.json` - TypeScript configuration

**Ruff Settings (in `pyproject.toml`):**
- Line length: 100
- Target: Python 3.12
- Lint rules: E (pycodestyle), F (pyflakes), I (isort), N (naming), UP (pyupgrade), B (bugbear), SIM (simplify)

**Pytest Settings (in `pyproject.toml`):**
- Test paths: `tests/`
- Async mode: `auto` (all async tests run automatically without markers)

## Data Storage

**SQLite (Local Persistence):**
- Custom wrapper: `src/zeroth/storage/sqlite.py` (`SQLiteDatabase` class)
- WAL mode, foreign keys enabled, NORMAL synchronous
- Schema migration system with versioned `Migration` dataclass per scope
- Optional Fernet-based field encryption (`EncryptedField`)
- Used by all repository classes: deployments, graphs, contracts, runs, threads, approvals, audit

**Redis (Distributed Runtime State):**
- Config: `src/zeroth/storage/redis.py` (`RedisConfig`)
- Deployment modes: local, Docker, remote
- Three GovernAI-backed stores: `RedisRunStore`, `RedisInterruptStore`, `RedisAuditEmitter`
- Key prefix: `zeroth` (configurable)
- Optional TTL for runs and audit events

**JSON:**
- Helper module: `src/zeroth/storage/json.py`

## Platform Requirements

**Development:**
- Python 3.12+
- `uv` package manager
- Redis (local, Docker, or remote) for runtime state
- GovernAI sibling repo at `/Users/dondoe/coding/governai` (local path dependency)
- Node.js + npm (only for `apps/studio-mockups/` frontend)
- Docker (optional, for Redis container mode)

**Production:**
- Python 3.12+ with uvicorn ASGI server
- Redis instance for distributed runtime state
- SQLite file storage (local filesystem)
- JWT/OIDC provider or static API keys for authentication

**Build & Test Commands:**
```bash
uv sync                    # Install/update Python dependencies
uv run pytest -v           # Run all tests
uv run ruff check src/     # Lint
uv run ruff format src/    # Format
```

---

*Stack analysis: 2026-04-05*
