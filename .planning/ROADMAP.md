# Roadmap: Zeroth

## Milestones

- ✅ **v1.0 Runtime Foundation** - Phases 1-9 (backend/runtime foundation complete)
- 🚧 **v1.1 Production Readiness** - Phases 11-17 (in progress)
- 📋 **v2.0 Zeroth Studio** - Phases 18-21 (planned)

## Phases

<details>
<summary>✅ v1.0 Runtime Foundation (Phases 1-9) - SHIPPED 2026-03-27</summary>

### Phase 1: Core Foundation
**Goal**: Establish graph, contract, mapping, run, validation, and versioning foundations.
**Depends on**: Nothing (first phase)
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Graphs, contracts, mappings, and runs have typed persisted foundations
  2. Validation and versioning support safe authoring/runtime progression
**Plans**: Complete

Plans:
- [x] 01-01: Domain models and graph schema
- [x] 01-02: Contract registry, mappings, validation, and versioning

### Phase 2: Execution Core
**Goal**: Deliver orchestration, agent runtime, execution units, conditions, and thread persistence.
**Depends on**: Phase 1
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Governed workflows execute end-to-end with agents, units, and branching
  2. Thread continuity and checkpoints persist across runs
**Plans**: Complete

Plans:
- [x] 02-01: Execution unit and agent runtime foundations
- [x] 02-02: Orchestration, conditions, tools, and thread handling

### Phase 3: Platform Control
**Goal**: Add memory connectors, approvals, and audit-aware control surfaces.
**Depends on**: Phase 2
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. Memory, approvals, and control interactions work through persisted platform models
  2. Governance-relevant actions are captured in audit flows
**Plans**: Complete

Plans:
- [x] 03-01: Memory and approval lifecycle
- [x] 03-02: Approval API and audit integration

### Phase 4: Deployment Surface
**Goal**: Expose published deployments through a service wrapper.
**Depends on**: Phase 3
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Published graphs can be deployed and invoked through HTTP service APIs
  2. Deployment bootstrap and service routes work against pinned snapshots
**Plans**: Complete

Plans:
- [x] 04-01: Deployment and service wrapper

### Phase 5: Integration & Polish
**Goal**: Verify end-to-end behavior and document implementation-facing specs.
**Depends on**: Phase 4
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. End-to-end service flows are validated
  2. MVP runtime foundation is documented and shippable
**Plans**: Complete

Plans:
- [x] 05-01: Integration verification and specs

### Phase 6: Identity & Tenant Governance
**Goal**: Add service authentication, RBAC, and tenant/workspace scoping.
**Depends on**: Phase 5
**Requirements**: RUN-02
**Success Criteria** (what must be TRUE):
  1. Authenticated access and scope enforcement protect service surfaces
  2. Identity lineage is visible in runtime/audit flows
**Plans**: Complete

Plans:
- [x] 06-01: Authentication, RBAC, and tenant isolation

### Phase 7: Transparent Governance & Provenance
**Goal**: Expose audit, evidence, and attestation surfaces.
**Depends on**: Phase 6
**Requirements**: RUN-02
**Success Criteria** (what must be TRUE):
  1. Users can inspect public audit and evidence surfaces
  2. Provenance and attestation are verifiable
**Plans**: Complete

Plans:
- [x] 07-01: Audit, evidence, and attestation surfaces

### Phase 8: Runtime Security Hardening
**Goal**: Harden sandboxing, policy enforcement, and secret protection.
**Depends on**: Phase 7
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. Runtime execution is hardened against unsafe execution paths
  2. Secret handling and policy enforcement are production-oriented
**Plans**: Complete

Plans:
- [x] 08-01: Runtime security hardening

### Phase 9: Durable Control Plane & Production Operations
**Goal**: Make dispatch, recovery, guardrails, metrics, and admin controls durable.
**Depends on**: Phase 8
**Requirements**: RUN-01, RUN-02, RUN-03
**Success Criteria** (what must be TRUE):
  1. Run dispatch and recovery survive process turnover
  2. Operators can observe and control runs safely
