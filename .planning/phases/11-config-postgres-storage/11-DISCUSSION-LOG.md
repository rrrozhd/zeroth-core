# Phase 11: Config & Postgres Storage - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-06
**Phase:** 11-config-postgres-storage
**Areas discussed:** Config architecture, Repository abstraction, Migration strategy, Testing approach

---

## Config Architecture

### Settings model structure

| Option | Description | Selected |
|--------|-------------|----------|
| Single flat BaseSettings | One ZerothSettings class with all env vars. Simple, all-in-one. | |
| Nested sub-models | ZerothSettings with typed sub-models (db, redis, auth). More organized. | ✓ |
| You decide | Claude picks the approach. | |

**User's choice:** Nested sub-models
**Notes:** None

### RedisConfig integration

| Option | Description | Selected |
|--------|-------------|----------|
| Absorb into settings | RedisConfig becomes a sub-model. One source of truth. | ✓ |
| Keep separate | RedisConfig stays standalone. New settings only covers DB and service. | |
| You decide | Claude picks. | |

**User's choice:** Absorb into settings
**Notes:** None

### Settings module location

| Option | Description | Selected |
|--------|-------------|----------|
| src/zeroth/settings.py | Top-level module. | |
| src/zeroth/config/ | Dedicated config package. | ✓ |
| You decide | Claude picks. | |

**User's choice:** src/zeroth/config/
**Notes:** None

### Env var prefix

| Option | Description | Selected |
|--------|-------------|----------|
| ZEROTH_ prefix | Consistent with existing ZEROTH_REDIS_* and ZEROTH_SERVICE_* vars. | ✓ |
| No prefix | Standard names like DB_HOST. Simpler but risk collision. | |
| You decide | Claude picks. | |

**User's choice:** ZEROTH_ prefix
**Notes:** None

### YAML config support

**User's addition:** User proactively suggested YAML config files loaded into pydantic models alongside .env via load_dotenv.

| Option | Description | Selected |
|--------|-------------|----------|
| YAML primary, env overrides | YAML holds all config structure/defaults. Env vars override for deployment/secrets. | ✓ |
| Env primary, YAML optional | Env vars main source. YAML optional convenience for local dev. | |
| You decide | Claude picks. | |

**User's choice:** YAML primary, env overrides
**Notes:** User specifically wanted YAML as the primary configuration source.

---

## Repository Abstraction

### Multi-backend strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Abstract Database interface | Database protocol both SQLite and Postgres implement. Same transaction/execute API. | ✓ |
| SQLAlchemy Core | Replace raw SQL with SQLAlchemy Core. Handles dialect differences. Bigger change. | |
| Dual implementations | Keep SQLiteDatabase, write parallel Postgres repos. Backend flag picks. | |
| You decide | Claude picks. | |

**User's choice:** Abstract Database interface
**Notes:** None

### Sync vs Async

| Option | Description | Selected |
|--------|-------------|----------|
| Keep sync | Postgres adapter is a drop-in swap. Minimal changes. | |
| Full async now | Rewrite all repos to async. Bigger scope but better long-term. | ✓ |
| Sync now, async later | Ship sync this phase. Plan async migration later. | |

**User's choice:** Full async now
**Notes:** User expressed strong preference for async as "the way to go when building apps". After discussing the scope implications (all 7 repos + all callers need rewrite), user confirmed full async.

### Postgres driver

| Option | Description | Selected |
|--------|-------------|----------|
| psycopg 3 async | Same library handles sync and async. AsyncConnection API. | ✓ |
| asyncpg | Purpose-built async driver. Faster raw performance. Different API. | |

**User's choice:** psycopg 3 async
**Notes:** None

---

## Migration Strategy

### Alembic coexistence

| Option | Description | Selected |
|--------|-------------|----------|
| Alembic for Postgres only | Keep existing Migration system for SQLite. Alembic for Postgres. Two paths. | |
| Alembic for both | Migrate SQLite to Alembic too. Single system. More work upfront. | ✓ |
| You decide | Claude picks. | |

**User's choice:** Alembic for both
**Notes:** None

### Alembic approach

| Option | Description | Selected |
|--------|-------------|----------|
| Raw SQL migrations | Hand-written DDL. No SQLAlchemy models. Keeps raw SQL pattern. | ✓ |
| SQLAlchemy Table metadata | Define tables as SA objects. Auto-generates migrations from metadata. | |
| You decide | Claude picks. | |

**User's choice:** Raw SQL migrations
**Notes:** None

### Migration location

| Option | Description | Selected |
|--------|-------------|----------|
| src/zeroth/migrations/ | Inside the package. Ships with the app. | ✓ |
| migrations/ at repo root | Separate from source. Django-style. | |
| You decide | Claude picks. | |

**User's choice:** src/zeroth/migrations/
**Notes:** None

---

## Testing Approach

### Postgres testing

| Option | Description | Selected |
|--------|-------------|----------|
| testcontainers-python | Real Postgres containers in pytest. No Docker setup in CI. | ✓ |
| docker-compose in CI | CI starts Postgres via docker-compose. Local devs run manually. | |
| Both backends in same suite | Parametrize tests for both SQLite and Postgres. Maximum coverage. | |
| You decide | Claude picks. | |

**User's choice:** testcontainers-python
**Notes:** None

### Test parametrization

| Option | Description | Selected |
|--------|-------------|----------|
| Parametrize both backends | Key repo tests run against both SQLite and Postgres. Catches dialect differences. | ✓ |
| SQLite default, Postgres separate | Existing 280 tests stay SQLite. Separate Postgres suite. Less CI time. | |
| You decide | Claude picks. | |

**User's choice:** Parametrize both backends
**Notes:** None

---

## Claude's Discretion

- Async Database protocol method signatures
- Connection pooling strategy for Postgres
- aiosqlite wrapping details
- Alembic env.py configuration
- YAML config file naming/discovery

## Deferred Ideas

None — discussion stayed within phase scope
