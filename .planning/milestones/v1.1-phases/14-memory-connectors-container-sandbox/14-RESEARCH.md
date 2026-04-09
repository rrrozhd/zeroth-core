# Phase 14: Memory Connectors & Container Sandbox - Research

**Researched:** 2026-04-07
**Domain:** Persistent memory backends (Redis, pgvector, ChromaDB, Elasticsearch) + Docker sidecar sandbox architecture
**Confidence:** HIGH

## Summary

Phase 14 has two distinct workstreams: (1) rewriting Zeroth's memory connector system to implement GovernAI v0.3.0's async `MemoryConnector` protocol with five new persistent backends, and (2) adding a Docker sidecar architecture for untrusted execution. Both workstreams have clear, well-defined interfaces to implement against.

The GovernAI `MemoryConnector` protocol is a simple 4-method async interface (`read`, `write`, `delete`, `search`) parameterized by `MemoryScope` and `target`. The existing Zeroth connectors are sync and use a different signature (`MemoryContext` + `key`). The rewrite replaces both the protocol and the existing in-memory implementations, then adds Redis KV, Redis thread, pgvector, ChromaDB, and Elasticsearch backends. Each connector is wrapped with GovernAI's `ScopedMemoryConnector` and `AuditingMemoryConnector` at resolution time.

The sidecar workstream adds a new `SIDECAR` backend mode to `SandboxManager` that communicates with a separate FastAPI container over HTTP. This container holds the Docker socket and creates isolated containers for untrusted code. The API container never touches the Docker socket.

**Primary recommendation:** Implement connectors in dependency order -- rewrite core protocol first, then existing in-memory connectors, then Redis pair, then vector backends in parallel. Sidecar is independent and can be built concurrently.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Rewrite Zeroth's `MemoryConnector` protocol to match GovernAI's async `MemoryConnector` interface -- `read(key, scope, target)`, `write(key, value, scope, target)`, `delete(key, scope, target)`, `search(query, scope, target)` returning `MemoryEntry`
- D-02: All new connectors implement GovernAI's `MemoryConnector` protocol directly -- no Zeroth-specific intermediate protocol
- D-03: Existing `MemoryContext` replaced by GovernAI's `MemoryScope` enum + `target` parameter pattern -- scope resolution moves to `ScopedMemoryConnector` wrapper
- D-04: Every connector wrapped with `ScopedMemoryConnector` (auto-fills scope targets from execution context) and `AuditingMemoryConnector` (emits audit events) at resolution time in `MemoryConnectorResolver`
- D-05: Existing in-memory connectors rewritten to implement GovernAI protocol -- retained for dev/test use
- D-06: Two separate Redis connectors: `RedisKVMemoryConnector` (MEM-01) and `RedisThreadMemoryConnector` (MEM-02)
- D-07: Both Redis connectors reuse existing `RedisConfig` from `src/zeroth/storage/redis.py`
- D-08: Redis KV uses simple GET/SET/DEL with JSON serialization; Redis thread uses sorted sets or lists for ordered message history
- D-09: Data persists across process restarts
- D-10: Direct per-backend implementations -- no common vector abstraction layer
- D-11: `PgvectorMemoryConnector` uses existing async Postgres connection from Phase 11, separate table(s) with pgvector extension
- D-12: `ChromaDBMemoryConnector` connects to external ChromaDB server (not embedded)
- D-13: `ElasticsearchMemoryConnector` uses official elasticsearch-py async client
- D-14: Embedding generation is connector-internal -- each connector handles its own embedding strategy via connector-specific config
- D-15: New pydantic-settings sub-models in `ZerothSettings`: `RedisMemorySettings`, `PgvectorSettings`, `ChromaSettings`, `ElasticsearchSettings`
- D-16: Agent nodes reference connector by type name string (e.g., `memory: redis_kv`)
- D-17: Connector instances are singletons per type -- created at bootstrap, shared across agent nodes
- D-18: Sandbox sidecar runs as separate container with Docker socket mounted -- API container NEVER mounts Docker socket
- D-19: Sidecar exposes REST API for sandbox operations: execute, status, cancel
- D-20: `SandboxManager` gains new `SIDECAR` backend mode alongside existing `LOCAL` and `DOCKER`
- D-21: Sidecar enforces resource limits using existing `ResourceConstraints` and `build_docker_resource_flags()` logic
- D-22: Sidecar is lightweight FastAPI service in `src/zeroth/sandbox_sidecar/` package
- D-23: Network isolation: untrusted containers have no host network access -- sidecar creates isolated Docker network per execution
- D-24: Unit tests mock external backends
- D-25: Integration tests use testcontainers-python gated behind `@pytest.mark.live`
- D-26: ChromaDB integration test uses real ChromaDB container via testcontainers
- D-27: Sidecar integration tests verify end-to-end sandboxed execution via HTTP API

