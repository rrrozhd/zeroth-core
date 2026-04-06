# Project Research Summary

**Project:** Zeroth v1.1 Production Readiness
**Domain:** Governed multi-agent AI workflow platform — production hardening milestone
**Researched:** 2026-04-06
**Confidence:** HIGH

## Executive Summary

Zeroth is a mature, well-architected governed multi-agent platform that has completed Phases 1-9 with a solid foundation: graph authoring, runtime orchestration, RBAC/tenant isolation, per-node audit chains, subprocess sandbox, and SQLite-backed durable dispatch. The v1.1 milestone is not a greenfield project — it is a targeted production hardening pass on an existing, tested codebase. Research across stack, features, architecture, and pitfalls converges on a single conclusion: the existing abstraction boundaries (ProviderAdapter, MemoryConnector, SQLiteDatabase interface, LeaseManager) are already correctly designed, and the v1.1 work is primarily a matter of providing production-grade implementations behind those boundaries, not redesigning the architecture.

The recommended approach is to ship in dependency order: unified config and Postgres storage first (everything downstream depends on them), then real LLM providers and Regulus economics integration, then memory connectors and Docker sandbox hardening, then webhooks and approval SLA, then distributed dispatch, and finally deployment packaging and health probes. This sequence ensures every phase delivers a working, tested increment without breaking the 280-test suite. The key architectural insight is that all new capabilities slot into existing protocol boundaries as plugins or decorators — the orchestrator, runner, and repository interfaces remain stable throughout.

The critical risks are mechanical and well-understood: avoid converting the synchronous repository layer to async (which breaks all 280 tests), avoid letting the Regulus telemetry SDK auto-instrument at the wrong layer (which corrupts the digest-chained audit trail), resolve the GovernAI local-path dependency before writing any Dockerfile, and design webhook delivery as durable at-least-once (not fire-and-forget). None of these risks require architectural rethinking — they require discipline in sequencing and explicit design decisions before implementation begins.

---

## Key Findings

### Recommended Stack

The existing stack (FastAPI, Pydantic, Redis, PyJWT, httpx, GovernAI) requires no replacement. Seven new libraries cover the entire gap: `sqlalchemy>=2.0.49` + `asyncpg>=0.31.0` + `alembic>=1.18.4` for Postgres (with the critical note to keep the repository layer **synchronous** using psycopg2/psycopg3 sync), `pgvector>=0.4.2` for vector memory, `arq>=0.26` for distributed dispatch, `tenacity>=9.1.4` for retry, `docker>=7.1.0` for container sandbox, and `structlog>=25.0` for production logging. The Regulus SDK (`econ-instrumentation-sdk`) is a local path dep. LangChain-openai and langchain-anthropic pin explicitly in pyproject.toml (already transitive through GovernAI, but need explicit pinning for version clarity).

**Core technologies:**
- `langchain-openai >= 0.3.12` / `langchain-anthropic >= 0.3.0`: LLM provider SDKs wrapping GovernedLLM — explicit pins ensure version clarity over GovernAI's transitive dependency
- `sqlalchemy >= 2.0.49` + `asyncpg >= 0.31.0` + `alembic >= 1.18.4`: Postgres backend for production storage — replaces SQLite write contention bottleneck; repository layer stays synchronous (use psycopg2 or psycopg3 sync, NOT asyncpg, for the repository layer)
- `pgvector >= 0.4.2`: Vector similarity search inside Postgres — avoids a separate vector DB service for MVP
- `arq >= 0.26`: Redis-backed async task queue for distributed worker dispatch — reuses existing Redis dependency, async-native, maps directly to LeaseManager's job model
- `tenacity >= 9.1.4`: Provider-aware exponential backoff with model fallback — wraps ProviderAdapter, not the orchestrator
- `docker >= 7.1.0`: Container sandbox SDK for hardened untrusted execution units — Phase 8A completion
- `structlog >= 25.0`: Structured JSON logging for production log aggregators — replaces unstructured stdlib logging
- `econ-instrumentation-sdk 0.1.1` (local path): Regulus token metering, cost attribution, budget enforcement per tenant/node

