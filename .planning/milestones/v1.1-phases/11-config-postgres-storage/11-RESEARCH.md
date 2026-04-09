# Phase 11: Config & Postgres Storage - Research

**Researched:** 2026-04-06
**Domain:** Configuration management (pydantic-settings), async database abstraction (SQLite + Postgres), schema migrations (Alembic)
**Confidence:** HIGH

## Summary

Phase 11 delivers two major subsystems: (1) a unified pydantic-settings configuration model with YAML primary source and env var overrides, and (2) a production-grade Postgres storage backend behind an async abstract Database interface that also supports the existing SQLite path. All seven repositories must be rewritten from synchronous to async, and Alembic replaces the custom Migration dataclass for schema management.

The key technical challenges are: normalizing SQL parameter placeholders (`?` in SQLite vs `%s` in psycopg3), handling SQLite-specific features (PRAGMAs, `executescript`, `row_factory`), writing cross-dialect DDL for Alembic migrations, and converting all callers (bootstrap, orchestrator, dispatch, guardrails) to async. pydantic-settings has built-in YAML support via `YamlConfigSettingsSource`, which simplifies the config layer significantly.

**Primary recommendation:** Build a thin async Database protocol with `transaction()`, `execute()`, `fetch_one()`, `fetch_all()` methods that normalize placeholders internally, so repositories remain dialect-agnostic with `?`-style parameters.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Unified settings model using pydantic-settings with nested sub-models (DatabaseSettings, RedisSettings, AuthSettings, etc.) in a dedicated `src/zeroth/config/` package
- **D-02:** YAML is the primary config source with structured defaults; env vars override for deployment and secrets
- **D-03:** `.env` files loaded via `python-dotenv` / `load_dotenv()` as an additional override layer
- **D-04:** All env vars use the `ZEROTH_` prefix, consistent with existing `ZEROTH_REDIS_*` and `ZEROTH_SERVICE_*` conventions
- **D-05:** Existing `RedisConfig` absorbed into the unified settings model as a sub-model -- one source of truth loaded once at startup
- **D-06:** Config loading priority: YAML defaults -> .env file -> environment variables (highest priority)
- **D-07:** Abstract async Database protocol/interface that both SQLiteDatabase and PostgresDatabase implement -- repos take Database, not a specific implementation
- **D-08:** Full async rewrite of all 7 repositories (graph, runs, contracts, deployments, approvals, audit, threads) -- `async def` methods throughout
- **D-09:** All callers (bootstrap, orchestrator, dispatch, service handlers) updated to `await` repository calls
- **D-10:** psycopg 3 async mode for the Postgres driver
- **D-11:** aiosqlite for the async SQLite path
- **D-12:** `ZEROTH_DB_BACKEND` env var selects `sqlite` or `postgres` at startup
- **D-13:** Alembic for both SQLite and Postgres -- single migration system replaces the custom `Migration` dataclass
- **D-14:** Raw SQL migrations (hand-written DDL) -- no SQLAlchemy models or ORM
- **D-15:** Alembic migrations live at `src/zeroth/migrations/` inside the package
- **D-16:** testcontainers-python for Postgres testing -- spins up real Postgres containers in pytest
- **D-17:** Key repository tests parametrized to run against both SQLite and Postgres backends via fixture parametrization

### Claude's Discretion
- Exact async Database protocol method signatures (transaction(), execute(), fetchone(), fetchall())
- Connection pooling strategy for Postgres (psycopg pool vs manual)
- Whether to use aiosqlite directly or wrap it to match the psycopg async interface
- Alembic env.py configuration details
- YAML config file naming and discovery (zeroth.yaml, zeroth.yml, etc.)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CFG-01 | Platform loads configuration from environment variables and .env files with startup validation | pydantic-settings with `YamlConfigSettingsSource`, `DotEnvSettingsSource`, and `EnvSettingsSource` provides built-in validation and multi-source loading |
| CFG-02 | Postgres storage backend available behind existing sync repository interface with Alembic migrations | Async Database protocol with psycopg3 `AsyncConnectionPool` + Alembic raw SQL migrations; note the requirement says "sync" but CONTEXT.md decision D-08 overrides to async |
| CFG-03 | Storage backend selectable via ZEROTH_DB_BACKEND flag (sqlite/postgres) at startup | Factory function in bootstrap reads `ZEROTH_DB_BACKEND` from unified settings and constructs either `AsyncSQLiteDatabase` or `AsyncPostgresDatabase` |
</phase_requirements>

