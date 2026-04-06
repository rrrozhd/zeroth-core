# Stack Research

**Domain:** Production-readiness additions to a governed multi-agent platform (Python/FastAPI)
**Researched:** 2026-04-06
**Confidence:** HIGH (current versions verified via PyPI/official sources; integration approach cross-checked against existing codebase)

---

## Existing Stack (Do Not Duplicate)

The following is already present and must not be replaced:

| Technology | Version | Role |
|------------|---------|------|
| Python | >=3.12 | Primary language |
| FastAPI | >=0.115 | HTTP API framework |
| Pydantic | >=2.10 | Validation, settings |
| Uvicorn | >=0.30 | ASGI server |
| redis | >=5.0.0 | Distributed runtime state (GovernAI-backed stores) |
| PyJWT[crypto] | >=2.10 | JWT bearer verification |
| httpx | >=0.27 | Async HTTP client |
| governai | git@7452de4 | Core governance engine (GovernedLLM, stores, audit) |
| SQLite | stdlib | Current persistence layer (dev/test retained) |

---

## Recommended Stack — New Additions Only

### LLM Provider Integration

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `langchain-openai` | >=0.3.12 | ChatOpenAI for GovernedLLM.from_chat_openai() | GovernAI's `GovernedLLM` wraps LangChain chat models; `from_chat_openai()` is its only named constructor. GovernAI already depends on `langchain>=1.2.10` and `langchain-openai>=1.1.10` transitively — pinning explicitly ensures version clarity in Zeroth's own pyproject.toml |
| `langchain-anthropic` | >=0.3.0 | ChatAnthropic for GovernedLLM wrapping | GovernAI's `GovernedLLM` accepts any LangChain chat model; langchain-anthropic provides `ChatAnthropic` which slots in identically to ChatOpenAI. No direct Anthropic SDK calls needed at the Zeroth layer |
| `openai` | >=2.20,<3.0 | Direct OpenAI SDK (for Regulus instrumentation) | Regulus' `econ_instrumentation` `integrations[integrations]` extra pins `openai>=1.40,<2.0` but the SDK is now at v2.x — verify compatibility with Regulus pinned range before adding to Zeroth's deps |
| `anthropic` | >=0.87,<1.0 | Direct Anthropic SDK (for Regulus instrumentation) | Same rationale as openai; Regulus pins `anthropic>=0.34,<1.0`, current is 0.89 — within range |

**Confidence:** HIGH for langchain-openai/anthropic (verified against GovernAI source). MEDIUM for direct SDK versions (Regulus pin ranges need re-verification when GovernAI commit is updated).

**Integration note:** Do not bypass GovernedLLM to call LLM providers directly from Zeroth. GovernedLLM is the normalization layer. Add `langchain-openai` and `langchain-anthropic` as explicit deps in Zeroth's pyproject.toml so they are not implicitly pinned only through GovernAI.

---

### Regulus Economics SDK

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `econ-instrumentation-sdk` | 0.1.1 (local path) | Token metering, cost attribution per node/run/tenant, budget enforcement | This is the Regulus SDK at `/Users/dondoe/coding/regulus/sdk/python/`. Add as local path dep in dev (`file:///...`) or package as a wheel for production. The SDK is async-safe (ContextVar-based), uses httpx for transport, and its Pydantic models (`ExecutionEvent`, `OutcomeEvent`) align with Zeroth's data model conventions |

**Integration pattern:**
- Use `econ_instrumentation.configure(InstrumentationConfig(...))` at app startup via FastAPI lifespan
- Wrap each agent node execution with `track_execution(join_key=run_id, capability_id=node_id)` context manager
- Use `instrument_openai_async_client` / `instrument_anthropic_async_client` for auto-capture if using provider SDKs directly
- `InstrumentationConfig.base_url` points to the companion Regulus FastAPI backend (not embedded in Zeroth)
- Env vars: `ECP_BASE_URL`, `ECP_ENABLED`, `ECP_CAPTURE_CONTENT`

**Confidence:** HIGH (code-verified from local Regulus SDK source).

---