**What NOT to add:** Do not bypass GovernedLLM to call providers directly. Do not use `langchain-postgres` (pulls unnecessary transitive deps). Do not use Celery (sync-first, incompatible with asyncio runtime). Do not add a dedicated vector DB service for MVP (pgvector on Postgres is sufficient).

### Expected Features

The platform has a clear P1/P2/P3 priority structure grounded in real feature dependencies. The critical insight from FEATURES.md is that real LLM adapters are the dependency root for the entire economic instrumentation chain — nothing else makes sense to build before them.

**Must have (P1 — platform is not production-viable without these):**
- Real LLM provider adapters (OpenAI, Anthropic) — fills the `DeterministicProviderAdapter` test-only gap; the entire platform is non-functional without this
- Token/cost capture per node invocation — foundation for all economic governance; required before Regulus integration
- Budget enforcement per tenant — spend caps are the first enterprise procurement question; hard caps via GuardrailConfig
- Postgres storage backend — removes SQLite write contention ceiling at ~10 concurrent runs
- Containerized deployment (Dockerfile + docker-compose) — no reproducible deployment exists today
- Readiness/liveness health probes — required by all deployment orchestrators
- TLS/HTTPS (reverse proxy config + documentation) — API keys in plaintext HTTP is unacceptable
- Hardened container sandbox (Docker enforcement, Phase 8A) — untrusted code in LOCAL mode is a critical security gap
- Config management (`ZerothConfig` via pydantic-settings) — force multiplier for all deployment work; eliminates scattered `from_env()` calls
- API versioning (`/api/v1/`) + OpenAPI spec — stable API contracts required for enterprise onboarding

**Should have (P2 — high value, can follow core foundation):**
- Regulus SDK integration — cost attribution per node/run/tenant; unique differentiator no commodity platform offers
- Webhook/callback notifications — push-based run completion and approval events; required for async workflow platforms
- Provider-aware retry with model fallback — resilience; depends on real LLM adapters
- External memory connectors (Redis KV + thread state) — persistent agent memory across restarts/workers
- Approval escalation and SLA timeout policies — closes the human-in-the-loop governance loop
- Horizontal worker scaling validation — validates multi-worker model once Postgres lease backend is live

**Defer (v1.2+ / future):**
- Vector store memory connector — valuable but adds dependency; defer until Redis KV is proven
- Real message queue (Redis Streams) — only needed if Postgres-backed lease dispatch is insufficient; adds operational complexity without eliminating reliability risk
- Streaming LLM responses — only needed if Studio UI requires interactive sessions; architectural change
- Studio UI / authoring frontend — Phase 10 work; backend must be production-stable first
- gVisor / Kata Containers, Kafka, Judge/evaluation subsystem, multi-region deployment — all out of scope for v1.1

### Architecture Approach

Zeroth's modular monolith architecture is well-positioned for v1.1 additions. All new capabilities integrate via existing protocol boundaries — `ProviderAdapter`, `MemoryConnector`, `SecretProvider` — without touching the orchestrator or repository interfaces. The key architectural pattern is the **decorator stack**: new capabilities (retry, instrumentation) wrap provider adapters in bootstrap rather than modifying the orchestrator. The `InstrumentedProviderAdapter` wraps the `RetryProviderAdapter` which wraps the raw `OpenAIProviderAdapter` or `AnthropicProviderAdapter`, and the orchestrator never knows the difference. Configuration centralization (`ZerothConfig`) enables backend selection (SQLite vs Postgres, in-memory vs Redis memory, poll-loop vs MQ) without conditional logic in domain modules. The two most important architectural decisions are: (1) keep repositories synchronous to preserve all 280 tests, and (2) treat the database as the authoritative queue with the MQ as a notification supplement only.

