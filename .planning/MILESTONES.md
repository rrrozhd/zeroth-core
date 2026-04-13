# Milestones

## v4.0 Platform Extensions for Production Agentic Workflows (Shipped: 2026-04-13)

**Phases completed:** 3 phases, 8 plans, 14 tasks

**Key accomplishments:**

- 1. [Rule 3 - Blocking] Circular import in subgraph/__init__.py
- 1. [Rule 3 - Blocking] Circular import via __init__.py
- 1. [Rule 1 - Bug] Missing explicit WAITING_APPROVAL on nested resume
- RED phase:
- 1. [Rule 1 - Bug] Fixed E501 line-too-long in executor.py
- Six v4.0 concept pages fleshed out from 3-line stubs to 38-44 line substantive docs, OpenAPI spec synced with v4.0 endpoints, README updated with v4.0 section and architecture diagram

---

## v1.1 Production Readiness (Shipped: 2026-04-09)

**Phases:** 11 (Phases 11-21) | **Plans:** 30 | **Timeline:** 4 days
**Stats:** 168 commits, 350 files changed, +47,444 / -3,273 lines

**Key accomplishments:**

- Real LLM provider integration via LiteLLM routing to 100+ models with exponential backoff retry and token usage audit
- Regulus economics: per-call cost events, per-tenant budget enforcement with fail-open, cost REST endpoints
- 8 memory connector types (Redis KV/thread, pgvector, ChromaDB, Elasticsearch, 3 in-memory) bridged to GovernAI v0.3.0 protocol
- Docker sandbox sidecar architecture with per-execution network isolation
- Durable webhooks with HMAC signing, dead-letter store, and approval SLA escalation policies
- Production deployment: multi-stage Dockerfile, 6-service docker-compose, Nginx TLS termination, health probes
- Native LLM API parity: tool schemas, structured output, model parameters, MCP server connections
- Postgres storage backend with Alembic migrations, ARQ wakeup, horizontal worker scaling via SKIP LOCKED

**Tech debt at completion:** 8 cosmetic items (stale comments, dead code, duplicate fields, documentation staleness) — 0 blockers

**Archive:** `milestones/v1.1-ROADMAP.md`, `milestones/v1.1-REQUIREMENTS.md`, `milestones/v1.1-MILESTONE-AUDIT.md`, `milestones/v1.1-phases/`

---