### Production Storage — Postgres

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `sqlalchemy` | >=2.0.49,<3.0 | Async ORM / query builder for Postgres | Industry standard for Python + Postgres. v2.0 has first-class asyncio support. Zeroth's existing custom SQLite layer uses versioned migrations and raw SQL — SQLAlchemy's migration + ORM sits above this cleanly. The modular monolith pattern calls for a single engine shared across all repository adapters |
| `asyncpg` | >=0.31.0 | Async Postgres driver (backend for SQLAlchemy async) | Fastest Python Postgres driver (binary protocol, C-implemented). Required by `create_async_engine("postgresql+asyncpg://...")`. Outperforms psycopg3 for raw throughput; psycopg3 offers more Pythonic API but asyncpg is the standard SQLAlchemy async backend |
| `alembic` | >=1.18.4 | Schema migrations for Postgres | Standard companion to SQLAlchemy. Initialize with `alembic init -t async` for async-engine compatibility. Run migrations at startup or as a separate step — do NOT auto-migrate in production |

**Pattern:**
```
Postgres repositories (new) → AsyncSession (SQLAlchemy 2) → asyncpg → Postgres
SQLite repositories (existing) → SQLiteDatabase wrapper → SQLite (dev/test only)
```

Zeroth's existing `Migration` dataclass system is for SQLite only. For Postgres, Alembic is the replacement. The two can coexist during transition: SQLite for local dev, Postgres for production, controlled by a `ZEROTH_DB_BACKEND=sqlite|postgres` config flag.

**Confidence:** HIGH (versions verified via PyPI; SQLAlchemy 2.0.49 released 2026-04-03, alembic 1.18.4 released 2026-02-10, asyncpg 0.31.0 current).

---

### Message Queue — Durable Distributed Dispatch

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `arq` | >=0.26 | Async Redis-backed distributed task queue | Zeroth already has Redis as a required dependency. ARQ is async-native (asyncio from the ground up), requires no new infrastructure beyond Redis already in place, and integrates with FastAPI's lifespan pattern cleanly. The existing `RunWorker` poll-loop maps directly to an ARQ worker function. ARQ's job deduplication prevents duplicate execution analogous to Zeroth's current LeaseManager |

**Why not Celery:** Celery is sync-first and requires spawning separate processes. Zeroth's async FastAPI runtime would need `celery --pool gevent` workarounds. ARQ runs worker coroutines on the same asyncio event loop as the app.

**Why not Dramatiq:** Dramatiq supports RabbitMQ + Redis but adds a broker abstraction layer. Zeroth doesn't need broker flexibility — Redis is locked in via GovernAI. ARQ's simpler surface area reduces integration risk.

**Why not plain Redis Streams:** ARQ wraps Redis Streams/sorted sets with a clean Python API for job scheduling, retries, priorities, and timeouts. Building this from raw Redis commands would duplicate ARQ's battle-tested logic.

**Integration note:** The existing `LeaseManager` and `RunWorker` can be refactored to emit jobs via `arq.ArqRedis.enqueue_job()` and consume them in an ARQ worker class. The current SQLite-based dispatch (durable lease store) continues for dev; ARQ provides the production-grade distributed queue.

**Confidence:** MEDIUM (ARQ versions not verified against PyPI in this session; logic-verified as right choice given constraints).

---

### External Memory Connectors

#### Redis (already present — extend for memory)

The existing `redis>=5.0.0` dependency is already used for GovernAI runtime stores. Extending it for external agent memory (key-value, conversation history) requires no new package — only new Redis key namespacing and TTL strategy.

**Pattern:** Use `zeroth:memory:{tenant_id}:{thread_id}` key prefix with configurable TTL. Implement as a new `RedisMemoryConnector` module that implements GovernAI's memory interface.

#### Vector Store

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `pgvector` | >=0.4.2 | Vector similarity search in Postgres | Zeroth is already adding Postgres (above). pgvector adds a native Postgres extension, keeping the infrastructure footprint minimal — no separate vector database service (Pinecone, Weaviate, Qdrant) needed for MVP. pgvector-python supports asyncpg and SQLAlchemy natively. For production scale requiring dedicated vector DB, this is easily swapped |

**Confidence:** HIGH (pgvector 0.4.2 verified as current on PyPI April 2026; async support confirmed via pgvector-python GitHub).

**What NOT to use:** Do not add `langchain-postgres` as a vector store dependency. It bundles LangChain's `PGVectorStore` which pulls in LangChain community packages Zeroth does not otherwise need. Use `pgvector` directly with SQLAlchemy models instead.

---