**Major components:**
1. `config.py` (`ZerothConfig`) — unified pydantic-settings configuration; replaces scattered `from_env()` calls; drives all backend selection
2. `llm_providers/` — concrete `OpenAIProviderAdapter`, `AnthropicProviderAdapter`, `RetryProviderAdapter`; plug into existing `ProviderAdapter` protocol
3. `econ/` — Regulus SDK glue: `EconClient`, `InstrumentedProviderAdapter` decorator, `BudgetEnforcer` with TTL cache
4. `storage/postgres.py` — `PostgresDatabase` implementing same interface as `SQLiteDatabase`; synchronous via psycopg2/psycopg3 sync
5. `webhooks/` — durable outgoing webhook dispatch with dead-letter store; triggered by orchestrator and approval service
6. `health/` — readiness/liveness probe logic checking DB, Redis, and optional Regulus

### Critical Pitfalls

1. **Async repository rewrite breaks 280 tests** — Never add `async def` to a repository method. Use psycopg2 or psycopg3 in sync mode behind `PostgresDatabase`. The async boundary stays at the orchestrator layer, exactly where it is today. Any asyncpg import in `src/zeroth/storage/` is a red flag.

2. **Regulus auto-instrumentation corrupts the audit chain** — Do NOT call `econ_instrumentation.auto_instrument()` or any monkey-patching entry point. Instrument explicitly at `AgentRunner.run()` return by reading `ProviderResponse.usage` and emitting a single span. Validate with `AuditContinuityReport` after every Regulus integration change.

3. **GovernAI local path blocks Docker build** — The `governai @ file:///Users/dondoe/coding/governai` pin in pyproject.toml is a hard blocker for `docker build`. Change to `governai @ git+https://github.com/rrrozhd/governai.git@7452de4` before writing any Dockerfile. This is a prerequisite, not part of the containerization phase.

4. **Message queue replacing lease durability** — The database is the authoritative queue. Adding a MQ (Redis Pub/Sub or Streams) is a wakeup notification only — workers still claim runs via `LeaseManager.claim_pending()`. Never design a path where `ACK`ing a queue message is the commit point for run ownership.

5. **Webhook fire-and-forget drops consequential events** — `asyncio.create_task(httpx.post(...))` is not at-least-once delivery. Store webhook delivery attempts in a `webhook_deliveries` table with status, retry count, and next-retry timestamp. A background poller retries with exponential backoff and eventually writes to dead-letter. Design the delivery table before writing any HTTP dispatch code.

6. **Docker sandbox with socket mount on API container** — Mounting `/var/run/docker.sock` on the Zeroth API container is a container escape vector. The sandbox worker must be a separate sidecar container. The API container never touches the Docker socket. Design the sandbox sidecar architecture and the Dockerfile together — not sequentially.

7. **API versioning breaks existing paths** — Do not use `fastapi-versioning` or `VersionedFastAPI` wrappers that remove unversioned routes. Register a new `APIRouter` under `/v1/` prefix alongside existing unversioned routes. Keep unversioned paths as aliases during the transition. All existing `tests/service/` tests must pass with zero URL changes.

---

## Implications for Roadmap

Research from all four files converges on the same 7-phase dependency sequence. The order is load-bearing — skipping ahead causes integration failures.

### Phase 1: Foundation — Config + Postgres Storage
**Rationale:** `ZerothConfig` is required by every subsequent phase (all new features need new env vars, and all backend selection goes through config). Postgres storage removes the SQLite write contention bottleneck that caps all production scaling. These two are co-dependent: Postgres backend selection is driven by config. All 280 tests must remain green throughout — the test harness continues using the SQLite backend.
**Delivers:** Unified pydantic-settings configuration; `PostgresDatabase` with same interface as `SQLiteDatabase` (synchronous, psycopg2/psycopg3); `RunRepository` and `LeaseManager` targeting Postgres when configured; Alembic migrations for high-contention tables.
**Addresses:** Postgres storage backend (P1), Config management (P1)
**Avoids:** Async repository rewrite pitfall (Pitfall 1); keep psycopg2/psycopg3 sync

