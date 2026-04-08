# Requirements: Zeroth v1.1 Production Readiness

**Defined:** 2026-04-06
**Core Value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## v1.1 Requirements

Requirements for production readiness milestone. Each maps to roadmap phases.

### Configuration & Storage

- [x] **CFG-01**: Platform loads configuration from environment variables and .env files with startup validation
- [x] **CFG-02**: Postgres storage backend available behind existing sync repository interface with Alembic migrations
- [x] **CFG-03**: Storage backend selectable via ZEROTH_DB_BACKEND flag (sqlite/postgres) at startup

### LLM Providers

- [x] **LLM-01**: OpenAI provider adapter implements ProviderAdapter protocol via langchain-openai
- [x] **LLM-02**: Anthropic provider adapter implements ProviderAdapter protocol via langchain-anthropic
- [x] **LLM-03**: Provider calls retry with exponential backoff and jitter on rate limits and transient failures
- [x] **LLM-04**: Token usage (input/output) captured from provider responses and attached to node audit records

### Economics

- [x] **ECON-01**: InstrumentedProviderAdapter wraps any ProviderAdapter and emits Regulus ExecutionEvent per LLM call
- [x] **ECON-02**: Token cost attributed per node, run, tenant, and deployment in audit records
- [x] **ECON-03**: Per-tenant and per-deployment budget caps enforced via policy guard before execution
- [x] **ECON-04**: REST endpoints expose cumulative cost per tenant and deployment

### Memory Connectors

- [x] **MEM-01**: Redis-backed key-value memory connector replacing in-memory dict
- [x] **MEM-02**: Redis-backed conversation/thread memory connector replacing in-memory store
- [x] **MEM-03**: pgvector-backed semantic memory connector for agent context retrieval
- [x] **MEM-04**: ChromaDB memory connector for vector similarity search
- [x] **MEM-05**: Elasticsearch memory connector for full-text and hybrid search
- [x] **MEM-06**: Zeroth memory connectors bridged to GovernAI v0.3.0 ScopedMemoryConnector and AuditingMemoryConnector

### Sandbox & Security

- [x] **SBX-01**: Docker-based sandbox backend for untrusted executable units with resource limits and network isolation
- [x] **SBX-02**: Sandbox sidecar architecture prevents Docker socket exposure on the API container

### Operations

- [x] **OPS-01**: Durable webhook notifications for run completion, approval needed, and failure events
- [x] **OPS-02**: Approval SLA timeouts with escalation and delegation policies
- [x] **OPS-03**: Readiness and liveness health probes with dependency checks (DB, Redis, Regulus)
- [x] **OPS-04**: Multi-worker horizontal scaling with shared Postgres lease store (code exists, settings not merged)
- [x] **OPS-05**: ARQ (Redis queue) wakeup notifications supplementing existing lease poller (code exists, bootstrap wiring not merged)

### Deployment

- [x] **DEP-01**: Dockerfile and docker-compose for Zeroth, Postgres, Redis, and Regulus backend
- [x] **DEP-02**: API routes prefixed with /v1/ with version negotiation headers
- [x] **DEP-03**: OpenAPI spec auto-generated from FastAPI route definitions
- [x] **DEP-04**: TLS/HTTPS support via reverse proxy or uvicorn SSL configuration

### Agent Node LLM API Parity

- [ ] **API-01**: ProviderRequest carries native tool/function-calling schemas to provider adapters; ToolAttachmentManifest converts to provider-native format
- [ ] **API-02**: Agent nodes support native structured output via response_format (json_schema mode) for providers that support it, with post-hoc validation fallback
- [ ] **API-03**: Agent nodes support per-node model parameters (temperature, max_tokens, top_p, stop, seed, tool_choice) forwarded to provider API
- [ ] **API-04**: Agent nodes can declare MCP server connections; tools discovered at startup and callable during execution

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
| CFG-01 | Phase 11 | Complete |
| CFG-02 | Phase 11 | Complete |
| CFG-03 | Phase 11 | Complete |
| LLM-01 | Phase 12 | Complete |
| LLM-02 | Phase 12 | Complete |
| LLM-03 | Phase 12 | Complete |
| LLM-04 | Phase 12 | Complete |
| ECON-01 | Phase 18 | Complete |
| ECON-02 | Phase 18 | Complete |
| ECON-03 | Phase 13 | Complete |
| ECON-04 | Phase 18 | Complete |
| MEM-01 | Phase 18 | Complete |
| MEM-02 | Phase 14 | Complete |
| MEM-03 | Phase 14 | Complete |
| MEM-04 | Phase 14 | Complete |
| MEM-05 | Phase 14 | Complete |
| MEM-06 | Phase 14 | Complete |
| SBX-01 | Phase 14 | Complete |
| SBX-02 | Phase 14 | Complete |
| OPS-01 | Phase 15 | Complete |
| OPS-02 | Phase 15 | Complete |
| OPS-03 | Phase 17 | Complete |
| OPS-04 | Phase 18 | Complete |
| OPS-05 | Phase 18 | Complete |
| DEP-01 | Phase 17 | Complete |
| DEP-02 | Phase 17 | Complete |
| DEP-03 | Phase 17 | Complete |
| DEP-04 | Phase 17 | Complete |

**Coverage:**
- v1.1 requirements: 28 total
- Satisfied: 28
- Partial: 0
- Unmapped: 0

---
*Requirements defined: 2026-04-06*
*Last updated: 2026-04-08 after Phase 18 gap closure completion*