### Retry & Resilience

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `tenacity` | >=9.1.4 | Provider-aware retry with exponential backoff and jitter | Standard Python retry library, async-native via `AsyncRetrying`. The existing GovernAI `GovernedLLM` does not implement retry internally — retries belong in Zeroth's provider adapter layer. Tenacity's `retry_if_exception_type`, `wait_exponential_jitter`, and `stop_after_attempt` compose cleanly into a `ProviderRetryPolicy` per provider/model |

**Pattern:**
```python
from tenacity import AsyncRetrying, retry_if_exception_type, wait_exponential_jitter, stop_after_attempt

async for attempt in AsyncRetrying(
    retry=retry_if_exception_type((RateLimitError, APITimeoutError)),
    wait=wait_exponential_jitter(initial=1, max=60),
    stop=stop_after_attempt(5),
):
    with attempt:
        response = await llm.ainvoke(messages)
```

**Confidence:** HIGH (tenacity 9.1.4 verified as current release on PyPI February 2026; async support verified via official docs).

---

### Container-Based Sandbox

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `docker` (docker SDK for Python) | >=7.1.0 | Spawn and exec into isolated containers for untrusted execution units | The existing Phase 8A sandbox uses `subprocess` + `tempdir`. Hardening requires full container isolation. Docker SDK provides `container.exec_run()` for command execution inside a running container, `containers.run()` for one-shot containers, and resource limit parameters (CPU, memory, network). This maps directly to Zeroth's `ExecutionUnit` interface |

**Container security posture:**
- Use `network_disabled=True`, `read_only=True` (with explicit tmpfs mounts) for untrusted units
- Set `mem_limit`, `cpu_quota`, `pids_limit` to cap runaway processes
- Base image: `python:3.12-slim` — minimal attack surface
- Do NOT mount the host Docker socket inside agent containers (privilege escalation vector)

**Why not nsjail:** nsjail requires Linux-specific syscalls and a separate binary install, making it non-portable across macOS dev and Linux prod. Docker SDK provides the same namespace/cgroup isolation with cross-platform tooling.

**Why not gVisor:** gVisor provides stronger isolation but requires a custom Docker runtime (`runsc`) that adds significant operational complexity. Appropriate for multi-tenant public execution — overkill for Zeroth's internal governed agents where identity/RBAC already constrains who submits execution units.

**Confidence:** MEDIUM (Docker SDK 7.1.0 verified via docs.docker.com; security posture from community best practices, not official benchmark).

---

### Containerized Deployment

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Docker multi-stage build | — | Production image packaging | Standard pattern: `builder` stage with `uv sync`, `runtime` stage with slim base. Zeroth's `uv` dependency management maps cleanly to Docker layer caching |
| `docker-compose` v2 | >=2.27 | Local development orchestration | Compose v2 (`docker compose` not `docker-compose`) is the current standard. Services: `zeroth`, `postgres`, `redis`, optional `regulus`. Use `depends_on` with `condition: service_healthy` for startup ordering |
| Traefik or Nginx | latest | TLS termination proxy | FastAPI docs and production guides recommend terminating TLS at the proxy, not in Uvicorn directly. Traefik handles Let's Encrypt automatically; Nginx is simpler for static cert deployment. Both are valid — choose based on operational preference |

**Confidence:** HIGH (standard containerization patterns, well-documented).

---

### Observability Additions

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `structlog` | >=25.0 | Structured JSON logging | Zeroth currently uses stdlib `logging`. Structlog wraps stdlib logging with a processor pipeline that emits JSON for production log aggregators (Datadog, CloudWatch, Loki). Async-safe via contextvars integration. Required for observability at production scale — plain text logs don't parse well at volume |

**Confidence:** MEDIUM (structlog version estimated; async+FastAPI integration pattern well-documented from multiple 2025-2026 sources).

---

### API Versioning and OpenAPI

No new library needed. FastAPI's built-in `APIRouter` with prefix `/v1` handles API versioning. `app.openapi()` generates OpenAPI 3.1 spec automatically — expose it at `/openapi.json`. For multi-version support, mount separate routers:

```python
app.include_router(v1_router, prefix="/v1")
app.include_router(v2_router, prefix="/v2")  # future
```

**Confidence:** HIGH (FastAPI built-in, no new dependency).

---

### Webhook / Callback Notifications

No new library needed. Use `httpx.AsyncClient` (already in pyproject.toml) from within FastAPI `BackgroundTasks` for outgoing webhook delivery. For reliable delivery with retry, wrap in tenacity (already above). For high-volume production, promote to ARQ job (also above).