**Plans**: Complete

Plans:
- [x] 09-01: Durable dispatch, guardrails, observability, and admin controls

</details>

<details>
<summary>⏸ Phase 10: Studio Shell & Workflow Authoring (paused — v1.1 takes priority)</summary>

### Phase 10: Studio Shell & Workflow Authoring
**Goal**: Establish the Studio shell, canvas-first navigation, workflow drafts, and authoring-time contracts/validation UX.
**Depends on**: Phase 9
**Requirements**: STU-01, STU-02, AST-04, UX-01, UX-02
**Success Criteria** (what must be TRUE):
  1. User can open a Studio shell with workflow rail, canvas, inspector, and mode switch
  2. User can manage workflow drafts separately from deployed runtime graphs
  3. Authoring validation and contract configuration work in node-local flows
**Plans**: 3 plans

Plans:
- [ ] 10-01: Studio backend session, draft, revision, and lease foundations
- [ ] 10-02: Frontend shell, routing, and canvas/inspector baseline
- [ ] 10-03: Validation, contract-authoring UX, and autosave boundaries

</details>

### 🚧 v1.1 Production Readiness (In Progress)

**Milestone Goal:** Close every gap between Zeroth's architecture and a production-viable governed AI workflow platform — real LLM providers, economic control via Regulus, reliable infrastructure, hardened governance, and deployable operations.

- [x] **Phase 11: Config & Postgres Storage** - Unified config and production-grade storage backend (completed 2026-04-06)
- [x] **Phase 12: Real LLM Providers & Retry** - OpenAI/Anthropic adapters with retry and token capture (completed 2026-04-06)
- [x] **Phase 13: Regulus Economics Integration** - Token metering, cost attribution, and budget enforcement (completed 2026-04-07)
- [ ] **Phase 14: Memory Connectors & Container Sandbox** - External memory backends and hardened Docker sandbox
- [ ] **Phase 15: Webhooks & Approval SLA** - Durable webhook delivery and approval escalation policies
- [ ] **Phase 16: Distributed Dispatch & Horizontal Scaling** - ARQ-backed wakeup and multi-worker lease validation
- [ ] **Phase 17: Deployment Packaging & Operations** - Dockerfile, API versioning, health probes, and TLS

## Phase Details

### Phase 11: Config & Postgres Storage
**Goal**: All platform configuration loads from a unified pydantic-settings source and Postgres is available as a production storage backend behind an async repository interface with Alembic-managed migrations.
**Depends on**: Phase 9
**Requirements**: CFG-01, CFG-02, CFG-03
**Success Criteria** (what must be TRUE):
  1. Platform starts with all settings resolved from environment variables and .env files, failing fast on missing required values
  2. `ZEROTH_DB_BACKEND=postgres` boots the platform against a Postgres database with no code changes beyond the env var
  3. `ZEROTH_DB_BACKEND=sqlite` continues to pass all 280 existing tests without modification
  4. Alembic migrations run cleanly against a fresh Postgres database and produce a schema matching the SQLite test schema
**Plans**: 3 plans

Plans:
- [x] 11-01-PLAN.md — Config package, async database protocol, implementations, Alembic migrations
- [ ] 11-02-PLAN.md — Async rewrite of all repositories and callers
- [x] 11-03-PLAN.md — Test infrastructure, dual-backend verification, Postgres integration tests