## Conflict Note

STATE.md (accumulated decision) says "repos stay synchronous (psycopg2/psycopg3 sync, NOT asyncpg)". CONTEXT.md (Phase 11 discussion) explicitly overrides this with D-08 "Full async rewrite" and D-10 "psycopg 3 async mode". CONTEXT.md represents the latest user decision and takes precedence. The planner should follow CONTEXT.md.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pydantic-settings | >=2.13 | Unified configuration with env, .env, YAML sources | Official pydantic companion; built-in YAML support via YamlConfigSettingsSource |
| psycopg[binary] | >=3.3 | Postgres async driver | Native async, server-side binding, modern Python API; successor to psycopg2 |
| psycopg-pool | >=3.2 | Async connection pooling | Official psycopg companion; AsyncConnectionPool with health checks |
| aiosqlite | >=0.22 | Async SQLite driver | Asyncio bridge to stdlib sqlite3; widely used |
| alembic | >=1.18 | Schema migration management | Industry standard for Python; supports both SQLite and Postgres |
| PyYAML | >=6.0 | YAML parsing (pydantic-settings dependency) | Required by pydantic-settings YamlConfigSettingsSource |
| python-dotenv | >=1.0 | .env file loading | Standard .env loader; used by pydantic-settings DotEnvSettingsSource |

### Supporting (dev/test only)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| testcontainers[postgres] | >=4.14 | Postgres containers for testing | Integration tests that verify Postgres backend |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| psycopg3 async | asyncpg | asyncpg is faster but has incompatible API and no sync fallback; psycopg3 is the user's locked choice |
| aiosqlite | sqlite3 in thread pool | Thread pool adds overhead; aiosqlite provides cleaner async API |
| Alembic | yoyo-migrations | Alembic has broader ecosystem support; Alembic is the user's locked choice |

**Installation:**
```bash
# Production dependencies
uv add "pydantic-settings>=2.13" "psycopg[binary]>=3.3" "psycopg-pool>=3.2" "aiosqlite>=0.22" "alembic>=1.18" "PyYAML>=6.0" "python-dotenv>=1.0"

# Dev dependencies
uv add --dev "testcontainers[postgres]>=4.14"
```

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/
  config/
    __init__.py          # exports ZerothSettings, get_settings()
    settings.py          # Root settings model + sub-models
  storage/
    __init__.py          # Updated exports
    sqlite.py            # Existing SQLiteDatabase (kept for reference/compat)
    database.py          # Abstract async Database protocol
    async_sqlite.py      # AsyncSQLiteDatabase implementation
    async_postgres.py    # AsyncPostgresDatabase implementation
    factory.py           # create_database() factory from settings
    redis.py             # Existing (RedisConfig absorbed into settings)
  migrations/
    alembic.ini          # Alembic config (or at project root)
    env.py               # Alembic environment script
    versions/
      001_initial_schema.py  # Initial migration covering all 8 tables
```

### Pattern 1: Unified Settings Model
**What:** Single pydantic-settings BaseSettings with nested sub-models and YAML + env var sources
**When to use:** Always -- one global settings instance at startup
**Example:**
```python
# src/zeroth/config/settings.py
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class DatabaseSettings(BaseSettings):
    backend: str = "sqlite"                 # "sqlite" or "postgres"
    sqlite_path: str = "zeroth.db"
    postgres_dsn: SecretStr | None = None
    postgres_pool_min: int = 2
    postgres_pool_max: int = 10
    encryption_key: SecretStr | None = None

class RedisSettings(BaseSettings):
    # Absorbs existing RedisConfig fields
    mode: str = "local"
    host: str = "127.0.0.1"
    port: int = 6379
    # ... rest of existing RedisConfig fields

class AuthSettings(BaseSettings):
    api_keys_json: str | None = None
    bearer_json: str | None = None

class ZerothSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="ZEROTH_",
        env_nested_delimiter="__",
        yaml_file="zeroth.yaml",
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    database: DatabaseSettings = Field(default_factory=DatabaseSettings)
    redis: RedisSettings = Field(default_factory=RedisSettings)
    auth: AuthSettings = Field(default_factory=AuthSettings)

    @classmethod
    def settings_customise_sources(cls, settings_cls, **kwargs):
        from pydantic_settings import (
            DotEnvSettingsSource,
            EnvSettingsSource,
            YamlConfigSettingsSource,
        )
        return (
            EnvSettingsSource(settings_cls),          # Highest priority
            DotEnvSettingsSource(settings_cls),       # Second
            YamlConfigSettingsSource(settings_cls),   # Lowest (defaults)
        )