**Confidence:** HIGH (httpx already present; pattern well-documented).

---

### Health Probes

No new library needed. Extend the existing `GET /health` endpoint with dependency checks: Postgres connectivity, Redis ping, GovernAI runtime state. Return structured JSON with per-dependency status for readiness vs. liveness distinction:

```json
{"status": "ready", "dependencies": {"postgres": "ok", "redis": "ok", "regulus": "degraded"}}
```

**Confidence:** HIGH (FastAPI built-in, no new dependency).

---

## Installation

```bash
# LLM provider integration (add to pyproject.toml dependencies)
uv add langchain-openai>=0.3.12
uv add langchain-anthropic>=0.3.0

# Regulus SDK (local path dep)
uv add "econ-instrumentation-sdk @ file:///Users/dondoe/coding/regulus/sdk/python"

# Postgres + ORM + migrations
uv add "sqlalchemy>=2.0.49,<3.0"
uv add asyncpg>=0.31.0
uv add alembic>=1.18.4

# Vector store (Postgres extension bridge)
uv add pgvector>=0.4.2

# Message queue
uv add arq>=0.26

# Retry
uv add tenacity>=9.1.4

# Container sandbox
uv add docker>=7.1.0

# Structured logging
uv add structlog>=25.0
```

---

## Alternatives Considered

| Recommended | Alternative | Why Not |
|-------------|-------------|---------|
| `asyncpg` | `psycopg3` | psycopg3 is more Pythonic but asyncpg is the default SQLAlchemy async backend; switching would require `postgresql+psycopg` URL scheme and psycopg3 async extra — adds setup friction with no benefit over asyncpg for Zeroth's workload |
| `arq` | `celery` | Celery is sync-first; async integration requires process-based workers. Zeroth is a pure asyncio service and adding celery workers would require a separate process pool and broker config (RabbitMQ or Redis with separate Celery-specific key namespacing) |
| `arq` | `dramatiq` | Dramatiq requires a separate broker process (or Redis in a different mode). ARQ reuses the same Redis instance Zeroth already depends on with zero new infrastructure |
| `pgvector` | Pinecone / Weaviate | External vector DB services add network latency, new auth credentials, and separate operational surface. pgvector runs inside Postgres (already added) — same infra, same ops model |
| `docker` SDK | `nsjail` | nsjail requires Linux-only syscall capabilities and a separate binary — incompatible with macOS dev workflow. Docker SDK works identically across dev (macOS Docker Desktop) and prod (Linux) |
| `tenacity` | `backoff` | `backoff` is older, less maintained, and doesn't have native async support as a first-class feature. Tenacity has `AsyncRetrying` natively |
| `structlog` | stdlib logging only | Stdlib logging produces unstructured text. JSON logs are required for production log aggregators and correlation ID tracing at scale |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Direct OpenAI/Anthropic SDK calls bypassing GovernedLLM | Loses normalization, tool-call extraction, and governance instrumentation that GovernedLLM provides | `GovernedLLM.from_chat_openai()` or `GovernedLLM(ChatAnthropic(...))` |
| `langchain-postgres` PGVectorStore | Pulls in LangChain community packages (langchain-community) not otherwise needed; adds ~15 transitive deps for a feature Zeroth can implement with `pgvector` + SQLAlchemy directly | `pgvector>=0.4.2` with SQLAlchemy `Vector` column type |
| `celery` | Not async-native; process-based workers conflict with Zeroth's single asyncio event loop design | `arq` |
| Dedicated vector DB (Pinecone, Qdrant, Weaviate) | Separate infrastructure + operational overhead for MVP. pgvector handles 99% of governed agent use cases at Zeroth's expected scale | `pgvector` extension on existing Postgres |
| Mounting Docker socket inside agent containers | Privilege escalation — any agent with socket access can spawn arbitrary containers on the host | Pass execution parameters as environment variables; never expose `/var/run/docker.sock` to untrusted containers |
| `alembic autogenerate` in production with `--autogenerate` | Auto-diff migrations can generate destructive statements (DROP COLUMN) unexpectedly | Review all generated migrations before applying; run `alembic check` in CI |
| TLS termination inside Uvicorn (`--ssl-certfile`) | Certificate rotation requires app restart; proxy-level termination allows zero-downtime cert renewal | Traefik or Nginx in front of Uvicorn on plain HTTP |

---

## Stack Patterns by Variant