### Phase 12: Real LLM Providers & Retry
**Goal**: The platform can invoke real OpenAI and Anthropic models through typed adapters, with automatic retry on transient failures and token usage captured in node audit records.
**Depends on**: Phase 11
**Requirements**: LLM-01, LLM-02, LLM-03, LLM-04
**Success Criteria** (what must be TRUE):
  1. An agent node configured with `provider: openai` executes against a real OpenAI model when live credentials are present
  2. An agent node configured with `provider: anthropic` executes against a real Anthropic model when live credentials are present
  3. A provider call that encounters a rate-limit error retries with exponential backoff and jitter before propagating failure
  4. Node audit records include `token_usage.input` and `token_usage.output` after a successful provider call
**Plans**: 3 plans

Plans:
- [ ] 12-01-PLAN.md — LiteLLM dependencies, TokenUsage model, LiteLLMProviderAdapter
- [x] 12-02-PLAN.md — Retry module with exponential backoff, error classification, runner upgrade
- [x] 12-03-PLAN.md — Token audit wiring, unit tests, live integration tests

### Phase 13: Regulus Economics Integration
**Goal**: Every LLM call emits a cost event to the Regulus backend, token costs are attributed per node/run/tenant, budget caps are enforced before execution, and cost totals are queryable via REST.
**Depends on**: Phase 12
**Requirements**: ECON-01, ECON-02, ECON-03, ECON-04
**Success Criteria** (what must be TRUE):
  1. An `InstrumentedProviderAdapter` wrapping a real provider adapter emits a Regulus `ExecutionEvent` for each LLM call without modifying the orchestrator
  2. Node audit records carry cost attribution fields (node, run, tenant, deployment) populated from Regulus event data
  3. A tenant that has exceeded its budget cap receives a policy rejection before any LLM call is attempted
  4. `GET /v1/tenants/{id}/cost` returns a cumulative spend figure consistent with audit records
**Plans**: 3 plans

Plans:
- [x] 13-01-PLAN.md — Econ module foundation: SDK dependency, InstrumentedProviderAdapter, cost estimation, config
- [ ] 13-02-PLAN.md — Budget enforcement: BudgetEnforcer with TTL cache, AgentRunner integration
- [x] 13-03-PLAN.md — Cost REST endpoints and ServiceBootstrap wiring

### Phase 14: Memory Connectors & Container Sandbox
**Goal**: Agents can use persistent external memory backends (Redis KV, Redis thread, pgvector, ChromaDB, Elasticsearch) bridged to GovernAI protocol, and untrusted execution units run inside a Docker sandbox via a sidecar architecture.
**Depends on**: Phase 11
**Requirements**: MEM-01, MEM-02, MEM-03, MEM-04, MEM-05, MEM-06, SBX-01, SBX-02
**Success Criteria** (what must be TRUE):
  1. An agent configured with `memory: redis_kv` persists and retrieves key-value state across separate process restarts
  2. An agent configured with `memory: redis_thread` retains conversation history across separate process restarts
  3. Semantic memory queries against `pgvector`, `chroma`, or `elasticsearch` connectors return relevant results without in-process storage
  4. All Zeroth memory connectors expose `ScopedMemoryConnector` and `AuditingMemoryConnector` interfaces from GovernAI v0.3.0
  5. An executable unit marked `UNTRUSTED` runs inside a Docker container with resource limits and no host network access; the API container never mounts the Docker socket
**Plans**: 5 plans

Plans:
- [x] 14-01-PLAN.md — GovernAI protocol rewrite, in-memory connectors, resolver wrapping, AgentRunner update
- [ ] 14-02-PLAN.md — Redis KV and Redis thread memory connectors
- [ ] 14-03-PLAN.md — pgvector, ChromaDB, and Elasticsearch memory connectors
- [x] 14-04-PLAN.md — Sandbox sidecar service, HTTP client, SandboxManager SIDECAR mode
- [ ] 14-05-PLAN.md — Connector registration factory and ServiceBootstrap wiring