### Claude's Discretion
- Redis data structure choices for thread memory (sorted set vs list vs stream)
- pgvector table schema and index type (IVFFlat vs HNSW)
- Embedding model selection for vector connectors
- Sidecar REST API exact routes and request/response schemas
- ChromaDB collection naming and metadata conventions
- Elasticsearch index mapping and analyzer configuration
- Connection pool sizes and timeout defaults for each backend
- Whether sidecar needs authentication (internal network may be sufficient)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MEM-01 | Redis-backed key-value memory connector replacing in-memory dict | RedisKVMemoryConnector using redis-py async with GET/SET/DEL + JSON serialization |
| MEM-02 | Redis-backed conversation/thread memory connector replacing in-memory store | RedisThreadMemoryConnector using Redis sorted sets for ordered message history |
| MEM-03 | pgvector-backed semantic memory connector for agent context retrieval | PgvectorMemoryConnector using pgvector 0.4.2 + psycopg3 async + HNSW index |
| MEM-04 | ChromaDB memory connector for vector similarity search | ChromaDBMemoryConnector using chromadb-client 1.5.6 HTTP client |
| MEM-05 | Elasticsearch memory connector for full-text and hybrid search | ElasticsearchMemoryConnector using elasticsearch[async] 9.3.0 |
| MEM-06 | Zeroth memory connectors bridged to GovernAI v0.3.0 ScopedMemoryConnector and AuditingMemoryConnector | All connectors implement GovernAI MemoryConnector protocol; wrapped at resolution time |
| SBX-01 | Docker-based sandbox backend for untrusted executable units with resource limits and network isolation | SIDECAR mode in SandboxManager + SandboxSidecarClient HTTP client |
| SBX-02 | Sandbox sidecar architecture prevents Docker socket exposure on API container | Separate FastAPI sidecar service holds Docker socket; API communicates over internal network |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| redis | 7.3.0 | Redis async client for KV and thread connectors | Already in pyproject.toml; supports async natively |
| psycopg | 3.3.3 | Postgres async driver for pgvector connector | Already in pyproject.toml; D-11 reuses Phase 11 connection |
| fastapi | 0.135.1 | Sidecar REST API framework | Already in pyproject.toml; consistent with main API |
| httpx | 0.28.1 | HTTP client for sidecar communication + ChromaDB | Already in pyproject.toml; async-native |
| pydantic-settings | installed | Config sub-models for each backend | Already in pyproject.toml; D-15 pattern |

### New Dependencies (must add to pyproject.toml)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| pgvector | 0.4.2 | Vector type registration for psycopg3 | PgvectorMemoryConnector -- `register_vector_async` |
| chromadb-client | 1.5.6 | HTTP-only ChromaDB client (no embedded server) | ChromaDBMemoryConnector -- lightweight, no heavy deps |
| elasticsearch[async] | 9.3.0 | Async Elasticsearch client with aiohttp transport | ElasticsearchMemoryConnector |

### Test Dependencies (must add to dev group)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| testcontainers[redis] | >=4.14 | Redis integration tests | Already have testcontainers[postgres]; add redis extra |
| testcontainers[elasticsearch] | >=4.14 | Elasticsearch integration tests | Add elasticsearch extra |
| testcontainers[chroma] | >=4.14 | ChromaDB integration tests | Add chroma extra |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| chromadb-client | chromadb (full) | Full package pulls in heavy deps (onnxruntime, etc); client-only is correct per D-12 |
| elasticsearch 9.x | elasticsearch 8.x | 9.x is current; 8.x still works but would need `elasticsearch8` package name |
| Redis sorted sets for thread | Redis Streams | Streams add consumer group complexity; sorted sets are simpler for ordered history |
| HNSW index (pgvector) | IVFFlat index | HNSW has better recall and no training step; IVFFlat is faster to build but lower quality |

