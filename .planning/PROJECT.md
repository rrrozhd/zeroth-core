# Zeroth

## What This Is

Zeroth is a governed medium-code platform for building, running, and deploying production-grade multi-agent systems as standalone API services. The platform provides graph-based orchestration, typed contracts, sandboxed execution, human approvals, per-node audit trails, identity/RBAC, tenant isolation, deployment provenance, durable dispatch, real LLM provider integration (100+ models via LiteLLM), token economics via Regulus, external memory backends, and containerized deployment with health probes.

## Core Value

Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## Current Milestone: v2.0 Zeroth Studio

**Goal:** Build a visual workflow authoring UI for governed multi-agent systems, using Vue 3 + Vue Flow, informed by n8n's canvas patterns but reimplemented for Zeroth's governance-first domain model.

**Target features:**
- Visual graph editor (Vue Flow canvas) with node placement, edge drawing, auto-layout
- Graph authoring API (REST/WS) bridging Studio frontend to Zeroth's backend
- Studio shell with workflow rail, canvas area, and inspector panel
- Reusable asset authoring (agents, execution units, approval gates, memory resources)
- Environment management and deployment-time bindings

## Current State

**Shipped:** v1.1 Production Readiness (2026-04-09)
**Codebase:** ~22K LOC source + ~18K LOC tests (Python)
**Tech stack:** Python / FastAPI / Pydantic / SQLAlchemy / Alembic / LiteLLM / ARQ / Docker
**Frontend stack (v2.0):** Vue 3 / Vite / Pinia / Vue Flow / dagre / CodeMirror 6

The platform is production-viable: real LLM providers, economic controls, external memory, durable webhooks, horizontal scaling, and containerized deployment are all wired and verified. v2.0 adds Studio UI for visual workflow authoring.

## Requirements

### Validated

- ✓ Governed workflow graph modeling, validation, and versioning — v1.0
- ✓ Runtime orchestration, approvals, memory, and deployment-bound service APIs — v1.0
- ✓ Identity, governance evidence, runtime hardening, and durable control-plane foundations — v1.0
- ✓ Unified pydantic-settings config and async Postgres storage backend — v1.1
- ✓ Real LLM provider adapters (OpenAI, Anthropic via LiteLLM) with retry and token audit — v1.1
- ✓ Regulus economics: cost events, cost attribution, budget enforcement, cost REST endpoints — v1.1
- ✓ External memory connectors (Redis KV/thread, pgvector, ChromaDB, Elasticsearch) with GovernAI protocol — v1.1
- ✓ Container sandbox sidecar architecture — v1.1
- ✓ Durable webhooks with HMAC signing, retry, dead-letter store — v1.1
- ✓ Approval SLA timeouts with escalation policies — v1.1
- ✓ Distributed dispatch with ARQ wakeup and horizontal worker scaling — v1.1
- ✓ Containerized deployment (Dockerfile, docker-compose, Nginx TLS, health probes) — v1.1
- ✓ API versioning (/v1/), OpenAPI spec, TLS/HTTPS support — v1.1
- ✓ Native LLM API parity: tool schemas, structured output, model params, MCP servers — v1.1

### Active

- [ ] Visual graph editor with interactive node placement and edge drawing
- [ ] Graph authoring API (REST/WS) for Studio frontend
- [ ] Studio shell with workflow rail, canvas, inspector
- [ ] Reusable asset authoring (agents, execution units, memory resources)
- [ ] Environment management and deployment-time bindings

### Out of Scope

- Mobile apps — web-based platform only
- Judge/evaluation subsystem — preserved as extension point per original PLAN.md
- Runtime architecture rewrite — production backend is solid, next work is Studio UI
- Real-time streaming — async invocation model sufficient
- Custom LLM hosting — integrates with hosted providers only
- LLM response caching / model routing optimization — deferred to future economics milestone
- Model fallback chains / streaming responses — deferred to future provider milestone

## Context

Mature Python backend with 280+ tests, lint clean, broad pytest coverage. GovernAI v0.3.0-dev pinned from GitHub for memory, secrets, capability policy, agent spec, and tool manifest modules. Regulus companion service provides LLM economics telemetry. All v1.1 requirements verified via milestone audit with zero gaps.

**v2.0 design reference:** n8n's frontend (SUL-licensed, not forkable) studied for canvas patterns, shell layout, and inspector UX. MIT libraries adopted directly: @vue-flow/core, @dagrejs/dagre, CodeMirror 6. Our canvas must be governance-aware (approval gates, audit decorators, sandbox indicators, RBAC) — a clean reimplementation, not a port.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GovernAI pinned to GitHub commit, not PyPI | v0.3.0 unreleased but has needed memory/secrets/policy modules | ✓ Good |
| Regulus integrated via SDK, not embedded | Separation of concerns — economics is a companion service | ✓ Good — fail-open pattern works well |
| Postgres as production storage, SQLite retained for dev/test | Production needs vs developer experience | ✓ Good — dual backend via env var |
| Studio UI deferred to v2.0 | Backend must be production-viable before adding frontend | ✓ Good — v1.1 shipped solid |
| Vue 3 + Vue Flow for Studio | Same stack as n8n reference; Vue Flow is purpose-built for Vue, MIT-licensed | — Pending |
| n8n as design reference only | SUL license prevents forking; tight coupling to n8n backend makes extraction impractical | ✓ Good |
| LiteLLM as provider abstraction layer | Routes to 100+ models without per-provider adapters | ✓ Good — unified retry/token capture |
| Worktree isolation for parallel phase development | Independent progress on subsystems | ⚠️ Revisit — creates integration wiring gaps |
| Bootstrap wiring as dedicated integration phase | Connects independently developed subsystems | ✓ Good — effective gap closure pattern |

## Constraints

- **Tech stack**: Python/FastAPI/Pydantic backend — all new work integrates with existing foundation
- **GovernAI**: Pinned to git+https://github.com/rrrozhd/governai.git@7452de4 — use new v0.3.0 modules where applicable
- **Regulus**: SDK-level integration preferred — Regulus backend runs as companion service, not embedded
- **Backward compatibility**: Existing tests must continue passing through all changes
- **Architecture**: Modular monolith — new capabilities are new modules, not separate services (except Regulus backend)

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? Move to Out of Scope with reason
2. Requirements validated? Move to Validated with phase reference
3. New requirements emerged? Add to Active
4. Decisions to log? Add to Key Decisions
5. "What This Is" still accurate? Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-09 after v2.0 Zeroth Studio milestone start*
