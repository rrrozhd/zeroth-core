# Phase 11: Config & Postgres Storage - Context

**Gathered:** 2026-04-06
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 11 delivers a unified configuration system (YAML primary + env var overrides) using pydantic-settings and a production-grade Postgres storage backend behind a new async abstract Database interface. All existing repositories are rewritten to async, with Alembic managing migrations for both SQLite and Postgres. The SQLite backend is retained for dev/test.

</domain>

<decisions>
## Implementation Decisions

### Config Architecture
- **D-01:** Unified settings model using pydantic-settings with nested sub-models (DatabaseSettings, RedisSettings, AuthSettings, etc.) in a dedicated `src/zeroth/config/` package
- **D-02:** YAML is the primary config source with structured defaults; env vars override for deployment and secrets
- **D-03:** `.env` files loaded via `python-dotenv` / `load_dotenv()` as an additional override layer
- **D-04:** All env vars use the `ZEROTH_` prefix, consistent with existing `ZEROTH_REDIS_*` and `ZEROTH_SERVICE_*` conventions
- **D-05:** Existing `RedisConfig` absorbed into the unified settings model as a sub-model — one source of truth loaded once at startup
- **D-06:** Config loading priority: YAML defaults -> .env file -> environment variables (highest priority)

### Repository Abstraction
- **D-07:** Abstract async Database protocol/interface that both SQLiteDatabase and PostgresDatabase implement — repos take Database, not a specific implementation
- **D-08:** Full async rewrite of all 7 repositories (graph, runs, contracts, deployments, approvals, audit, threads) — `async def` methods throughout
- **D-09:** All callers (bootstrap, orchestrator, dispatch, service handlers) updated to `await` repository calls
- **D-10:** psycopg 3 async mode for the Postgres driver
- **D-11:** aiosqlite for the async SQLite path
- **D-12:** `ZEROTH_DB_BACKEND` env var selects `sqlite` or `postgres` at startup

### Migration Strategy
- **D-13:** Alembic for both SQLite and Postgres — single migration system replaces the custom `Migration` dataclass
- **D-14:** Raw SQL migrations (hand-written DDL) — no SQLAlchemy models or ORM
- **D-15:** Alembic migrations live at `src/zeroth/migrations/` inside the package

### Testing Approach
- **D-16:** testcontainers-python for Postgres testing — spins up real Postgres containers in pytest
- **D-17:** Key repository tests parametrized to run against both SQLite and Postgres backends via fixture parametrization

### Claude's Discretion
- Exact async Database protocol method signatures (transaction(), execute(), fetchone(), fetchall())
- Connection pooling strategy for Postgres (psycopg pool vs manual)
- Whether to use aiosqlite directly or wrap it to match the psycopg async interface
- Alembic env.py configuration details
- YAML config file naming and discovery (zeroth.yaml, zeroth.yml, etc.)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Storage Layer
- `src/zeroth/storage/sqlite.py` -- Current SQLiteDatabase class with transaction(), apply_migrations(), EncryptedField
- `src/zeroth/storage/redis.py` -- RedisConfig with from_env() pattern, to be absorbed into unified settings
- `src/zeroth/storage/__init__.py` -- Current storage package exports

### Repositories (all need async rewrite)
- `src/zeroth/graph/repository.py` -- GraphRepository
- `src/zeroth/runs/repository.py` -- RunRepository, ThreadRepository
- `src/zeroth/contracts/registry.py` -- ContractRegistry
- `src/zeroth/deployments/repository.py` -- SQLiteDeploymentRepository
- `src/zeroth/approvals/repository.py` -- ApprovalRepository
- `src/zeroth/audit/repository.py` -- AuditRepository
- `src/zeroth/agent_runtime/thread_store.py` -- RepositoryThreadResolver, RepositoryThreadStateStore

### Callers (need await updates)
- `src/zeroth/service/bootstrap.py` -- ServiceBootstrap factory wires all repos
- `src/zeroth/orchestrator/runtime.py` -- RuntimeOrchestrator calls repos in _drive() loop
- `src/zeroth/dispatch/worker.py` -- RunWorker calls repos for lease management
- `src/zeroth/dispatch/lease.py` -- LeaseManager uses SQLite directly
- `src/zeroth/guardrails/rate_limit.py` -- TokenBucketRateLimiter uses SQLite
- `src/zeroth/service/auth.py` -- Config loading from env vars

### Other Infrastructure
- `src/zeroth/service/app.py` -- FastAPI app factory, auth middleware, lifespan
- `.planning/REQUIREMENTS.md` -- CFG-01, CFG-02, CFG-03 requirements
- `.planning/ROADMAP.md` -- Phase 11 success criteria

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `SQLiteDatabase` class: transaction() context manager, apply_migrations(), EncryptedField — async equivalent needed
- `RedisConfig` pydantic model: from_env() pattern with validation — will be absorbed into unified settings
- `Migration` dataclass: versioned schema migrations per scope — to be replaced by Alembic

### Established Patterns
- All repositories follow: constructor takes `SQLiteDatabase`, calls `apply_migrations()`, provides CRUD via `self._database.transaction()`
- Raw SQL throughout — no ORM. This pattern continues with Alembic raw SQL migrations.
- Config via env vars with `ZEROTH_` prefix — maintained in new settings model
- `ServiceBootstrap` dataclass as composition root — will need async factory

### Integration Points
- `bootstrap_service()` in `src/zeroth/service/bootstrap.py` — main wiring point that constructs all repos with a Database instance
- `create_app()` in `src/zeroth/service/app.py` — lifespan hooks where database connections are established
- Every test fixture that creates a `SQLiteDatabase` — needs async equivalent or migration

</code_context>

<specifics>
## Specific Ideas

- YAML config as primary with env var overrides is inspired by production platform conventions (similar to Kubernetes, Airflow)
- Full async is chosen to align with FastAPI's async-first nature — avoids threadpool overhead for DB calls
- psycopg 3 chosen for its native async support and modern Python API
- testcontainers gives each test run an isolated real Postgres instance without CI infrastructure changes

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 11-config-postgres-storage*
*Context gathered: 2026-04-06*