**Installation:**
```bash
uv add pgvector chromadb-client "elasticsearch[async]>=9.0,<10"
uv add --group dev "testcontainers[redis,elasticsearch,chroma]>=4.14"
```

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/
  memory/
    connectors.py          # Rewritten: GovernAI-protocol in-memory connectors
    models.py              # Updated: Remove MemoryContext, adapt to GovernAI models
    registry.py            # Updated: MemoryConnectorResolver wraps with Scoped+Auditing
    redis_kv.py            # NEW: RedisKVMemoryConnector
    redis_thread.py        # NEW: RedisThreadMemoryConnector
    pgvector_connector.py  # NEW: PgvectorMemoryConnector
    chroma_connector.py    # NEW: ChromaDBMemoryConnector
    elastic_connector.py   # NEW: ElasticsearchMemoryConnector
  config/
    settings.py            # Updated: Add memory backend sub-models
  execution_units/
    sandbox.py             # Updated: Add SIDECAR to SandboxBackendMode
    sidecar_client.py      # NEW: SandboxSidecarClient (httpx async)
  sandbox_sidecar/
    __init__.py
    app.py                 # NEW: FastAPI sidecar application
    executor.py            # NEW: Docker execution logic (reuses build_docker_resource_flags)
    models.py              # NEW: Request/response schemas
tests/
  memory/
    test_connectors.py     # Updated: Tests for rewritten in-memory connectors
    test_redis_kv.py       # NEW: Unit tests (mocked) + integration tests
    test_redis_thread.py   # NEW: Unit tests (mocked) + integration tests
    test_pgvector.py       # NEW: Unit tests (mocked) + integration tests
    test_chroma.py         # NEW: Unit tests (mocked) + integration tests
    test_elastic.py        # NEW: Unit tests (mocked) + integration tests
    test_resolver.py       # NEW: Resolver wrapping with Scoped+Auditing
  execution_units/
    test_sidecar_client.py # NEW: Unit tests for sidecar HTTP client
  sandbox_sidecar/
    test_app.py            # NEW: Sidecar API unit tests
    test_executor.py       # NEW: Sidecar execution logic tests
```

### Pattern 1: GovernAI MemoryConnector Implementation
**What:** Each connector implements the 4-method async protocol from GovernAI
**When to use:** Every new memory backend
**Example:**
```python
# Source: GovernAI memory/connector.py protocol + dict_connector.py reference
from governai.memory.models import MemoryEntry, MemoryScope
from governai.models.common import JSONValue

class RedisKVMemoryConnector:
    """Redis-backed key-value memory connector."""

    def __init__(self, redis_client: redis.asyncio.Redis, *, key_prefix: str = "zeroth:mem:kv") -> None:
        self._redis = redis_client
        self._prefix = key_prefix

    def _key(self, key: str, scope: MemoryScope, target: str | None) -> str:
        return f"{self._prefix}:{scope.value}:{target or ''}:{key}"

    async def read(self, key: str, scope: MemoryScope, *, target: str | None = None) -> MemoryEntry | None:
        raw = await self._redis.get(self._key(key, scope, target))
        if raw is None:
            return None
        return MemoryEntry.model_validate_json(raw)

    async def write(self, key: str, value: JSONValue, scope: MemoryScope, *, target: str | None = None) -> None:
        entry = MemoryEntry(key=key, value=value, scope=scope, scope_target=target or "")
        await self._redis.set(self._key(key, scope, target), entry.model_dump_json())

    async def delete(self, key: str, scope: MemoryScope, *, target: str | None = None) -> None:
        deleted = await self._redis.delete(self._key(key, scope, target))
        if not deleted:
            raise KeyError(key)

    async def search(self, query: dict, scope: MemoryScope, *, target: str | None = None) -> list[MemoryEntry]:
        # KV connector: scan keys matching prefix pattern, filter by text
        pattern = f"{self._prefix}:{scope.value}:{target or ''}:*"
        results = []
        async for redis_key in self._redis.scan_iter(match=pattern):
            raw = await self._redis.get(redis_key)
            if raw:
                entry = MemoryEntry.model_validate_json(raw)
                text = query.get("text", "").lower()
                if not text or text in entry.key.lower() or text in str(entry.value).lower():
                    results.append(entry)
        return results