### Phase 2: Real LLM Providers + Retry
**Rationale:** Core platform value — all economic instrumentation and governance depends on real provider calls returning real token usage. Retry is essential for production reliability and must be designed before Regulus integration (so the retry telemetry contract is established first). Provider tests must be gated behind `@pytest.mark.live` + `ZEROTH_LIVE_TESTS` env var before writing the first live adapter.
**Delivers:** `OpenAIProviderAdapter`, `AnthropicProviderAdapter`, `RetryProviderAdapter` with exponential backoff and model fallback; `TokenUsage` field on `ProviderResponse`; token usage propagated to `NodeAuditRecord`; `@pytest.mark.live` test convention established.
**Addresses:** Real LLM provider adapters (P1), Provider-aware retry (P2), Token/cost capture (P1 foundation)
**Avoids:** Test suite contamination (Pitfall 4), Retry double-counting tokens (Pitfall 3)

### Phase 3: Regulus Economics Integration
**Rationale:** Depends on Phase 2 (needs real `TokenUsage` from provider responses). The `InstrumentedProviderAdapter` decorator pattern keeps Regulus instrumentation fully decoupled from both the LLM adapters and the orchestrator. Budget enforcement is pre-execution (GuardrailConfig), with a TTL-cached budget state to avoid per-node synchronous HTTP calls to the Regulus backend.
**Delivers:** `econ/` module with `EconClient`, `InstrumentedProviderAdapter`, `BudgetEnforcer` (TTL cache); token cost telemetry flowing to Regulus backend; budget enforcement per tenant; GovernAI local path prerequisite changed to git ref.
**Addresses:** Regulus SDK integration (P2), Budget enforcement per tenant (P1), Token/cost capture (P1)
**Avoids:** Auto-instrumentation corrupting audit chain (Pitfall 2), Synchronous budget checks per node (Architecture anti-pattern 4)

### Phase 4: Memory Connectors + Container Sandbox Hardening
**Rationale:** Independent from Phases 2-3; can be developed in parallel after Phase 1 provides config. Redis is already a stack dependency — `RedisKeyValueConnector` and `RedisThreadConnector` are low-complexity adds. Docker sandbox hardening (Phase 8A completion) must be designed together with the Dockerfile (Phase 7) — specifically the sandbox sidecar architecture — so do the architectural design here even if full deployment packaging lands later.
**Delivers:** `RedisKeyValueConnector`, `RedisThreadConnector` implementing `MemoryConnector` protocol; fix `AgentRunner` default thread store to `RepositoryThreadStateStore`; Docker sandbox made default for `UNTRUSTED` manifests via `ZerothConfig.sandbox.require_docker`; sandbox sidecar architecture documented.
**Addresses:** External memory connectors (P2), Hardened container sandbox (P1 — Phase 8A)
**Avoids:** Docker sandbox container escape (Pitfall 5) — sidecar architecture designed here

### Phase 5: Webhooks + Approval SLA
**Rationale:** Depends on stable run lifecycle (Phases 1-2). Webhook delivery must be durable (database-backed delivery table with retry) — design the `webhook_deliveries` table before writing any HTTP dispatch code. Approval SLA escalation naturally depends on webhook delivery (escalation fires a webhook). These two belong in the same phase.
**Delivers:** `webhooks/` module with durable `webhook_deliveries` table, `WebhookNotifier` (async, retry, dead-letter), webhook subscription CRUD API; `WebhookNotifier` wired into `RuntimeOrchestrator` and `ApprovalService`; approval SLA timeout and escalation policy.
**Addresses:** Webhook/callback notifications (P2), Approval escalation/SLA (P2)
**Avoids:** Fire-and-forget webhook delivery (Pitfall 8)