### Phase 15: Webhooks & Approval SLA
**Goal**: Callers receive durable push notifications on run completion, approval requests, and failure events, and approval SLA timeouts trigger escalation rather than silent expiry.
**Depends on**: Phase 11, Phase 12
**Requirements**: OPS-01, OPS-02
**Success Criteria** (what must be TRUE):
  1. A subscriber that registers a webhook URL receives an HTTP POST within a reasonable window after run completion or failure, even if the first delivery attempt fails
  2. A failed webhook delivery is retried with exponential backoff and eventually written to a dead-letter store rather than silently dropped
  3. An approval that is not actioned within its configured SLA window escalates to the configured delegate or raises an alert rather than hanging indefinitely
**Plans**: 3 plans

Plans:
- [ ] 15-01-PLAN.md — TBD
- [ ] 15-02-PLAN.md — TBD
- [ ] 15-03-PLAN.md — TBD

### Phase 16: Distributed Dispatch & Horizontal Scaling
**Goal**: Multiple worker processes share a Postgres lease store for run ownership, and an ARQ-backed wakeup notification reduces lease poll latency without replacing the database as the authoritative queue.
**Depends on**: Phase 11
**Requirements**: OPS-04, OPS-05
**Success Criteria** (what must be TRUE):
  1. Two or more worker processes started against the same Postgres instance each claim disjoint sets of pending runs with no duplicate execution
  2. Submitting a run triggers an ARQ wakeup notification that causes a worker to begin processing sooner than the configured poll interval
  3. Killing a worker mid-run causes another worker to reclaim the run after the lease expires, with no manual intervention
**Plans**: 3 plans

Plans:
- [ ] 16-01-PLAN.md — TBD
- [ ] 16-02-PLAN.md — TBD
- [ ] 16-03-PLAN.md — TBD

### Phase 17: Deployment Packaging & Operations
**Goal**: The platform ships as a reproducible container image with versioned API routes, auto-generated OpenAPI documentation, TLS support, and readiness/liveness probes that block traffic until all dependencies are healthy.
**Depends on**: Phase 11, Phase 14, Phase 16
**Requirements**: DEP-01, DEP-02, DEP-03, DEP-04, OPS-03
**Success Criteria** (what must be TRUE):
  1. `docker compose up` brings up Zeroth, Postgres, Redis, Regulus backend, and the sandbox sidecar from a single compose file with no manual setup steps
  2. All API routes respond under the `/v1/` prefix; existing unversioned paths remain active as aliases so no existing tests break
  3. `GET /health/ready` returns HTTP 200 only when the database, Redis, and optional Regulus backend are all reachable, and returns a structured error otherwise
  4. The OpenAPI spec served at `/openapi.json` documents all `/v1/` routes with correct schemas and authentication requirements
  5. The platform accepts HTTPS traffic when configured with a TLS certificate or behind a Nginx/Traefik reverse proxy
**Plans**: 3 plans

Plans:
- [ ] 17-01-PLAN.md — TBD
- [ ] 17-02-PLAN.md — TBD
- [ ] 17-03-PLAN.md — TBD

### 📋 v2.0 Zeroth Studio (Planned)

**Milestone Goal:** Deliver the authoring, asset, environment, and execution UX needed to turn the backend runtime foundation into a full Studio product.

### Phase 18: Studio Shell & Workflow Authoring
**Goal**: Establish the Studio shell, canvas-first navigation, workflow drafts, and authoring-time contracts/validation UX.
**Depends on**: Phase 17
**Requirements**: STU-01, STU-02, AST-04, UX-01, UX-02
**Success Criteria** (what must be TRUE):
  1. User can open a Studio shell with workflow rail, canvas, inspector, and mode switch
  2. User can manage workflow drafts separately from deployed runtime graphs
  3. Authoring validation and contract configuration work in node-local flows
**Plans**: 3 plans
**UI hint**: yes

Plans:
- [ ] 18-01: Studio backend session, draft, revision, and lease foundations
- [ ] 18-02: Frontend shell, routing, and canvas/inspector baseline
- [ ] 18-03: Validation, contract-authoring UX, and autosave boundaries

