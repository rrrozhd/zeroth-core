# Phase 14: Memory Connectors & Container Sandbox - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-07
**Phase:** 14-memory-connectors-container-sandbox
**Areas discussed:** GovernAI memory bridge, Redis connector architecture, Vector store abstraction, Sandbox sidecar design, Connector configuration
**Mode:** Auto (--auto flag — all areas auto-selected with recommended defaults)

---

## GovernAI Memory Bridge

| Option | Description | Selected |
|--------|-------------|----------|
| Implement GovernAI protocol directly | Rewrite Zeroth connectors to match GovernAI async MemoryConnector, wrap with ScopedMemoryConnector and AuditingMemoryConnector | ✓ |
| Adapter layer between protocols | Keep Zeroth protocol, add adapter that translates to GovernAI interface | |
| Dual protocol support | Connectors implement both Zeroth and GovernAI protocols | |

**User's choice:** [auto] Implement GovernAI protocol directly (recommended default)
**Notes:** GovernAI's protocol is async (matches Phase 11 async rewrite), has built-in scoping and auditing wrappers. Direct implementation avoids unnecessary adapter layer.

---

## Redis Connector Architecture

| Option | Description | Selected |
|--------|-------------|----------|
| Two separate connectors | RedisKVMemoryConnector + RedisThreadMemoryConnector — mirrors existing in-memory split | ✓ |
| Single unified connector | One RedisMemoryConnector handling both KV and thread via scope parameter | |
| Three connectors | Separate KV, thread, and pub/sub connectors | |

**User's choice:** [auto] Two separate connectors (recommended default)
**Notes:** Mirrors existing in-memory pattern (KeyValueMemoryConnector / ThreadMemoryConnector). Different storage semantics (GET/SET vs ordered collections) justify separate implementations.

---

## Vector Store Abstraction

| Option | Description | Selected |
|--------|-------------|----------|
| Direct per-backend implementations | Each backend implements GovernAI MemoryConnector directly — no intermediate layer | ✓ |
| Common vector abstraction | Shared VectorMemoryConnector base class with backend-specific drivers | |
| LangChain vectorstore integration | Use LangChain's VectorStore abstraction as the common interface | |

**User's choice:** [auto] Direct per-backend implementations (recommended default)
**Notes:** Each vector backend has different query semantics, embedding handling, and configuration. A common abstraction would be leaky. GovernAI's MemoryConnector.search() is already the common interface.

---

## Sandbox Sidecar Design

| Option | Description | Selected |
|--------|-------------|----------|
| HTTP REST sidecar | Sidecar exposes REST API, API container calls over internal Docker network | ✓ |
| gRPC sidecar | Sidecar uses gRPC for lower-latency sandbox operations | |
| Unix socket sidecar | Sidecar communicates via shared volume Unix socket | |

**User's choice:** [auto] HTTP REST sidecar (recommended default)
**Notes:** REST is consistent with the platform's FastAPI architecture. Lower complexity than gRPC, easier to debug/test. Internal Docker network provides sufficient performance.

---

## Connector Configuration

| Option | Description | Selected |
|--------|-------------|----------|
| Per-connector settings in ZerothSettings | New sub-models (RedisMemorySettings, PgvectorSettings, etc.) in unified config | ✓ |
| Connection string per node | Each agent node specifies a full connection string for its memory backend | |
| External config file | Separate memory-connectors.yaml config file | |

**User's choice:** [auto] Per-connector settings in ZerothSettings (recommended default)
**Notes:** Follows Phase 11 config pattern. Centralized config avoids connection string duplication across nodes. Connector instances are singletons shared across agent nodes.

---

## Claude's Discretion

- Redis data structure choices for thread memory
- pgvector table schema and index type
- Embedding model selection for vector connectors
- Sidecar REST API routes and schemas
- ChromaDB collection naming conventions
- Elasticsearch index mapping and analyzer config
- Connection pool sizes and timeouts
- Sidecar authentication requirements

## Deferred Ideas

None — discussion stayed within phase scope.