### Phase 6: Distributed Dispatch (MQ + Horizontal Scaling)
**Rationale:** Depends on Phase 1 (Postgres lease backend must be live before MQ supplements it). The MQ is a wakeup notification mechanism only — the database lease remains authoritative for durability and crash recovery. ARQ on existing Redis is the lowest-friction choice. Fix in-memory metrics aggregation in this phase (Redis-backed counters) before advertising multi-worker support.
**Delivers:** `MQPublisher` / `MQConsumer` in `dispatch/` (Redis Pub/Sub or ARQ); `MQRunWorker` subclass; `ZerothConfig.mq.backend` drives backend selection; Redis-backed metrics aggregation (replaces in-process dict); horizontal worker scaling validated with concurrent workers under Postgres.
**Addresses:** Horizontal worker scaling (P2), Real message queue (foundation for P3)
**Avoids:** MQ replacing lease durability (Pitfall 6), In-memory metrics masking multi-worker state (Pitfall 10)

### Phase 7: Deployment Packaging + API + Health
**Rationale:** Prerequisites: GovernAI git ref fix (Pitfall 9) must land before Phase 3 or this phase — whichever comes first; it is a single-line pyproject.toml change. API versioning uses the dual-router approach (existing paths stay alive). Docker sandbox sidecar architecture (designed in Phase 4) is implemented here in docker-compose.
**Delivers:** `Dockerfile` (multi-stage, non-root, health check); `docker-compose.yml` (zeroth + postgres + redis + regulus sidecar + sandbox sidecar); `.env.example`; `/api/v1/` prefix with existing unversioned paths as aliases; OpenAPI spec verification and metadata; `health/` module with `/health/ready` (DB + Redis + dependencies) and `/health/live`; TLS reverse proxy config (Nginx or Traefik).
**Addresses:** Containerized deployment (P1), Health probes (P1), TLS/HTTPS (P1), API versioning + OpenAPI (P1)
**Avoids:** GovernAI local path blocking Docker build (Pitfall 9), API versioning breaking existing paths (Pitfall 7)

### Phase Ordering Rationale

- Config and Postgres come first because every other phase depends on `ZerothConfig` for env var wiring and Postgres for write-safe concurrent storage.
- Real LLM providers come second because the entire economic instrumentation chain (Regulus, budget enforcement, token capture) requires real provider responses — fake providers have no usage data.
- Regulus integration comes third because it wraps the real providers (Phase 2) via the `InstrumentedProviderAdapter` decorator.
- Memory connectors and sandbox hardening are independent and can follow Phase 1 in parallel with Phases 2-3.
- Webhooks and approval SLA come fifth because they depend on a stable run lifecycle (Phases 1-2) but not on Regulus.
- Distributed dispatch comes sixth because it depends on the Postgres lease backend (Phase 1) and benefits from Postgres being battle-tested in production.
- Deployment packaging comes last (with the GovernAI git ref fix as a prerequisite that should land before Phase 3 at the latest).

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Regulus Integration):** The Regulus SDK's `TelemetryTransport` background thread behavior under asyncio event loop shutdown needs careful validation. Verify buffer flush behavior on SIGTERM before planning implementation tasks.
- **Phase 4 (Container Sandbox):** The sandbox sidecar architecture (separate container, restricted API surface) is not a well-documented pattern — needs design spike before implementation tasks are written.
- **Phase 6 (Distributed Dispatch):** ARQ version not PyPI-verified in this research session (MEDIUM confidence). Verify ARQ 0.26 is current and confirm job deduplication semantics match Zeroth's lease model before committing to ARQ.