```

### Pattern 2: Resolver Wrapping with ScopedMemoryConnector + AuditingMemoryConnector
**What:** At resolution time, wrap raw connector with GovernAI wrappers
**When to use:** In `MemoryConnectorResolver.resolve()`
**Example:**
```python
# Source: GovernAI scoped.py + auditing.py
from governai.memory import ScopedMemoryConnector, AuditingMemoryConnector

# In MemoryConnectorResolver.resolve():
raw_connector = self._connector_factory(connector_type)  # e.g., RedisKVMemoryConnector
auditing = AuditingMemoryConnector(
    raw_connector,
    emitter=self._audit_emitter,
    run_id=run_id,
    thread_id=thread_id,
    workflow_name=workflow_name,
)
scoped = ScopedMemoryConnector(
    auditing,
    run_id=run_id,
    thread_id=thread_id,
    workflow_name=workflow_name,
)
# scoped is what the agent receives
```

### Pattern 3: Redis Thread Memory with Sorted Sets
**What:** Store ordered conversation messages using Redis sorted sets (score = timestamp)
**When to use:** RedisThreadMemoryConnector
**Example:**
```python
# Thread memory: write appends to sorted set, read returns latest N entries
async def write(self, key: str, value: JSONValue, scope: MemoryScope, *, target: str | None = None) -> None:
    sorted_key = f"{self._prefix}:{scope.value}:{target or ''}:{key}"
    entry = MemoryEntry(key=key, value=value, scope=scope, scope_target=target or "")
    score = datetime.now(timezone.utc).timestamp()
    await self._redis.zadd(sorted_key, {entry.model_dump_json(): score})

async def read(self, key: str, scope: MemoryScope, *, target: str | None = None) -> MemoryEntry | None:
    sorted_key = f"{self._prefix}:{scope.value}:{target or ''}:{key}"
    # Return most recent entry
    items = await self._redis.zrevrange(sorted_key, 0, 0)
    if not items:
        return None
    return MemoryEntry.model_validate_json(items[0])
```

### Pattern 4: pgvector Connector with psycopg3 Async
**What:** Semantic search using pgvector extension in existing Postgres database
**When to use:** PgvectorMemoryConnector
**Example:**
```python
# Source: pgvector-python README (psycopg3 async section)
from pgvector.psycopg import register_vector_async
import numpy as np

# At initialization:
async def _ensure_setup(self, conn) -> None:
    await register_vector_async(conn)
    await conn.execute("CREATE EXTENSION IF NOT EXISTS vector")
    await conn.execute("""
        CREATE TABLE IF NOT EXISTS zeroth_memory_vectors (
            id SERIAL PRIMARY KEY,
            key TEXT NOT NULL,
            scope TEXT NOT NULL,
            scope_target TEXT NOT NULL,
            value JSONB NOT NULL,
            embedding vector(%s) NOT NULL,
            metadata JSONB DEFAULT '{}',
            created_at TIMESTAMPTZ DEFAULT NOW(),
            updated_at TIMESTAMPTZ DEFAULT NOW(),
            UNIQUE(key, scope, scope_target)
        )
    """, [self._dimensions])
    await conn.execute("""
        CREATE INDEX IF NOT EXISTS idx_memory_vectors_embedding
        ON zeroth_memory_vectors USING hnsw (embedding vector_cosine_ops)
    """)

# Search (semantic):
async def search(self, query: dict, scope: MemoryScope, *, target: str | None = None) -> list[MemoryEntry]:
    embedding = await self._embed(query.get("text", ""))
    rows = await conn.execute(
        "SELECT * FROM zeroth_memory_vectors WHERE scope = %s AND scope_target = %s ORDER BY embedding <=> %s LIMIT %s",
        [scope.value, target or "", embedding, query.get("limit", 10)]
    )
    return [self._row_to_entry(row) for row in rows]