```

### Pattern 2: Async Database Protocol
**What:** Abstract protocol that normalizes SQLite and Postgres differences behind a common async interface
**When to use:** All repository code -- never import sqlite3 or psycopg directly in repositories

**Recommendation for placeholder normalization:** Repositories continue using `?` placeholders. The Database implementations translate internally:
- `AsyncSQLiteDatabase`: passes `?` through to aiosqlite (native)
- `AsyncPostgresDatabase`: converts `?` to `%s` before passing to psycopg3

**Example:**
```python
# src/zeroth/storage/database.py
from __future__ import annotations
from typing import Any, Protocol, runtime_checkable
from contextlib import asynccontextmanager
from collections.abc import AsyncIterator

@runtime_checkable
class AsyncDatabase(Protocol):
    """Abstract async database interface for all repositories."""

    @asynccontextmanager
    async def transaction(self) -> AsyncIterator[AsyncConnection]:
        """Open a connection, yield it, commit or rollback."""
        ...

    async def run_migrations(self, alembic_cfg_path: str) -> None:
        """Apply pending Alembic migrations."""
        ...


@runtime_checkable
class AsyncConnection(Protocol):
    """Abstraction over a database connection within a transaction."""

    async def execute(self, sql: str, params: tuple[Any, ...] = ()) -> None:
        ...

    async def fetch_one(self, sql: str, params: tuple[Any, ...] = ()) -> dict[str, Any] | None:
        ...

    async def fetch_all(self, sql: str, params: tuple[Any, ...] = ()) -> list[dict[str, Any]]:
        ...

    async def execute_script(self, sql: str) -> None:
        """Execute a multi-statement SQL script (used by Alembic)."""
        ...
```

### Pattern 3: Postgres AsyncConnectionPool Lifecycle
**What:** Pool created in FastAPI lifespan, passed to Database implementation
**When to use:** Production Postgres path

**Recommendation:** Use `psycopg_pool.AsyncConnectionPool` with `open=False`, then `await pool.open()` in FastAPI lifespan. Close in lifespan shutdown.

```python
# src/zeroth/storage/async_postgres.py
from psycopg_pool import AsyncConnectionPool

class AsyncPostgresDatabase:
    def __init__(self, pool: AsyncConnectionPool):
        self._pool = pool

    @asynccontextmanager
    async def transaction(self):
        async with self._pool.connection() as conn:
            async with conn.transaction():
                yield PostgresConnection(conn)

    @classmethod
    async def create(cls, dsn: str, *, min_size: int = 2, max_size: int = 10):
        pool = AsyncConnectionPool(dsn, min_size=min_size, max_size=max_size, open=False)
        await pool.open()
        return cls(pool)

    async def close(self):
        await self._pool.close()
```

### Pattern 4: Repository Async Rewrite Pattern
**What:** Convert `with self._database.transaction() as conn:` to `async with`
**When to use:** All 7 repositories

**Before (current):**
```python
def save(self, graph: Graph) -> Graph:
    with self._database.transaction() as connection:
        existing = self._fetch_row(connection, graph.graph_id, graph.version)
        ...
```

**After:**
```python
async def save(self, graph: Graph) -> Graph:
    async with self._database.transaction() as connection:
        existing = await connection.fetch_one(
            "SELECT status, payload FROM graph_versions WHERE graph_id = ? AND version = ?",
            (graph.graph_id, graph.version),
        )
        ...
```

### Pattern 5: Alembic with Raw SQL (No ORM)
**What:** Migration files use `op.execute()` with raw DDL, no autogenerate
**When to use:** All schema migrations

```python
# src/zeroth/migrations/versions/001_initial_schema.py
"""Initial schema for all Zeroth repositories."""

revision = "001"
down_revision = None

from alembic import op