Phases with standard patterns (skip additional research):
- **Phase 1 (Config + Postgres):** SQLAlchemy 2.0 + psycopg2/psycopg3 sync behind a custom database interface is a well-documented pattern. High confidence.
- **Phase 2 (LLM Providers + Retry):** OpenAI and Anthropic SDK async patterns are well-documented. Tenacity AsyncRetrying is standard. High confidence.
- **Phase 5 (Webhooks):** Database-backed delivery table with background poller is a textbook pattern. High confidence.
- **Phase 7 (Deployment Packaging):** Multi-stage Docker + compose is fully standardized. High confidence.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All version numbers verified via PyPI (April 2026); GovernAI and Regulus SDK directly inspected from source; ARQ is MEDIUM (logic-verified, version not PyPI-confirmed in this session) |
| Features | HIGH | Primary context from direct codebase inspection (`src/zeroth/`); Regulus SDK schemas directly inspected; competitive analysis from MEDIUM-confidence secondary sources |
| Architecture | HIGH | Based on direct source inspection of both Zeroth and Regulus codebases; integration patterns code-verified against actual module interfaces |
| Pitfalls | HIGH | Codebase inspection identified specific fragile points (774-line orchestrator, 280-test suite, digest-chained audit); official docs confirm asyncpg/pgbouncer and asyncio context propagation gotchas |

**Overall confidence:** HIGH

### Gaps to Address

- **ARQ version verification:** ARQ 0.26 was logic-verified as the right choice but not confirmed as the current PyPI release. Verify before adding to pyproject.toml in Phase 6 planning.
- **Sandbox sidecar API design:** The sidecar architecture for Docker sandbox in composed deployment is recommended but not yet designed. This needs a design spike in Phase 4 planning before implementation tasks can be written.
- **Regulus `TelemetryTransport` shutdown behavior:** The Regulus SDK uses a background daemon thread for telemetry. Whether it flushes in-flight events on process SIGTERM is unverified in this research session. Validate before Phase 3 implementation to avoid telemetry loss at container shutdown.
- **GovernAI git ref:** The git+https URL `git+https://github.com/rrrozhd/governai.git@7452de4` is the inferred correct form for the local path dep. Verify it resolves before the Dockerfile phase (or Phase 3, whichever comes first).
- **psycopg2 vs psycopg3 sync for PostgresDatabase:** Both are valid for the synchronous repository layer. psycopg3 (psycopg) is the more actively maintained library. Choose one and pin it explicitly before Phase 1 planning.

---

## Sources

### Primary (HIGH confidence)
- Zeroth codebase direct inspection: `src/zeroth/` (April 2026) — all module interfaces, 280-test suite structure, CONCERNS.md
- Regulus SDK source: `/Users/dondoe/coding/regulus/sdk/python/econ_instrumentation/` — `__init__.py`, `schemas.py`, `client.py`, `transport.py`, `integrations/`
- GovernAI source: `/Users/dondoe/coding/governai/governai/integrations/llm.py` — GovernedLLM implementation and LangChain deps
- PyPI: asyncpg 0.31.0, SQLAlchemy 2.0.49, alembic 1.18.4, pgvector 0.4.2, tenacity 9.1.4, openai 2.30.0, anthropic 0.89.0 — versions current as of April 2026
- Docker SDK docs (docker-py.readthedocs.io) — version 7.1.0
- asyncpg FAQ — pgbouncer prepared statement issues (magicstack.github.io)
- OpenTelemetry asyncio context propagation docs

### Secondary (MEDIUM confidence)
- WebSearch: ARQ vs Celery vs Dramatiq async FastAPI 2025 comparison
- WebSearch: structlog FastAPI production 2025-2026
- WebSearch: nsjail vs Docker vs gVisor sandbox comparison 2025
- Portkey.ai: Retries, fallbacks, and circuit breakers in LLM apps
- Hookdeck: Webhook Infrastructure Guide
- Medium: Versioning REST APIs in FastAPI Without Breaking Old Clients
- Dev.to: Building Hierarchical Budget Controls for Multi-Tenant LLM Gateways
- How to sandbox AI agents in 2026 (northflank.com)

---

*Research completed: 2026-04-06*
*Ready for roadmap: yes*
