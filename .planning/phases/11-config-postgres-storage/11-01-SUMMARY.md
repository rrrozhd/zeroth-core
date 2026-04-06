---
phase: 11-config-postgres-storage
plan: 01
subsystem: config, storage
tags: [config, database, async, alembic, pydantic-settings]
dependency_graph:
  requires: []
  provides: [ZerothSettings, AsyncDatabase, AsyncConnection, AsyncSQLiteDatabase, AsyncPostgresDatabase, create_database]
  affects: [storage, config, migrations]
tech_stack:
  added: [pydantic-settings, psycopg3, psycopg-pool, aiosqlite, alembic, sqlalchemy, PyYAML, python-dotenv, testcontainers]
  patterns: [protocol-based-abstraction, factory-pattern, settings-singleton]
key_files:
  created:
    - src/zeroth/config/__init__.py
    - src/zeroth/config/settings.py
    - src/zeroth/storage/database.py
    - src/zeroth/storage/async_sqlite.py
    - src/zeroth/storage/async_postgres.py
    - src/zeroth/storage/factory.py
    - src/zeroth/migrations/__init__.py
    - src/zeroth/migrations/env.py
    - src/zeroth/migrations/versions/__init__.py
    - src/zeroth/migrations/versions/001_initial_schema.py
    - alembic.ini
    - zeroth.yaml
    - tests/test_config.py
    - tests/test_async_database.py
  modified:
    - pyproject.toml
    - src/zeroth/storage/__init__.py
decisions:
  - Pydantic-settings with YamlConfigSettingsSource for unified config (env > .env > YAML priority)
  - Runtime-checkable Protocol for AsyncDatabase allows isinstance checks
  - Per-transaction connections in AsyncSQLiteDatabase (no persistent connection)
  - Placeholder conversion (? to %s) in Postgres adapter keeps repositories dialect-agnostic
  - Alembic initial migration consolidates all 10 existing tables into one revision
metrics:
  duration: 266s
  completed: 2026-04-06
  tasks_completed: 2
  tasks_total: 2
  tests_added: 13
  tests_passing: 13
  files_created: 14
  files_modified: 2
requirements:
  - CFG-01
  - CFG-03
---

# Phase 11 Plan 01: Config & Async Database Foundation Summary

Unified pydantic-settings config with YAML/env/dotenv sources, async Database protocol with SQLite and Postgres implementations, factory-based backend selection, and Alembic initial migration consolidating all existing tables.

## What Was Done

### Task 1: Config Package with Unified Settings Model
- Added 8 production dependencies (pydantic-settings, psycopg, aiosqlite, alembic, sqlalchemy, PyYAML, python-dotenv) and 1 dev dependency (testcontainers)
- Created `src/zeroth/config/` package with `ZerothSettings(BaseSettings)` using `ZEROTH_` prefix and `__` nested delimiter
- Sub-models: `DatabaseSettings`, `RedisSettings` (absorbing existing RedisConfig fields), `AuthSettings`
- Config priority: env vars > .env file > zeroth.yaml defaults via custom `settings_customise_sources()`
- Module-level singleton via `get_settings()`
- Created `zeroth.yaml` with sensible defaults
- 5 tests covering defaults, env overrides, nested delimiter, field absorption
- **Commit:** 1cbcf92

### Task 2: Async Database Protocol, Implementations, Factory, and Alembic Migrations
- Defined `AsyncDatabase` and `AsyncConnection` as `@runtime_checkable` protocols in `database.py`
- `AsyncSQLiteDatabase`: per-transaction aiosqlite connections with WAL mode, foreign keys, PRAGMA synchronous=NORMAL
- `AsyncPostgresDatabase`: psycopg3 `AsyncConnectionPool` with `?` to `%s` placeholder conversion
- `create_database()` factory reads `settings.database.backend` to select implementation
- Updated `storage/__init__.py` with all new exports
- Created Alembic infrastructure: `alembic.ini`, `env.py`, and initial migration `001_initial_schema.py`
- Migration consolidates all 10 tables: schema_versions, graph_versions, runs, run_checkpoints, threads, contract_versions, deployment_versions, approvals, node_audits, rate_limit_buckets, quota_counters
- 8 tests covering transactions, rollback, fetch_one/fetch_all, factory, placeholder conversion, protocol compliance
- **Commit:** 4ccd14b

## Decisions Made

1. **Config source priority:** env vars (highest) > .env > zeroth.yaml (lowest) -- matches pydantic-settings conventions and CONTEXT.md D-06
2. **Protocol pattern:** `@runtime_checkable` protocols for AsyncDatabase/AsyncConnection -- enables isinstance checks without ABC coupling
3. **Per-transaction SQLite connections:** AsyncSQLiteDatabase opens a new aiosqlite connection per transaction() call, no persistent connection -- simpler lifecycle, no stale connections
4. **Dialect-agnostic SQL:** Postgres adapter converts `?` to `%s` internally so repositories write SQLite-style parameters everywhere
5. **Consolidated Alembic migration:** All 10 tables from 7 repositories consolidated into a single `001_initial_schema.py` -- clean starting point for new deployments

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all implementations are functional.

## Verification Results

1. `uv run pytest tests/test_config.py tests/test_async_database.py -v` -- 13/13 tests pass
2. `uv run ruff check` on all new files -- no lint errors
3. `python -c "from zeroth.config import ZerothSettings; s = ZerothSettings(); print(s.database.backend)"` -- prints "sqlite"
4. `python -c "from zeroth.storage import AsyncDatabase, AsyncSQLiteDatabase; print(isinstance(AsyncSQLiteDatabase('/tmp/test.db'), AsyncDatabase))"` -- prints "True"

## Self-Check: PASSED

All 14 created files verified present. Both commits (1cbcf92, 4ccd14b) verified in git log.
