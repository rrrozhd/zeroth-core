# Requirements: Zeroth v1.1 Production Readiness

**Defined:** 2026-04-06
**Core Value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## v1.1 Requirements

Requirements for production readiness milestone. Each maps to roadmap phases.

### Configuration & Storage

- [ ] **CFG-01**: Platform loads configuration from environment variables and .env files with startup validation
- [ ] **CFG-02**: Postgres storage backend available behind existing sync repository interface with Alembic migrations
- [ ] **CFG-03**: Storage backend selectable via ZEROTH_DB_BACKEND flag (sqlite/postgres) at startup

### LLM Providers

- [ ] **LLM-01**: OpenAI provider adapter implements ProviderAdapter protocol via langchain-openai
- [ ] **LLM-02**: Anthropic provider adapter implements ProviderAdapter protocol via langchain-anthropic
- [ ] **LLM-03**: Provider calls retry with exponential backoff and jitter on rate limits and transient failures
- [ ] **LLM-04**: Token usage (input/output) captured from provider responses and attached to node audit records

### Economics

- [ ] **ECON-01**: InstrumentedProviderAdapter wraps any ProviderAdapter and emits Regulus ExecutionEvent per LLM call
- [ ] **ECON-02**: Token cost attributed per node, run, tenant, and deployment in audit records
- [ ] **ECON-03**: Per-tenant and per-deployment budget caps enforced via policy guard before execution
- [ ] **ECON-04**: REST endpoints expose cumulative cost per tenant and deployment

### Memory Connectors

- [ ] **MEM-01**: Redis-backed key-value memory connector replacing in-memory dict
- [ ] **MEM-02**: Redis-backed conversation/thread memory connector replacing in-memory store
- [ ] **MEM-03**: pgvector-backed semantic memory connector for agent context retrieval
- [ ] **MEM-04**: ChromaDB memory connector for vector similarity search
- [ ] **MEM-05**: Elasticsearch memory connector for full-text and hybrid search
- [ ] **MEM-06**: Zeroth memory connectors bridged to GovernAI v0.3.0 ScopedMemoryConnector and AuditingMemoryConnector

### Sandbox & Security

- [ ] **SBX-01**: Docker-based sandbox backend for untrusted executable units with resource limits and network isolation
- [ ] **SBX-02**: Sandbox sidecar architecture prevents Docker socket exposure on the API container

### Operations

- [ ] **OPS-01**: Durable webhook notifications for run completion, approval needed, and failure events
- [ ] **OPS-02**: Approval SLA timeouts with escalation and delegation policies
- [ ] **OPS-03**: Readiness and liveness health probes with dependency checks (DB, Redis, Regulus)
- [ ] **OPS-04**: Multi-worker horizontal scaling with shared Postgres lease store
- [ ] **OPS-05**: ARQ (Redis queue) wakeup notifications supplementing existing lease poller

### Deployment

- [ ] **DEP-01**: Dockerfile and docker-compose for Zeroth, Postgres, Redis, and Regulus backend
- [ ] **DEP-02**: API routes prefixed with /v1/ with version negotiation headers
- [ ] **DEP-03**: OpenAPI spec auto-generated from FastAPI route definitions
- [ ] **DEP-04**: TLS/HTTPS support via reverse proxy or uvicorn SSL configuration

## Future Requirements

Deferred to subsequent milestones.

### Studio UI

- **UI-01**: Visual graph editor with interactive node placement and edge drawing
- **UI-02**: Graph authoring API (REST/WS) for Studio frontend
- **UI-03**: Studio shell with workflow rail, canvas, inspector

### Advanced Economics

- **ECON-05**: LLM response caching (semantic and exact-match)
- **ECON-06**: Model routing and cost optimization (route cheaper queries to smaller models)
- **ECON-07**: Regulus A/B experiments for model comparison

### Extended Providers

- **LLM-05**: Model fallback chains (primary → fallback provider on failure)
- **LLM-06**: Streaming response support for agent nodes

## Out of Scope

| Feature | Reason |
|---------|--------|
| Studio UI | Deferred to next milestone — production backend must be solid first |
| Mobile apps | Web-based platform only |
| Judge/evaluation subsystem | Preserved as extension point per original PLAN.md |
| Runtime architecture rewrite | This milestone hardens existing architecture, not rewrites |
| Real-time streaming | Async invocation model is sufficient for v1.1 |
| Custom LLM hosting | Platform integrates with hosted providers, not self-hosted models |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| CFG-01 | — | Pending |
| CFG-02 | — | Pending |
| CFG-03 | — | Pending |
| LLM-01 | — | Pending |
| LLM-02 | — | Pending |
| LLM-03 | — | Pending |
| LLM-04 | — | Pending |
| ECON-01 | — | Pending |
| ECON-02 | — | Pending |
| ECON-03 | — | Pending |
| ECON-04 | — | Pending |
| MEM-01 | — | Pending |
| MEM-02 | — | Pending |
| MEM-03 | — | Pending |
| MEM-04 | — | Pending |
| MEM-05 | — | Pending |
| MEM-06 | — | Pending |
| SBX-01 | — | Pending |
| SBX-02 | — | Pending |
| OPS-01 | — | Pending |
| OPS-02 | — | Pending |
| OPS-03 | — | Pending |
| OPS-04 | — | Pending |
| OPS-05 | — | Pending |
| DEP-01 | — | Pending |
| DEP-02 | — | Pending |
| DEP-03 | — | Pending |
| DEP-04 | — | Pending |

**Coverage:**
- v1.1 requirements: 28 total
- Mapped to phases: 0
- Unmapped: 28

---
*Requirements defined: 2026-04-06*
*Last updated: 2026-04-06 after milestone v1.1 initialization*
