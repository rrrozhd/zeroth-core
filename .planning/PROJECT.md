# Zeroth

## What This Is

Zeroth is a governed medium-code platform for building, running, and deploying production-grade multi-agent systems as standalone API services. The repository contains a complete backend/runtime foundation (Phases 1-9) covering graph-based orchestration, typed contracts, sandboxed execution, human approvals, per-node audit trails, identity/RBAC, tenant isolation, deployment provenance, and durable dispatch. The current milestone focuses on closing the gap between this architecture and a production-viable platform.

## Core Value

Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## Current Milestone: v1.1 Production Readiness

**Goal:** Close every gap between Zeroth's architecture and a production-viable governed AI workflow platform — real LLM providers, economic control via Regulus, reliable infrastructure, hardened governance, and deployable operations.

**Target features:**
- Real LLM provider adapters (OpenAI, Anthropic) replacing test stubs
- Regulus SDK integration for token metering, cost attribution per node/run/tenant, budget enforcement
- Production storage backend (Postgres) replacing SQLite-only persistence
- Real message queue integration for durable distributed dispatch
- External memory connectors (Redis, vector store) replacing in-memory implementations
- Hardened container-based sandbox backend (finish Phase 8A)
- Approval escalation and SLA timeout policies
- Provider-aware retry with exponential backoff and model fallback
- Containerized deployment (Dockerfile, docker-compose, config management)
- TLS/HTTPS support
- API versioning
- Webhook/callback notifications for run completion and approval events
- OpenAPI spec generation
- Readiness/liveness health probes with dependency checks
- Horizontal worker scaling support

## Requirements

### Validated

- ✓ Governed workflow graph modeling, validation, and versioning exist — phases 1-1F
- ✓ Runtime orchestration, approvals, memory, and deployment-bound service APIs exist — phases 2-5
- ✓ Identity, governance evidence, runtime hardening, and durable control-plane foundations exist — phases 6-9
- ✓ GovernAI dependency pinned to GitHub v0.3.0-dev (memory, secrets, capabilities, agent specs, tool manifests)
- ✓ Unified pydantic-settings config (YAML + env vars) and async Postgres storage backend — Phase 11
- ✓ Real LLM provider adapters (OpenAI, Anthropic via LiteLLM), retry with exponential backoff/jitter, token usage in audit records — Phase 12
- ✓ Regulus economics integration: cost event emission per LLM call, cost attribution in audit records, budget enforcement, cost REST endpoints — Phase 13
- ✓ External memory connectors (Redis KV, Redis thread, pgvector, ChromaDB, Elasticsearch) bridged to GovernAI v0.3.0 protocol with ScopedMemoryConnector + AuditingMemoryConnector wrapping — Phase 14
- ✓ Container sandbox sidecar architecture: Docker socket isolated to sidecar service, API container communicates via HTTP — Phase 14
- ✓ Durable webhook notifications (run.completed, run.failed, approval.requested/resolved/escalated) with HMAC-SHA256 signing, exponential backoff retry, and dead-letter store — Phase 15
- ✓ Approval SLA timeout enforcement with configurable escalation (delegate, auto-reject, alert) and double-escalation prevention — Phase 15

### Active
- ✓ Real message queue (ARQ/Redis) for durable distributed dispatch — Phase 16 + Phase 18 wiring
- ✓ Containerized deployment (Dockerfile, docker-compose, config management) — Phase 17
- ✓ API versioning and OpenAPI spec generation — Phase 17
- ✓ Readiness/liveness health probes with dependency checks — Phase 17
- ✓ Horizontal worker scaling support (Postgres SKIP LOCKED + DispatchSettings) — Phase 16 + Phase 18 wiring
- ✓ TLS/HTTPS support — Phase 17
- ✓ Native LLM API parity: tool schemas, structured output, model parameters, MCP server integration — Phase 19

### Out of Scope

- Studio UI — deferred to a separate milestone after production backend is solid
- Mobile apps — web-based platform only
- Judge/evaluation subsystem — preserved as extension point per original PLAN.md
- Replacing runtime/control-plane internals — this milestone hardens, not rewrites

## Context

The repository is a mature Python backend (280+ tests, lint clean) with domain packages under `src/zeroth/`, broad pytest coverage under `tests/`, and phase-oriented planning in `phases/` and `PROGRESS.md`. GovernAI v0.3.0-dev is now pinned from GitHub with new memory, secrets, capability policy, agent spec, and tool manifest modules. An adjacent project, Regulus (`/Users/dondoe/coding/regulus`), provides a Python SDK (`econ_instrumentation`) for LLM economics telemetry with OpenAI/Anthropic/LangChain/LangGraph auto-instrumentation, a FastAPI backend for cost/value analysis, and tenant-scoped pricing catalogs.

## Constraints

- **Tech stack**: Python/FastAPI/Pydantic backend — all new work integrates with existing foundation
- **GovernAI**: Pinned to git+https://github.com/rrrozhd/governai.git@7452de4 — use new v0.3.0 modules where applicable
- **Regulus**: SDK-level integration preferred — Regulus backend runs as companion service, not embedded
- **Backward compatibility**: Existing 280 tests must continue passing through all changes
- **Architecture**: Modular monolith — new capabilities are new modules, not separate services (except Regulus backend)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GovernAI pinned to GitHub commit, not PyPI | v0.3.0 unreleased but has needed memory/secrets/policy modules | ✓ Good |
| Regulus integrated via SDK, not embedded | Separation of concerns — economics is a companion service | — Pending |
| Postgres as production storage, SQLite retained for dev/test | Production needs vs developer experience | — Pending |
| Studio UI deferred to next milestone | Backend must be production-viable before adding frontend | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-09 after Phase 20 completion — Bootstrap Integration Wiring (MemoryConnectorResolver and BudgetEnforcer wired into AgentRunner dispatch path)*