```

### Pattern 5: Sidecar HTTP Client
**What:** API container communicates with sidecar over internal Docker network
**When to use:** SandboxSidecarClient for SIDECAR backend mode
**Example:**
```python
class SandboxSidecarClient:
    """HTTP client for the sandbox sidecar service."""

    def __init__(self, base_url: str, *, timeout: float = 30.0) -> None:
        self._client = httpx.AsyncClient(base_url=base_url, timeout=timeout)

    async def execute(self, request: SidecarExecuteRequest) -> SidecarExecuteResponse:
        resp = await self._client.post("/execute", content=request.model_dump_json())
        resp.raise_for_status()
        return SidecarExecuteResponse.model_validate_json(resp.content)

    async def status(self, execution_id: str) -> SidecarStatusResponse:
        resp = await self._client.get(f"/executions/{execution_id}")
        resp.raise_for_status()
        return SidecarStatusResponse.model_validate_json(resp.content)

    async def cancel(self, execution_id: str) -> None:
        resp = await self._client.post(f"/executions/{execution_id}/cancel")
        resp.raise_for_status()
```

### Anti-Patterns to Avoid
- **Embedding model hardcoded in connector:** Make embedding model/dimensions configurable per connector instance via settings, not hardcoded. Default to a sensible choice (e.g., OpenAI text-embedding-3-small, 1536 dims) but allow override.
- **Sharing Redis keys across connector types:** Use distinct key prefixes (`zeroth:mem:kv:`, `zeroth:mem:thread:`, etc.) to prevent collision between KV and thread connectors sharing the same Redis instance.
- **Blocking calls in async connectors:** All external I/O must be async. redis-py has native async (`redis.asyncio`), psycopg3 has async support, elasticsearch has `AsyncElasticsearch`, and chromadb-client uses httpx under the hood.
- **Mounting Docker socket on API container:** This is explicitly forbidden by SBX-02. The sidecar holds the socket; the API talks to sidecar over HTTP.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Scope/target resolution | Custom scope logic per connector | GovernAI `ScopedMemoryConnector` | Already handles RUN/THREAD/SHARED resolution with target fallback |
| Audit event emission | Custom audit logging per connector | GovernAI `AuditingMemoryConnector` | Wraps any connector; emits consistent events for read/write/delete/search |
| Vector type registration | Manual SQL casting for vectors | `pgvector` package (`register_vector_async`) | Handles VECTOR, HALFVEC, SPARSEVEC type registration with psycopg3 |
| Docker resource flags | Custom flag building for sidecar | Reuse `build_docker_resource_flags()` from constraints.py | Already translates ResourceConstraints to --cpus, --memory, --pids-limit, --network flags |
| Redis connection management | Custom connection pooling | `redis.asyncio.Redis.from_url()` with connection pool | redis-py handles pooling, reconnection, health checks |
| Elasticsearch async transport | Custom aiohttp setup | `elasticsearch[async]` extra | Installs aiohttp transport; AsyncElasticsearch handles connection pooling |

**Key insight:** GovernAI provides the wrapper stack (Scoped + Auditing) and the protocol. Zeroth connectors only need to implement the 4 raw methods. All cross-cutting concerns are handled by the wrapper chain.

## Common Pitfalls

### Pitfall 1: MemoryContext Removal Breaking AgentRunner
**What goes wrong:** The existing `AgentRunner._load_memory()` and `_store_memory()` methods build `MemoryContext` objects and call `connector.read(context, key)`. After rewrite, the signature is `connector.read(key, scope, target=...)`.
**Why it happens:** The rewrite changes the protocol signature fundamentally.
**How to avoid:** Update `MemoryConnectorResolver` to return wrapped connectors (Scoped+Auditing) that the runner calls with the new GovernAI signature. The resolver becomes the adapter layer.
**Warning signs:** `TypeError: read() got an unexpected keyword argument 'scope'` or `read() missing required argument: 'context'`

### Pitfall 2: Redis Key Collision Between KV and Thread Connectors
**What goes wrong:** Both connectors share the same Redis instance but use overlapping key patterns, causing data corruption.
**Why it happens:** Forgetting to use distinct prefixes when both connector types are configured against the same Redis.
**How to avoid:** Use distinct prefixes: `zeroth:mem:kv:{scope}:{target}:{key}` vs `zeroth:mem:thread:{scope}:{target}:{key}`. The prefix is part of the connector, not the config.
**Warning signs:** Reading a KV entry returns thread data or vice versa.

### Pitfall 3: pgvector Extension Not Installed
**What goes wrong:** `CREATE EXTENSION vector` fails because the extension isn't available in the Postgres instance.
**Why it happens:** Standard Postgres doesn't include pgvector; it needs to be installed separately or use a Postgres image with pgvector pre-installed (e.g., `pgvector/pgvector:pg16`).
**How to avoid:** Use `ankane/pgvector` Docker image for testcontainers. For production, document the requirement. Add a startup check that tests for extension availability.
**Warning signs:** `ERROR: extension "vector" is not available`

### Pitfall 4: ChromaDB Client vs Server Version Mismatch
**What goes wrong:** `chromadb-client` version doesn't match the ChromaDB server version, causing API incompatibilities.
**Why it happens:** ChromaDB server and client must be compatible versions.
**How to avoid:** Pin `chromadb-client` version and document the required server version. In testcontainers, use a matching image version.
**Warning signs:** HTTP 400/500 errors from ChromaDB with schema mismatch messages.

### Pitfall 5: Sidecar Network Isolation Incomplete
**What goes wrong:** Untrusted containers can still reach the host network or other containers.
**Why it happens:** Docker's default bridge network allows inter-container communication.
**How to avoid:** Sidecar must create an isolated Docker network per execution (`docker network create --internal`), connect the untrusted container to it, and remove the network after execution.
**Warning signs:** Untrusted container can `curl` external URLs or reach other containers.

### Pitfall 6: Embedding Dimension Mismatch
**What goes wrong:** Vector search returns no results or errors because stored embeddings have different dimensions than query embeddings.
**Why it happens:** Changing the embedding model after data is stored, or misconfiguring dimensions.
**How to avoid:** Store embedding model name and dimensions in connector config. Validate dimensions match at startup. Include model info in metadata.
**Warning signs:** pgvector error about vector dimension mismatch; ChromaDB silently returns empty results.

## Code Examples

### GovernAI MemoryConnector Protocol (from installed package)
```python
# Source: .venv/lib/python3.12/site-packages/governai/memory/connector.py
@runtime_checkable
class MemoryConnector(Protocol):
    async def read(self, key: str, scope: MemoryScope, *, target: str | None = None) -> MemoryEntry | None: ...
    async def write(self, key: str, value: JSONValue, scope: MemoryScope, *, target: str | None = None) -> None: ...
    async def delete(self, key: str, scope: MemoryScope, *, target: str | None = None) -> None: ...
    async def search(self, query: dict, scope: MemoryScope, *, target: str | None = None) -> list[MemoryEntry]: ...