def upgrade():
    # Use dialect-compatible SQL
    op.execute("""
        CREATE TABLE IF NOT EXISTS graph_versions (
            graph_id TEXT NOT NULL,
            version INTEGER NOT NULL,
            status TEXT NOT NULL,
            schema_version INTEGER NOT NULL DEFAULT 1,
            payload TEXT NOT NULL,
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL,
            PRIMARY KEY (graph_id, version)
        )
    """)
    # ... remaining tables

def downgrade():
    op.execute("DROP TABLE IF EXISTS graph_versions")
    # ...
```

### Anti-Patterns to Avoid
- **Dialect-specific SQL in repositories:** Never use PRAGMAs, `executescript`, `REAL` (SQLite) or `DOUBLE PRECISION` (Postgres) in repository code. Keep all SQL in the abstract layer or migrations.
- **Inline migrations in repositories:** Current pattern has each repository calling `apply_migrations()` in `__init__`. Replace with Alembic run at startup before any repository is created.
- **Blocking async calls:** Never use `asyncio.run()` inside an async context. Use `await` throughout.
- **Connection-per-query in Postgres:** Always use the connection pool. Never create individual connections.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Config loading from multiple sources | Custom env/YAML merge logic | pydantic-settings `settings_customise_sources` | Built-in source priority, validation, nested model support |
| SQL parameter placeholder conversion | Regex replacement in every query | Single `_normalize_params()` in AsyncConnection wrapper | One bug-free place; `?` to `%s` is simple but error-prone if scattered |
| Connection pooling | Manual pool with semaphore | `psycopg_pool.AsyncConnectionPool` | Handles health checks, connection recycling, wait queues |
| Schema migrations | Keep custom `Migration` dataclass | Alembic | Handles version tracking, multi-dialect, up/down, CLI tooling |
| Docker test containers | Manual docker-compose for tests | testcontainers-python | Automatic lifecycle, port mapping, wait strategies |

**Key insight:** The custom Migration system, while clean, cannot handle multi-dialect DDL or produce the audit trail that Alembic provides. Replacing it is essential for the Postgres path.

## Common Pitfalls

### Pitfall 1: SQL Placeholder Mismatch
**What goes wrong:** Repositories use `?` (sqlite3 dbapi2), psycopg3 uses `%s`. Queries silently fail or produce wrong results.
**Why it happens:** Both are valid Python DB-API 2.0 paramstyles but they differ by driver.
**How to avoid:** Normalize in the AsyncConnection wrapper. Repositories always write `?`, the Postgres wrapper converts to `%s` before execution. Write a single `_convert_placeholders(sql)` function.
**Warning signs:** `ProgrammingError: syntax error at or near "?"` from psycopg3.

### Pitfall 2: SQLite TEXT vs Postgres TIMESTAMPTZ
**What goes wrong:** SQLite stores timestamps as TEXT (ISO format strings), Postgres returns `datetime` objects from TIMESTAMPTZ columns.
**Why it happens:** SQLite has no native datetime type; the codebase stores ISO strings and parses them manually.
**How to avoid:** Keep TEXT columns for timestamps in both dialects for now (Postgres TEXT works fine for ISO strings). This avoids rewriting all timestamp parsing. Migration to proper TIMESTAMPTZ can be deferred.
**Warning signs:** `TypeError` when comparing `str` to `datetime`.

### Pitfall 3: executescript is SQLite-Only
**What goes wrong:** `connection.executescript(sql)` does not exist in psycopg3.
**Why it happens:** It is a sqlite3-specific method that runs multiple semicolon-separated statements.
**How to avoid:** The `execute_script()` method on the async protocol must split and execute statements individually, or use psycopg3's `conn.execute()` which can handle multi-statement scripts when properly configured.
**Warning signs:** `AttributeError: 'AsyncConnection' object has no attribute 'executescript'`.

### Pitfall 4: ON CONFLICT Syntax Compatibility
**What goes wrong:** Both SQLite (3.24+) and Postgres support `ON CONFLICT ... DO UPDATE SET`, but column quoting and type casting may differ.
**Why it happens:** Subtle dialect differences in constraint naming and type handling.
**How to avoid:** The existing `ON CONFLICT(column) DO UPDATE SET` syntax used in the codebase is compatible with both dialects. Verify each upsert query in tests.
**Warning signs:** Test passes on SQLite but fails on Postgres with constraint errors.

### Pitfall 5: Async Init in Repository Constructors
**What goes wrong:** Current repositories call `self._database.apply_migrations()` in `__init__()`. You cannot `await` in `__init__`.
**Why it happens:** Sync-to-async migration -- constructors cannot be async.
**How to avoid:** Move migration execution to a startup phase (in FastAPI lifespan or a `bootstrap()` async factory). Repositories should NOT run migrations -- that is the platform's responsibility.
**Warning signs:** `RuntimeWarning: coroutine was never awaited`.

### Pitfall 6: aiosqlite Row Access Pattern
**What goes wrong:** sqlite3.Row supports `row["column_name"]` access, but aiosqlite returns the same Row type. However, the dict conversion pattern differs.
**Why it happens:** aiosqlite wraps sqlite3 but the connection.row_factory must be set.
**How to avoid:** Set `row_factory = aiosqlite.Row` on the connection, or convert rows to dicts in the AsyncSQLiteConnection wrapper's `fetch_one`/`fetch_all` methods.
**Warning signs:** `TypeError: tuple indices must be integers or slices, not str`.

### Pitfall 7: Alembic Requires SQLAlchemy Engine Even for Raw SQL
**What goes wrong:** Even though migrations are raw SQL, Alembic needs a SQLAlchemy engine/connection to execute them.
**Why it happens:** Alembic is built on SQLAlchemy's connection layer.
**How to avoid:** Create a lightweight SQLAlchemy engine (no ORM models needed) just for Alembic's `env.py`. Use `create_engine(url)` from sqlalchemy and pass the connection to `context.configure(connection=connection)`. This is standard practice.
**Warning signs:** Import errors or `context.configure()` failures if you try to bypass SQLAlchemy entirely.

## Code Examples

### YAML Config File (zeroth.yaml)
```yaml
# Default configuration -- override with env vars (ZEROTH_DATABASE__BACKEND=postgres)
database:
  backend: sqlite
  sqlite_path: zeroth.db
  postgres_pool_min: 2
  postgres_pool_max: 10

