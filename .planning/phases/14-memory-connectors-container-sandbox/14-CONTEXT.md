# Phase 14: Memory Connectors & Container Sandbox - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 14 delivers persistent external memory backends (Redis key-value, Redis conversation/thread, pgvector semantic, ChromaDB vector, Elasticsearch full-text) bridged to GovernAI v0.3.0 memory protocol (ScopedMemoryConnector, AuditingMemoryConnector), and hardens untrusted execution unit sandboxing via a Docker sidecar architecture that prevents Docker socket exposure on the API container.

</domain>

<decisions>
## Implementation Decisions

### GovernAI Memory Bridge
- **D-01:** Rewrite Zeroth's `MemoryConnector` protocol to match GovernAI's async `MemoryConnector` interface — `read(key, scope, target)`, `write(key, value, scope, target)`, `delete(key, scope, target)`, `search(query, scope, target)` returning `MemoryEntry`
- **D-02:** All new connectors implement GovernAI's `MemoryConnector` protocol directly — no Zeroth-specific intermediate protocol
- **D-03:** Existing `MemoryContext` replaced by GovernAI's `MemoryScope` enum + `target` parameter pattern — scope resolution moves to `ScopedMemoryConnector` wrapper
- **D-04:** Every connector wrapped with `ScopedMemoryConnector` (auto-fills scope targets from execution context) and `AuditingMemoryConnector` (emits audit events) at resolution time in `MemoryConnectorResolver`
- **D-05:** Existing in-memory connectors (`RunEphemeralMemoryConnector`, `KeyValueMemoryConnector`, `ThreadMemoryConnector`) rewritten to implement GovernAI protocol — retained for dev/test use

### Redis Connectors
- **D-06:** Two separate Redis connectors: `RedisKVMemoryConnector` (MEM-01) for key-value state and `RedisThreadMemoryConnector` (MEM-02) for conversation history — mirrors existing in-memory connector split
- **D-07:** Both Redis connectors reuse the existing `RedisConfig` from `src/zeroth/storage/redis.py` for connection management
- **D-08:** Redis KV uses simple GET/SET/DEL with JSON serialization; Redis thread uses sorted sets or lists for ordered message history
- **D-09:** Data persists across process restarts — this is the key differentiator from in-memory connectors

### Vector Store Connectors
- **D-10:** Direct per-backend implementations — no common vector abstraction layer. Each backend (pgvector, ChromaDB, Elasticsearch) implements GovernAI `MemoryConnector` directly
- **D-11:** `PgvectorMemoryConnector` (MEM-03) uses the existing async Postgres connection from Phase 11 — shares the same database, separate table(s) with pgvector extension
- **D-12:** `ChromaDBMemoryConnector` (MEM-04) connects to an external ChromaDB server (not embedded) — production pattern consistent with sidecar philosophy
- **D-13:** `ElasticsearchMemoryConnector` (MEM-05) uses the official elasticsearch-py async client for full-text and hybrid search
- **D-14:** Embedding generation is connector-internal — each connector handles its own embedding strategy (model, dimensions) via connector-specific config

### Connector Configuration
- **D-15:** New pydantic-settings sub-models in `ZerothSettings`: `RedisMemorySettings`, `PgvectorSettings`, `ChromaSettings`, `ElasticsearchSettings` — follows Phase 11 config pattern
- **D-16:** Agent nodes reference connector by type name string (e.g., `memory: redis_kv`, `memory: pgvector`) — `MemoryConnectorResolver` maps type names to configured connector instances
- **D-17:** Connector instances are singletons per type (one Redis connection pool, one Chroma client, etc.) — created at bootstrap, shared across agent nodes

### Container Sandbox Sidecar
- **D-18:** Sandbox sidecar runs as a separate container with the Docker socket mounted — the API container NEVER mounts the Docker socket (SBX-02)
- **D-19:** Sidecar exposes a REST API for sandbox operations: execute, status, cancel — API container calls sidecar over internal Docker network
- **D-20:** `SandboxManager` gains a new `SIDECAR` backend mode alongside existing `LOCAL` and `DOCKER` — selected via config, transparent to callers
- **D-21:** Sidecar enforces resource limits (CPU, memory, network isolation, PID limits) on untrusted containers using existing `ResourceConstraints` and `build_docker_resource_flags()` logic
- **D-22:** Sidecar is a lightweight FastAPI service (minimal dependencies) — lives in a new `src/zeroth/sandbox_sidecar/` package or a separate directory
- **D-23:** Network isolation: untrusted containers have no host network access — sidecar creates an isolated Docker network per execution

### Testing
- **D-24:** Unit tests mock external backends (Redis, Chroma, Elasticsearch) — validates connector logic without running services
- **D-25:** Integration tests use testcontainers-python (Redis, Postgres+pgvector, Elasticsearch) gated behind `@pytest.mark.live` — consistent with Phase 11/12 pattern
- **D-26:** ChromaDB integration test uses a real ChromaDB container via testcontainers
- **D-27:** Sidecar integration tests verify end-to-end sandboxed execution via HTTP API — gated behind `@pytest.mark.live`