```

### GovernAI MemoryEntry Model (from installed package)
```python
# Source: .venv/lib/python3.12/site-packages/governai/memory/models.py
class MemoryScope(str, Enum):
    RUN = "run"
    THREAD = "thread"
    SHARED = "shared"

class MemoryEntry(BaseModel):
    key: str
    value: JSONValue
    scope: MemoryScope
    scope_target: str
    created_at: datetime = Field(default_factory=_utcnow)
    updated_at: datetime = Field(default_factory=_utcnow)
    metadata: dict[str, Any] = Field(default_factory=dict)
```

### DictMemoryConnector Reference Implementation (from installed package)
```python
# Source: .venv/lib/python3.12/site-packages/governai/memory/dict_connector.py
# Storage layout: _store[scope.value][target][key] = MemoryEntry
# This is the canonical reference for how all connectors should behave
```

### Existing Zeroth MemoryConnector (to be rewritten)
```python
# Source: src/zeroth/memory/connectors.py (CURRENT -- will be replaced)
class MemoryConnector(Protocol):
    connector_type: str
    def read(self, context: MemoryContext, key: str) -> Any | None: ...
    def write(self, context: MemoryContext, key: str, value: Any) -> None: ...
```

### ScopedMemoryConnector Wrapping (from installed package)
```python
# Source: .venv/lib/python3.12/site-packages/governai/memory/scoped.py
# Constructor: ScopedMemoryConnector(connector, run_id=..., thread_id=..., workflow_name=...)
# Resolves target: RUN -> run_id, THREAD -> thread_id or run_id, SHARED -> "__shared__"
```

### AuditingMemoryConnector Wrapping (from installed package)
```python
# Source: .venv/lib/python3.12/site-packages/governai/memory/auditing.py
# Constructor: AuditingMemoryConnector(inner, emitter, run_id=..., thread_id=..., workflow_name=...)
# Emits: MEMORY_READ, MEMORY_WRITE, MEMORY_DELETE, MEMORY_SEARCH events
# CRITICAL: payload must NOT contain "value" on writes (D-15 in GovernAI)
```

### Existing SandboxBackendMode (to be extended)
```python
# Source: src/zeroth/execution_units/sandbox.py
class SandboxBackendMode(StrEnum):
    LOCAL = "local"
    DOCKER = "docker"
    AUTO = "auto"
    # ADD: SIDECAR = "sidecar"