**If running locally (dev/test):**
- Use SQLite (existing) — no Postgres needed
- Use Redis via Docker or local install
- Skip ARQ — existing RunWorker poll loop is sufficient
- Regulus SDK can disable transport (`ECP_ENABLED=false`)
- Docker sandbox optional (subprocess sandbox remains active)

**If running in production (Docker Compose):**
- Use Postgres with asyncpg via SQLAlchemy 2
- Use Redis with ARQ for distributed dispatch
- Enable Regulus SDK with `ECP_BASE_URL` pointing to companion service
- Enable Docker sandbox backend (requires Docker socket accessible to Zeroth container)
- Enable structlog JSON output
- TLS termination at Traefik/Nginx layer

**If horizontal scaling:**
- ARQ workers can run as separate containers (`arq zeroth.dispatch.worker.WorkerSettings`)
- Postgres connection pool via SQLAlchemy's `pool_size` / `max_overflow`
- Redis cluster or Redis Sentinel for HA
- Lease coordination for workers remains valid (arq's job deduplication replaces SQLite-based lease)

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| `sqlalchemy>=2.0.49` | `asyncpg>=0.31.0` | Use `create_async_engine("postgresql+asyncpg://...")` |
| `sqlalchemy>=2.0.49` | `alembic>=1.18.4` | Alembic 1.18 supports SQLAlchemy 2.0 async via `run_sync` pattern |
| `pgvector>=0.4.2` | `asyncpg>=0.31.0` | Use `pgvector.asyncpg` module; call `register_vector(conn)` on connection |
| `pgvector>=0.4.2` | `sqlalchemy>=2.0.49` | Use `from pgvector.sqlalchemy import Vector` column type |
| `langchain-openai>=0.3.12` | `governai@7452de4` | GovernAI pins `langchain-openai>=1.1.10` — version floor from GovernAI, ceiling from Zeroth |
| `langchain-anthropic>=0.3.0` | `governai@7452de4` | GovernAI does not pin langchain-anthropic; verify no conflicts after `uv sync` |
| `econ-instrumentation-sdk==0.1.1` | `pydantic>=2.7` | SDK pins `pydantic>=2.7.0`; Zeroth pins `pydantic>=2.10` — compatible |
| `econ-instrumentation-sdk==0.1.1` | `httpx>=0.27` | Both require `httpx>=0.27` — no conflict |
| `arq>=0.26` | `redis>=5.0.0` | ARQ uses redis-py under the hood; Zeroth's existing `redis>=5.0.0` dep covers this |
| `docker>=7.1.0` | Python 3.12 | Docker SDK 7.x requires Python >=3.9; compatible |

---

## Sources

- GovernAI source at `/Users/dondoe/coding/governai/governai/integrations/llm.py` — GovernedLLM implementation, LangChain dependency (HIGH confidence)
- Regulus SDK source at `/Users/dondoe/coding/regulus/sdk/python/` — econ_instrumentation interface, config, transport (HIGH confidence)
- PyPI asyncpg — version 0.31.0 confirmed current (HIGH confidence)
- PyPI SQLAlchemy — version 2.0.49 released 2026-04-03 (HIGH confidence)
- PyPI alembic — version 1.18.4 released 2026-02-10 (HIGH confidence)
- PyPI pgvector — version 0.4.2 confirmed current April 2026 (HIGH confidence)
- PyPI tenacity — version 9.1.4 released 2026-02-07 (HIGH confidence)
- PyPI openai — version 2.30.0 released 2026-03-25 (HIGH confidence)
- PyPI anthropic — version 0.89.0 released 2026-04-03 (HIGH confidence)
- Docker SDK docs (docker-py.readthedocs.io) — version 7.1.0 (HIGH confidence)
- WebSearch: ARQ vs Celery vs Dramatiq async FastAPI 2025 comparison — ARQ recommendation for async-native Redis queue (MEDIUM confidence — no direct PyPI version verification)
- WebSearch: structlog FastAPI production 2025-2026 — structlog async-safe via contextvars confirmed (MEDIUM confidence)
- WebSearch: pgvector asyncpg async support — `pgvector.asyncpg` module confirmed (HIGH confidence)
- WebSearch: nsjail vs Docker vs gVisor sandbox comparison 2025 — Docker recommendation for cross-platform portability (MEDIUM confidence)

---

*Stack research for: Zeroth v1.1 Production Readiness*
*Researched: 2026-04-06*