### Claude's Discretion
- Redis data structure choices for thread memory (sorted set vs list vs stream)
- pgvector table schema and index type (IVFFlat vs HNSW)
- Embedding model selection for vector connectors (can default to OpenAI text-embedding-3-small or make configurable)
- Sidecar REST API exact routes and request/response schemas
- ChromaDB collection naming and metadata conventions
- Elasticsearch index mapping and analyzer configuration
- Connection pool sizes and timeout defaults for each backend
- Whether sidecar needs authentication (internal network may be sufficient)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Memory System (existing Zeroth)
- `src/zeroth/memory/connectors.py` — Current `MemoryConnector` protocol (sync), `RunEphemeralMemoryConnector`, `KeyValueMemoryConnector`, `ThreadMemoryConnector`
- `src/zeroth/memory/models.py` — `ConnectorScope`, `ConnectorManifest`, `MemoryContext`, `ResolvedMemoryBinding`
- `src/zeroth/memory/registry.py` — `InMemoryConnectorRegistry`, `MemoryConnectorResolver`
- `tests/memory/test_connectors.py` — Existing memory connector tests

### GovernAI Memory Interfaces
- `.venv/lib/python3.12/site-packages/governai/memory/connector.py` — GovernAI `MemoryConnector` protocol (async: read, write, delete, search)
- `.venv/lib/python3.12/site-packages/governai/memory/models.py` — `MemoryScope` enum, `MemoryEntry` model
- `.venv/lib/python3.12/site-packages/governai/memory/scoped.py` — `ScopedMemoryConnector` wrapper
- `.venv/lib/python3.12/site-packages/governai/memory/auditing.py` — `AuditingMemoryConnector` wrapper
- `.venv/lib/python3.12/site-packages/governai/memory/dict_connector.py` — Reference in-memory implementation

### Sandbox System (existing)
- `src/zeroth/execution_units/sandbox.py` — `SandboxManager`, `SandboxBackendMode`, `DockerSandboxConfig`, `SandboxEnvironment`, `SandboxExecutionResult`
- `src/zeroth/execution_units/constraints.py` — `ResourceConstraints`, `build_docker_resource_flags()`
- `src/zeroth/execution_units/runner.py` — `ExecutableUnitRunner` (calls SandboxManager)
- `src/zeroth/execution_units/models.py` — `ExecutableUnitManifest`, `ResourceLimits`, `ExecutionMode`

### Storage & Config (Phase 11 patterns to follow)
- `src/zeroth/storage/redis.py` — `RedisConfig` for Redis connection management
- `src/zeroth/config/` — Unified pydantic-settings config pattern (ZerothSettings, sub-models)
- `src/zeroth/storage/sqlite.py` — Async database protocol reference

### Agent Runtime Integration
- `src/zeroth/agent_runtime/runner.py` — `AgentRunner` memory integration (where connectors are called)
- `src/zeroth/audit/models.py` — `MemoryAccessRecord`, `NodeAuditRecord`

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `MemoryConnector` protocol and `MemoryConnectorResolver`: Core registration/resolution pattern — new connectors plug into this existing system
- `RedisConfig`: Connection management for Redis — reuse for Redis memory connectors
- `SandboxManager` with LOCAL/DOCKER modes: Extensible backend selection — add SIDECAR mode
- `ResourceConstraints` and `build_docker_resource_flags()`: Docker resource limit translation — reuse in sidecar
- `InMemoryConnectorRegistry`: Connector lookup — register new connector types here
- GovernAI `ScopedMemoryConnector` and `AuditingMemoryConnector`: Ready-to-use wrappers — wrap all connectors at resolution time

### Established Patterns
- Async protocol/interface pattern (Phase 11): All new connectors must be async
- Decorator/wrapper pattern (Phase 13): GovernAI wrappers stack transparently like InstrumentedProviderAdapter
- Per-node configuration (Phase 12): Agent nodes specify their own memory connector type
- Pydantic-settings sub-models: New backend settings follow ZerothSettings pattern
- testcontainers for integration tests: Real Redis/Postgres/Elasticsearch containers in CI

### Integration Points
- `MemoryConnectorResolver.resolve()` — Where new connectors are instantiated and wrapped
- `AgentRunner._load_memory()` / `._save_memory()` — Where connectors are called during execution
- `ServiceBootstrap` — Where connector singletons are created and registered
- `SandboxManager._resolve_backend()` — Where SIDECAR mode selection happens
- `ZerothSettings` — Where new backend config sub-models are added

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Auto-mode selected recommended defaults for all areas.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 14-memory-connectors-container-sandbox*
*Context gathered: 2026-04-07*