### Phase 19: Studio Runtime, Executions, And Testing
**Goal**: Add execution timelines, test runs, and runtime/gateway views to Studio.
**Depends on**: Phase 18
**Requirements**: STU-03, STU-04, UX-03
**Success Criteria** (what must be TRUE):
  1. User can run draft tests from Studio against persisted authoring snapshots
  2. User can inspect runtime data by run and by node
  3. Studio reuses existing runtime, audit, approval, and admin surfaces through a gateway layer
**Plans**: 3 plans
**UI hint**: yes

Plans:
- [ ] 19-01: Studio runtime gateway and query normalization
- [ ] 19-02: Executions and tests views
- [ ] 19-03: Node-scoped and run-scoped governance UX

### Phase 20: Studio Assets
**Goal**: Add reusable asset authoring for agents, executable units, and memory resources.
**Depends on**: Phase 18
**Requirements**: AST-01, AST-02, AST-03
**Success Criteria** (what must be TRUE):
  1. User can browse and select reusable assets from Studio
  2. Asset definitions can be edited separately from workflow node instances
  3. Asset workflows preserve canvas context by default
**Plans**: 2 plans
**UI hint**: yes

Plans:
- [ ] 20-01: Asset models and backend persistence
- [ ] 20-02: Asset slide-over UX and deep-edit flows

### Phase 21: Environments & Deployment UX
**Goal**: Add environment management and deployment-time bindings for Studio.
**Depends on**: Phase 19, Phase 20
**Requirements**: AST-05
**Success Criteria** (what must be TRUE):
  1. User can switch current environment from the Studio header
  2. User can manage environment-bound secrets and bindings safely
  3. Publish/deploy flows use named environments as first-class configuration
**Plans**: 2 plans
**UI hint**: yes

Plans:
- [ ] 21-01: Environment registry and secret/binding management
- [ ] 21-02: Header environment UX and deploy integration

## Progress

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Core Foundation | v1.0 | 2/2 | Complete | 2026-03-19 |
| 2. Execution Core | v1.0 | 2/2 | Complete | 2026-03-19 |
| 3. Platform Control | v1.0 | 2/2 | Complete | 2026-03-19 |
| 4. Deployment Surface | v1.0 | 1/1 | Complete | 2026-03-20 |
| 5. Integration & Polish | v1.0 | 1/1 | Complete | 2026-03-26 |
| 6. Identity & Tenant Governance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 7. Transparent Governance & Provenance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 8. Runtime Security Hardening | v1.0 | 1/1 | Complete | 2026-03-27 |
| 9. Durable Control Plane & Production Operations | v1.0 | 1/1 | Complete | 2026-03-27 |
| 10. Studio Shell & Workflow Authoring | v2.0 | 0/3 | Paused | - |
| 11. Config & Postgres Storage | v1.1 | 2/3 | Complete    | 2026-04-06 |
| 12. Real LLM Providers & Retry | v1.1 | 2/3 | Complete    | 2026-04-06 |
| 13. Regulus Economics Integration | v1.1 | 2/3 | Complete    | 2026-04-07 |
| 14. Memory Connectors & Container Sandbox | v1.1 | 1/5 | In Progress|  |
| 15. Webhooks & Approval SLA | v1.1 | 0/TBD | Not started | - |
| 16. Distributed Dispatch & Horizontal Scaling | v1.1 | 0/TBD | Not started | - |
| 17. Deployment Packaging & Operations | v1.1 | 0/TBD | Not started | - |
| 18. Studio Shell & Workflow Authoring | v2.0 | 0/3 | Not started | - |
| 19. Studio Runtime, Executions, And Testing | v2.0 | 0/3 | Not started | - |
| 20. Studio Assets | v2.0 | 0/2 | Not started | - |
| 21. Environments & Deployment UX | v2.0 | 0/2 | Not started | - |