```

### Config Sub-Model Pattern (from existing settings.py)
```python
# Source: src/zeroth/config/settings.py
class RedisSettings(BaseModel):
    mode: str = "local"
    host: str = "127.0.0.1"
    port: int = 6379
    # ... follow this pattern for new memory backend settings
```

## Discretion Recommendations

### Redis Thread Data Structure: Sorted Sets
**Recommendation:** Use Redis sorted sets (ZADD/ZRANGE) with timestamp scores.
**Rationale:** Sorted sets provide natural time-ordered retrieval with O(log N) insert and O(log N + M) range queries. Lists would work for append-only but sorted sets allow deduplication and range queries by time. Streams add consumer group complexity not needed here.

### pgvector Index Type: HNSW
**Recommendation:** Use HNSW index (`vector_cosine_ops`).
**Rationale:** HNSW provides better recall than IVFFlat without requiring a training step. IVFFlat needs `CREATE INDEX ... WITH (lists = N)` tuning and periodic reindexing. HNSW is the default recommendation for pgvector since v0.5.0. Tradeoff: HNSW uses more memory and is slower to build, but for a memory connector with moderate data size this is acceptable.

### Embedding Model: Configurable, Default to OpenAI text-embedding-3-small
**Recommendation:** Make embedding model configurable per connector. Default to `text-embedding-3-small` (1536 dimensions) since the project already depends on OpenAI via litellm.
**Rationale:** Connector config should include `embedding_model: str` and `embedding_dimensions: int`. The connector calls litellm for embedding generation, which supports all providers. This avoids adding a direct OpenAI dependency.

### Sidecar REST API Routes
**Recommendation:**
```
POST   /execute              -- Submit execution request, returns execution_id
GET    /executions/{id}      -- Poll execution status and result
POST   /executions/{id}/cancel -- Cancel running execution
GET    /health               -- Health check
```
**Rationale:** Simple REST with polling. WebSocket streaming is unnecessary complexity for MVP. The execute endpoint returns immediately with an execution_id; the caller polls for completion.

### Sidecar Authentication: None (Internal Network Only)
**Recommendation:** No authentication for MVP. The sidecar is on an internal Docker network not exposed to the host.
**Rationale:** Adding auth adds complexity. The sidecar is only reachable from the API container over the internal Docker network. For production hardening, mTLS or a shared secret can be added later.

### Connection Pool Defaults
**Recommendation:**
- Redis: Use default redis-py pool (max 10 connections)
- Postgres/pgvector: Reuse Phase 11 psycopg pool (min 2, max 10 from DatabaseSettings)
- ChromaDB: httpx client default pool (max 20 connections)
- Elasticsearch: Default pool from AsyncElasticsearch (max 10)

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Docker | Sidecar, integration tests | Not found on dev machine | -- | Unit tests mock Docker; integration tests skip via @pytest.mark.live |
| Redis server | Redis connector integration tests | Not found on dev machine | -- | Unit tests mock redis; integration tests use testcontainers |
| Postgres server | pgvector integration tests | Not found on dev machine | -- | Unit tests mock; integration tests use testcontainers |
| pgvector extension | pgvector connector | N/A (extension in DB) | -- | testcontainers uses pgvector Docker image |
| ChromaDB server | ChromaDB integration tests | Not found on dev machine | -- | Unit tests mock; testcontainers for integration |
| Elasticsearch server | ES integration tests | Not found on dev machine | -- | Unit tests mock; testcontainers for integration |

**Missing dependencies with no fallback:**
- None -- all external services are only needed for integration tests (gated behind `@pytest.mark.live`).

**Missing dependencies with fallback:**
- All external services (Docker, Redis, Postgres, ChromaDB, Elasticsearch) are only needed for `@pytest.mark.live` integration tests. Unit tests mock all backends. This is consistent with Phase 11/12 patterns.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Zeroth sync MemoryConnector protocol | GovernAI async MemoryConnector protocol | Phase 14 | All connectors become async; signature changes from (context, key) to (key, scope, target) |
| MemoryContext for scope resolution | GovernAI MemoryScope enum + ScopedMemoryConnector wrapper | Phase 14 | Scope resolution moves from caller to wrapper |
| In-process dict storage only | External persistent backends | Phase 14 | Data survives process restarts |
| Docker socket on API container | Sidecar architecture | Phase 14 | Security improvement: API never touches Docker socket |
| elasticsearch-async package | elasticsearch[async] (built into main package since 7.8) | 2020+ | Use AsyncElasticsearch from main elasticsearch package |
| chromadb (full, embedded) | chromadb-client (HTTP only) | ChromaDB 1.x | Lighter dependency; connects to external server |
| pgvector IVFFlat default | pgvector HNSW default | pgvector 0.5.0+ | Better recall without training step |

## Open Questions

1. **Embedding generation latency budget**
   - What we know: Each `write()` and `search()` on vector connectors requires embedding generation (API call to LLM provider)
   - What's unclear: Acceptable latency for embedding calls in the critical path of agent execution
   - Recommendation: Use litellm for embedding (already in deps). Consider caching embeddings for identical inputs. Document that vector write/search adds ~100-500ms per embedding call.

2. **pgvector migration strategy**
   - What we know: Phase 11 uses Alembic for Postgres migrations. pgvector needs `CREATE EXTENSION vector` and new tables.
   - What's unclear: Whether to use Alembic migration or connector-level auto-setup
   - Recommendation: Use Alembic migration for the extension and table creation. This is consistent with Phase 11 and ensures schema is versioned.

3. **Sidecar container image**
   - What we know: Sidecar is a FastAPI service that needs Docker CLI access
   - What's unclear: Base image and packaging strategy
   - Recommendation: Use `python:3.12-slim` base with Docker CLI installed. Package as separate Dockerfile in the repo. The sidecar has minimal dependencies (fastapi, uvicorn, docker CLI).

## Project Constraints (from CLAUDE.md)

- **Build/Test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` for main package, `tests/` for pytest tests
- **Progress logging:** Must use `progress-logger` skill for every implementation session
- **Test pattern:** `@pytest.mark.live` for integration tests requiring external services; `addopts = "-m 'not live'"` excludes them by default
- **Async pattern:** All new code must be async (Phase 11 established this)
- **Config pattern:** Pydantic-settings sub-models in `ZerothSettings`
- **Dependency pinning:** GovernAI pinned to git commit `7452de4`