redis:
  mode: local
  host: 127.0.0.1
  port: 6379
  key_prefix: zeroth

auth:
  # API keys and bearer config loaded from env vars for security
```

### Database Factory
```python
# src/zeroth/storage/factory.py
from zeroth.config import ZerothSettings
from zeroth.storage.database import AsyncDatabase

async def create_database(settings: ZerothSettings) -> AsyncDatabase:
    if settings.database.backend == "postgres":
        from zeroth.storage.async_postgres import AsyncPostgresDatabase
        dsn = settings.database.postgres_dsn.get_secret_value()
        return await AsyncPostgresDatabase.create(
            dsn,
            min_size=settings.database.postgres_pool_min,
            max_size=settings.database.postgres_pool_max,
        )
    else:
        from zeroth.storage.async_sqlite import AsyncSQLiteDatabase
        return AsyncSQLiteDatabase(
            path=settings.database.sqlite_path,
            encryption_key=(
                settings.database.encryption_key.get_secret_value()
                if settings.database.encryption_key else None
            ),
        )
```

### Placeholder Normalization
```python
import re

_PLACEHOLDER_RE = re.compile(r"\?")

def sqlite_to_psycopg(sql: str) -> str:
    """Convert ? placeholders to %s for psycopg3."""
    return _PLACEHOLDER_RE.sub("%s", sql)
```

### testcontainers Postgres Fixture
```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:17") as pg:
        yield pg

@pytest.fixture
async def postgres_db(postgres_container):
    from zeroth.storage.async_postgres import AsyncPostgresDatabase
    dsn = postgres_container.get_connection_url()
    db = await AsyncPostgresDatabase.create(dsn.replace("psycopg2", ""))
    # Run migrations
    # ...
    yield db
    await db.close()
```

### Alembic env.py for Dual-Dialect Support
```python
# src/zeroth/migrations/env.py
from alembic import context
from sqlalchemy import create_engine

def run_migrations_online():
    url = context.config.get_main_option("sqlalchemy.url")
    engine = create_engine(url)
    with engine.connect() as connection:
        context.configure(connection=connection, target_metadata=None)
        with context.begin_transaction():
            context.run_migrations()
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| psycopg2 sync | psycopg3 async with pool | 2023+ | Native async, server-side binding, better performance |
| Manual env var parsing | pydantic-settings v2 | 2023 | Type-safe, validated, multi-source config |
| Custom migration tracking | Alembic | Long-standing standard | CLI tooling, version control, multi-dialect |
| sqlite3 sync | aiosqlite | 2020+ | Non-blocking SQLite in async apps |
| AsyncConnectionPool(open=True) | AsyncConnectionPool(open=False) + explicit open | psycopg 3.2+ | Opening pool in constructor is deprecated |