## Sources

### Primary (HIGH confidence)
- GovernAI memory module (installed package) -- `MemoryConnector` protocol, `MemoryEntry`, `MemoryScope`, `ScopedMemoryConnector`, `AuditingMemoryConnector`, `DictMemoryConnector`
- Zeroth source code -- `memory/connectors.py`, `memory/models.py`, `memory/registry.py`, `storage/redis.py`, `execution_units/sandbox.py`, `execution_units/constraints.py`, `config/settings.py`
- Phase 14 CONTEXT.md -- All locked decisions D-01 through D-27

### Secondary (MEDIUM confidence)
- [pgvector-python GitHub](https://github.com/pgvector/pgvector-python) -- psycopg3 async patterns, HNSW index usage
- [chromadb-client PyPI](https://pypi.org/project/chromadb-client/) -- v1.5.6, HTTP-only client
- [elasticsearch PyPI](https://pypi.org/project/elasticsearch/) -- v9.3.0, AsyncElasticsearch built-in
- [pgvector PyPI](https://pypi.org/project/pgvector/) -- v0.4.2
- [testcontainers-python](https://pypi.org/project/testcontainers/) -- v4.14+, supports redis/elasticsearch/chroma extras

### Tertiary (LOW confidence)
- None -- all findings verified against installed packages or official PyPI.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- redis already installed and working; pgvector/chromadb-client/elasticsearch verified on PyPI with current versions
- Architecture: HIGH -- GovernAI protocol is installed and inspected; existing Zeroth patterns are well-understood from source
- Pitfalls: HIGH -- derived from concrete code analysis of signature changes and infrastructure requirements

**Research date:** 2026-04-07
**Valid until:** 2026-05-07 (stable domain, 30 days)