## Open Questions

1. **Alembic SQLAlchemy dependency**
   - What we know: Alembic requires `sqlalchemy` as a dependency even for raw SQL migrations. The project does not currently use SQLAlchemy.
   - What's unclear: Whether adding sqlalchemy as a dependency conflicts with any project constraints.
   - Recommendation: Accept the dependency -- it is only used by Alembic's engine layer, not for ORM. Add `sqlalchemy>=2.0` to dependencies.

2. **Migration from per-repository schema versioning to Alembic**
   - What we know: Each repository currently maintains its own Migration list and schema_versions scope. Alembic uses a single `alembic_version` table.
   - What's unclear: Whether to migrate existing databases or only support fresh Alembic-managed databases.
   - Recommendation: For v1.1, support fresh databases only. Existing dev databases can be recreated. Add a one-time data migration script if needed later.

3. **EncryptedField in Postgres**
   - What we know: EncryptedField uses Fernet symmetric encryption and stores encrypted TEXT. This works identically in Postgres TEXT columns.
   - What's unclear: No issues expected.
   - Recommendation: Port EncryptedField unchanged -- it operates on Python strings, not database types.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python | Runtime | Yes | 3.12 (venv) | -- |
| PostgreSQL (psql) | Postgres backend testing | Yes | 17.9 (Homebrew) | testcontainers |
| Docker | testcontainers-python | Yes | 29.2.0 | Cannot run Postgres integration tests without Docker |
| uv | Package management | Yes | (installed) | -- |
| pydantic | Already installed | Yes | 2.12.5 | -- |

**Missing dependencies with no fallback:**
- None -- all required infrastructure is available.

**Missing dependencies with fallback:**
- None.

## Project Constraints (from CLAUDE.md)

- **Build/test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` (main package), `tests/` (pytest tests)
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Test framework:** pytest with `asyncio_mode = "auto"` (already configured in pyproject.toml)
- **Linter:** ruff with line-length 100, target Python 3.12
- **Dependency management:** uv (not pip)
- **Test count baseline:** 280 existing tests must continue to pass with `ZEROTH_DB_BACKEND=sqlite`

## Sources

### Primary (HIGH confidence)
- [pydantic-settings PyPI](https://pypi.org/project/pydantic-settings/) - v2.13.1, YAML support confirmed
- [psycopg PyPI](https://pypi.org/project/psycopg/) - v3.3.3, async mode confirmed
- [psycopg-pool PyPI](https://pypi.org/project/psycopg-pool/) - v3.2.6, AsyncConnectionPool
- [psycopg3 connection pool docs](https://www.psycopg.org/psycopg3/docs/advanced/pool.html) - open=False pattern
- [psycopg3 parameter passing docs](https://www.psycopg.org/psycopg3/docs/basic/params.html) - %s placeholder format
- [aiosqlite PyPI](https://pypi.org/project/aiosqlite/) - v0.22.1
- [Alembic PyPI](https://pypi.org/project/alembic/) - v1.18.4, raw SQL support confirmed
- [testcontainers PyPI](https://pypi.org/project/testcontainers/) - v4.14.2

### Secondary (MEDIUM confidence)
- [pydantic-settings YAML configuration](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) - YamlConfigSettingsSource usage
- [Alembic raw SQL without ORM](https://github.com/sqlalchemy/alembic/discussions/1630) - confirmed viable at scale
- [psycopg3 async best practices](https://www.psycopg.org/psycopg3/docs/advanced/async.html) - concurrent operations

### Tertiary (LOW confidence)
- None -- all findings verified against official sources.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all packages verified on PyPI with current versions; pydantic-settings YAML support confirmed in official docs
- Architecture: HIGH - patterns based on official psycopg3 and pydantic-settings documentation; async database protocol is a well-established pattern
- Pitfalls: HIGH - placeholder mismatch, executescript, and Alembic/SQLAlchemy dependency verified against official docs; codebase audit confirms all 13 `?`-placeholder occurrences across 9 files

**Research date:** 2026-04-06
**Valid until:** 2026-05-06 (stable libraries, 30-day validity)
