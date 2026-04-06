# Feature Research

**Domain:** Governed multi-agent AI workflow platform (production readiness milestone)
**Researched:** 2026-04-06
**Confidence:** HIGH (primary context from codebase + Regulus SDK inspection; supplemented by current web research)

---

## Context: What Already Exists

Zeroth has a mature Phases 1-9 foundation. This research covers only the production-readiness gap (v1.1 milestone). The existing capability set includes: graph authoring/versioning, runtime orchestration with loop guards, agent runtime with ProviderAdapter protocol, subprocess/tempdir sandbox, human approval system, per-node audit trail (digest-chained), identity/RBAC/tenant isolation, SQLite-backed durable dispatch (lease-based), Prometheus-compatible metrics, correlation IDs, and admin controls.

---

## Feature Landscape

### Table Stakes (Platform is Not Production-Viable Without These)

Features that are non-negotiable for a governed AI workflow platform to be called "production-ready." Operators and buyers will reject the platform outright if any of these are absent.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Real LLM provider adapters (OpenAI, Anthropic)** | Platform is worthless without real AI calls. `DeterministicProviderAdapter` is test-only. `ProviderAdapter` protocol already exists — this is filling it in. | MEDIUM | OpenAI and Anthropic Python SDKs are stable and well-documented. The `GovernedLLMProviderAdapter` wrapper exists; need concrete classes that resolve credentials via `SecretProvider` and return `ProviderResponse`. OpenAI SDK has built-in `max_retries` and exponential backoff; Anthropic SDK similarly. Token usage fields (`prompt_tokens`, `completion_tokens`) are in standard SDK responses. |
| **Token/cost capture per node invocation** | Without recording token usage, there is no cost attribution, no budget enforcement, and no accountability. This is the economic instrumentation foundation everything else rests on. | LOW-MEDIUM | `ProviderResponse` model must gain `usage` fields (prompt_tokens, completion_tokens, cost_usd). Regulus `ExecutionEvent` schema already defines `token_cost_usd`, `latency_ms`, `capability_id`. Capture at `AgentRunner.run()` time, write to audit record and emit to Regulus SDK. |
| **Budget enforcement per tenant/deployment** | Without spend caps, a misbehaving workflow can generate unbounded API costs. This is the first question enterprise procurement asks. | MEDIUM | Implemented as a pre-flight check in `GuardrailConfig` (analogous to existing `QuotaEnforcer`). Requires Regulus SDK `track_execution` + Regulus backend budget query, or a local spend accumulator in `RunRepository`. Hard caps block run creation; soft caps log warnings. Per-tenant, per-deployment granularity required. |
| **Production storage backend (Postgres)** | SQLite write contention blocks horizontal scaling. Single-writer SQLite caps throughput at ~50-200 writes/second — unacceptable under concurrent multi-run workloads. | HIGH | Zeroth's custom `SQLiteDatabase` + raw SQL pattern means no ORM migration: repository classes need a `PostgresDatabase` alternative that shares the same `transaction()` / `apply_migrations()` interface. asyncpg + SQLAlchemy 2.0 async engine is the standard FastAPI/Postgres stack in 2025-2026. Keep SQLite for dev/test; Postgres is the production default. Critical-path tables first: runs, run_checkpoints, leases, audit, rate_limit_buckets, quotas. Graph/contract/deployment tables can follow. |
| **Hardened container sandbox (Phase 8A completion)** | Executing untrusted LLM-generated code in LOCAL mode with no resource constraints is a critical security gap. Without Docker enforcement, the platform cannot run production executable units safely. | MEDIUM | `DockerSandboxConfig` and `SandboxManager` already exist. Phase 8A is explicitly tracked as incomplete. Need: make Docker the default strictness mode for `UNTRUSTED` manifests; enforce CPU/memory limits via Docker flags; add network isolation (no-network by default); complete `AdmissionController` digest verification end-to-end. Do not introduce Kata/gVisor — Docker is sufficient for the milestone. |
| **Containerized deployment (Dockerfile + docker-compose)** | No Dockerfile exists. Platform cannot be deployed reproducibly without one. Every production environment needs this. | LOW | Single-stage or two-stage Python Dockerfile targeting Python 3.12. `docker-compose.yml` with zeroth (uvicorn), postgres, redis services. `.dockerignore` to exclude dev artifacts. Environment variable documentation for all required vars. Non-root user, health check in Dockerfile. |
| **Readiness and liveness health probes** | Kubernetes, docker-compose healthchecks, and load balancers all need `/health/live` and `/health/ready`. Missing probes = orchestrator cannot detect crashed workers. | LOW | `GET /health/live` — returns 200 if process is up. `GET /health/ready` — checks database connectivity, Redis ping, and Postgres connectivity; returns 200 only if all dependencies are responding. Already have a basic health endpoint; needs dependency-aware readiness check. |
| **TLS/HTTPS support** | API keys and bearer tokens travel in plaintext over HTTP today. Unacceptable for any production deployment. | LOW | Zeroth should not terminate TLS itself (that's a reverse proxy concern) but must document the requirement and provide a Caddy or Nginx config in the docker-compose. Optionally expose uvicorn SSL args via config. CONCERNS.md flags this explicitly. |
| **Webhook/callback notifications** | Long-running workflows complete asynchronously. Callers need push notifications for run completion and approval events rather than polling. This is table stakes for async workflow platforms (every competitor — LangSmith, Temporal, Prefect — provides this). | MEDIUM | `WebhookConfig` per deployment or per run: URL, secret (HMAC-SHA256 signing), event types (run.completed, run.failed, approval.pending, approval.resolved). Delivery via httpx async POST with retry. Dead-letter on repeated failure. Signed payload for security. |
| **API versioning** | Without explicit versioning, any API change breaks existing callers. Production platforms must version their API surface. | LOW | URL-based versioning (`/api/v1/`) is the standard FastAPI pattern. FastAPI router prefixes make this mechanical. Requires mounting existing routes under `/api/v1/`. OpenAPI spec auto-generates correct versioned docs. |
| **OpenAPI spec generation** | Required for SDK generation, documentation, client integration, and enterprise onboarding. FastAPI generates this automatically — it just needs to be exposed and not disabled. | LOW | FastAPI already generates `/openapi.json`. Needs: correct metadata (title, version, contact), schema names cleaned up, security scheme documented (API key + bearer), spec cached, and a `/docs` endpoint enabled or explicitly disabled per environment. |

### Differentiators (Competitive Advantage)

Features that distinguish Zeroth from commodity alternatives. These advance the "governed, accountable, economically-controlled" value proposition.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Regulus SDK integration (cost attribution per node/run/tenant)** | Zeroth becomes the only governed agent platform with first-class LLM economics — not just token counts but cost attribution at node granularity with AER (Agent Efficiency Ratio). This is a unique capability no commodity platform offers. | MEDIUM | Regulus `econ_instrumentation` SDK already has `track_execution()`, `ExecutionEvent`, `instrument_openai_client()` / `instrument_anthropic_client()` (auto-instrumentation hooks). Integration: call `configure()` at bootstrap, emit `ExecutionEvent` after each agent node completes, join on `run_id` as `join_key`. Cost flows to Regulus backend (companion service). Zeroth stores `cost_usd`, `prompt_tokens`, `completion_tokens` locally in audit records for governance evidence. |
| **Provider-aware retry with model fallback** | Most platforms have basic retry. Zeroth can route on failure: primary model fails → cheaper fallback model → different provider. Tied to economic governance (fallback to cheaper model is a cost decision). | MEDIUM | `RetryPolicy` model already exists in `src/zeroth/agent_runtime/models.py`. Needs: provider error classification (rate-limit vs auth vs transient), per-provider backoff timing (OpenAI: respect Retry-After header; Anthropic 529: ~120s backoff), fallback chain in `AgentNode.model_provider` config. Disable SDK-level retries when using fallback chains (set `max_retries=0` on SDK client, handle in `AgentRunner`). |
| **External memory connectors (Redis KV, vector store)** | In-memory connectors lose state on restart and cannot be shared across workers. Redis-backed and vector-backed memory enables persistent agent memory — a key differentiator for long-running or multi-turn workflows. | MEDIUM | `MemoryConnector` protocol exists. Need concrete implementations: `RedisKeyValueConnector` (redis-py, already a dependency), `RedisThreadStateStore` (for conversation threads). Vector connector (`PgvectorConnector` or `QdrantConnector`) for semantic recall. CONCERNS.md explicitly flags in-memory connectors as a production gap. Redis is already in the stack — KV connector is LOW complexity. Vector store is MEDIUM (new dependency). |
| **Approval escalation and SLA timeout policies** | Production approval gates need enforcement: if nobody approves within N minutes, escalate or auto-reject. This closes the human-in-the-loop governance loop. Current implementation has no timeout handling. | MEDIUM | New fields on `HumanApprovalNode`: `sla_seconds`, `escalation_policy` (escalate_to, auto_approve, auto_reject). Background worker task (or scheduled poll in `RunWorker`) detects expired approvals and resolves them. Requires new `approval_sla` table or TTL field on `ApprovalRepository`. |
| **Horizontal worker scaling support** | Multiple worker processes competing for runs via lease-based claim is already architecturally designed for. Postgres moves the write-contention bottleneck away from SQLite. This makes horizontal scaling actually viable. | LOW (depends on Postgres) | The `LeaseManager` pattern is already sound. With Postgres as the lease backend, multiple workers on separate processes/containers can compete safely. Document the deployment model; add `WORKER_ID` env var; verify lease claim atomicity against Postgres. No new abstractions needed. |
| **Real message queue integration** | Redis Streams or a dedicated queue (RQ/Celery) can replace the SQLite-backed poll loop for distributed dispatch. This unlocks cross-host worker topology. | HIGH | This is explicitly in PROJECT.md requirements but carries high complexity. The current SQLite lease model already achieves reliable dispatch within a single host. Postgres lease model achieves multi-host dispatch without a dedicated queue. A real MQ (Redis Streams, RabbitMQ) adds operational complexity (new service, DLQ handling, exactly-once semantics). Recommend: ship Postgres-backed lease as the distributed dispatch mechanism first; defer true MQ to v1.2. |
| **Config management (unified, environment-aware)** | No unified config surface. Env var JSON blobs are operational debt. Pydantic Settings with `.env` file support, environment profiles (dev/prod), and startup validation is a force-multiplier for all deployment work. | LOW | Use `pydantic-settings` (already in FastAPI ecosystem). Single `ZerothConfig` Pydantic model with all env vars, loaded once at bootstrap. `.env.example` committed. CONCERNS.md flags this explicitly. Low effort, high leverage. |

### Anti-Features (Do Not Build in This Milestone)

Features that seem logical to add but are out of scope, would create scope creep, or carry risks that outweigh their value at this stage.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Studio UI / authoring frontend** | Workflow authors want a visual canvas. Phase 10 work exists. | Backend is not production-viable yet. Building UI on top of unstable infrastructure is wasteful rework. PROJECT.md explicitly defers this. | Complete this milestone first; Studio UI is the next milestone. |
| **Kafka / full message broker (RabbitMQ)** | Teams may want high-throughput distributed dispatch | Adds an entirely new operational service, DLQ handling, partition management, and monitoring surface. The lease-over-Postgres pattern achieves the same reliability for the workload volumes Zeroth targets. | Postgres-backed lease dispatch solves multi-worker scaling without new services. |
| **gVisor / Kata Containers** | Stronger isolation than Docker for untrusted code | Requires Kubernetes or custom container runtime. Operational complexity far exceeds the milestone scope. Phase 8A with Docker enforced is the correct scope. | Docker with resource constraints, no-network flag, and read-only filesystem covers the threat model for v1.1. |
| **Judge/evaluation subsystem** | Automated quality scoring of agent outputs | Out-of-scope per PROJECT.md ("preserved as extension point"). Adds significant ML infrastructure complexity. | The audit trail + human approval system provides governance evidence today. Evaluation is v2+. |
| **Multi-region / global deployment** | Enterprise customers may want geo-redundancy | Far exceeds the single-region production milestone. Requires distributed state management and data replication. | Horizontal scaling within a single region is the correct scope. |
| **Custom LLM provider plugins (arbitrary HTTP)** | Teams want to use local/custom models | Adds protocol abstraction complexity. OpenAI-compatible endpoints (including LiteLLM, Ollama) cover the majority of cases since they expose the OpenAI API shape. | OpenAI adapter with configurable `base_url` covers local/custom models via LiteLLM or Ollama. |
| **Streaming LLM responses to HTTP clients** | Real-time token streaming for interactive use | Zeroth's workflow model is step-completion oriented, not streaming. Streaming requires significant changes to the run model, API layer, and client contract. Not needed for batch/automation workflows. | Polling `/runs/{id}` with status updates is sufficient for v1.1. Streaming is a v2 feature if Studio UI needs it. |
| **Built-in secrets manager (Vault, AWS Secrets Manager)** | Secure credential storage | Out of scope for the milestone. The `SecretProvider` protocol exists; adding an `EnvSecretProvider` that reads from a well-managed `.env` is sufficient. | Document that operators should inject secrets via environment variables from their existing secrets manager (Vault, AWS SM, etc.). |

---

## Feature Dependencies

```
Real LLM provider adapters (OpenAI/Anthropic)
    └──required-by──> Token/cost capture per node
                          └──required-by──> Regulus SDK integration
                          └──required-by──> Budget enforcement per tenant

Postgres storage backend
    └──enables──> Horizontal worker scaling
    └──enables──> Real message queue (if ever needed; else Postgres lease is sufficient)
    └──resolves──> SQLite write contention bottleneck

Config management (ZerothConfig)
    └──required-by──> Containerized deployment
    └──required-by──> TLS/HTTPS documentation
    └──simplifies──> All env var wiring for new features

Containerized deployment (Dockerfile + docker-compose)
    └──required-by──> Readiness/liveness health probes (needs container healthcheck)
    └──required-by──> TLS/HTTPS (nginx/Caddy in compose stack)
    └──enables──> Horizontal worker scaling (multiple worker containers)

API versioning (/api/v1/)
    └──required-by──> OpenAPI spec generation (versioned paths in spec)

Hardened container sandbox (Docker enforcement)
    └──no-dependencies──> Can be completed independently

Approval escalation/SLA
    └──depends-on──> Webhook notifications (escalation often fires a webhook)

External memory connectors (Redis KV)
    └──depends-on──> Postgres (if using PG-backed threads)
    └──independent-of──> LLM provider adapters (MemoryConnector protocol is provider-agnostic)

Provider-aware retry with model fallback
    └──depends-on──> Real LLM provider adapters (needs real provider errors to handle)
```

### Dependency Notes

- **Token capture requires real LLM adapters:** You cannot capture usage tokens without making real API calls. Fake providers have no usage data.
- **Regulus integration requires token capture:** The `ExecutionEvent` schema needs `token_cost_usd` which requires cost data from real provider responses.
- **Budget enforcement can be local or Regulus-backed:** A local `SpendAccumulator` in the `GuardrailConfig` can enforce budgets independently of Regulus; Regulus provides the analytics layer on top.
- **Postgres unlocks horizontal scaling:** The architectural design already supports multi-worker lease competition. Postgres is the only blocker — SQLite cannot handle concurrent write load from N workers.
- **Config management is a low-effort force multiplier:** All new features require new env vars. A unified `ZerothConfig` model pays for itself immediately.
- **Webhook notifications are independent:** Can be shipped before or after other features. Not on the critical path for any other feature.

---

## MVP Definition

### v1.1 Launch With (Minimum Viable Production)

These features make the platform minimally production-viable. Without all of them, the platform cannot be deployed or trusted in a real environment.

- [ ] **Real LLM provider adapters (OpenAI, Anthropic)** — Platform cannot execute real AI workloads without these. Everything downstream depends on them.
- [ ] **Token/cost capture per node invocation** — Economic instrumentation foundation. Required for Regulus integration and budget enforcement.
- [ ] **Budget enforcement per tenant** — Without spend caps, the platform is economically unsafe for production use.
- [ ] **Postgres storage backend** — Removes the SQLite write contention bottleneck that caps all production scaling.
- [ ] **Containerized deployment (Dockerfile + docker-compose)** — Platform cannot be deployed reproducibly without this.
- [ ] **Readiness/liveness health probes** — Required for all production deployment orchestrators (Kubernetes, docker-compose, ECS).
- [ ] **TLS/HTTPS documentation + reverse proxy config** — API keys in plaintext HTTP is unacceptable in production.
- [ ] **Hardened container sandbox (Docker enforcement, Phase 8A)** — Untrusted code execution in LOCAL mode is a critical security gap.
- [ ] **Config management (ZerothConfig via pydantic-settings)** — Required for clean, validated, environment-aware production deployments.
- [ ] **API versioning (/api/v1/) + OpenAPI spec** — Required for stable API contracts and enterprise onboarding.

### Add After Core is Stable (v1.1 second wave)

Features with high value that can follow the core foundation without blocking it.

- [ ] **Regulus SDK integration** — Adds economic analytics layer. Depends on token capture (above). Companion service can be deployed independently.
- [ ] **Webhook/callback notifications** — Push-based completion events. High value, independent of other features.
- [ ] **Provider-aware retry with model fallback** — Resilience improvement. Depends on real LLM adapters.
- [ ] **External memory connectors (Redis KV + thread state)** — Resolves the in-memory memory gap. Redis is already a stack dependency.
- [ ] **Approval escalation and SLA timeout policies** — Closes the governance loop on human approvals.
- [ ] **Horizontal worker scaling documentation + Postgres lease validation** — Validates the multi-worker model once Postgres is live.

### Future Consideration (v1.2+)

- [ ] **Vector store memory connector** — Semantic recall adds significant value but requires a new dependency (pgvector or Qdrant). Defer until Redis KV is proven.
- [ ] **Real message queue (Redis Streams)** — Only needed if Postgres-backed lease dispatch proves insufficient at scale. Adds operational complexity.
- [ ] **Streaming LLM responses** — Needed if Studio UI requires interactive agent sessions. Architectural change; defer to Studio milestone.

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Real LLM provider adapters | HIGH | MEDIUM | P1 |
| Token/cost capture per node | HIGH | LOW | P1 |
| Budget enforcement per tenant | HIGH | MEDIUM | P1 |
| Postgres storage backend | HIGH | HIGH | P1 |
| Containerized deployment | HIGH | LOW | P1 |
| Readiness/liveness probes | HIGH | LOW | P1 |
| TLS/HTTPS (reverse proxy config) | HIGH | LOW | P1 |
| Hardened container sandbox | HIGH | MEDIUM | P1 |
| Config management | MEDIUM | LOW | P1 |
| API versioning + OpenAPI | MEDIUM | LOW | P1 |
| Regulus SDK integration | HIGH | MEDIUM | P2 |
| Webhook notifications | HIGH | MEDIUM | P2 |
| Provider-aware retry + fallback | MEDIUM | MEDIUM | P2 |
| Redis memory connectors | MEDIUM | MEDIUM | P2 |
| Approval escalation/SLA | MEDIUM | MEDIUM | P2 |
| Horizontal scaling validation | MEDIUM | LOW | P2 |
| Vector store connector | MEDIUM | HIGH | P3 |
| Real message queue (Redis Streams) | LOW | HIGH | P3 |
| Streaming LLM responses | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for v1.1 launch — platform is not production-viable without these
- P2: Should have in v1.1 second wave — high value, can follow core foundation
- P3: Future consideration — defer to v1.2+ or Studio milestone

---

## Competitor Feature Analysis

| Feature | LangSmith Deployment | Temporal | Zeroth Approach |
|---------|----------------------|----------|-----------------|
| LLM provider integration | LangChain abstractions; multi-provider routing | External; user-managed | Native OpenAI/Anthropic adapters + GovernedLLM wrapper; configurable per AgentNode |
| Token/cost tracking | Per-trace token counts; cost estimates | Not built-in | Per-node audit record captures tokens + cost_usd; forwarded to Regulus for analytics |
| Budget enforcement | Spend alerts; no hard enforcement in base product | Not built-in | Hard caps via QuotaEnforcer-style guardrail; soft caps via Regulus alerts |
| Durable dispatch | LangGraph Platform (renamed): durable execution, fault tolerance | Core value proposition | SQLite lease (current) → Postgres lease (v1.1); achieves same reliability without Temporal's operational weight |
| Human approvals | Human-in-the-loop via interrupt patterns | External signal integration | Native HumanApprovalNode with RBAC-controlled resolution + SLA escalation (v1.1) |
| Audit trail | LangSmith traces | Activity logs | Digest-chained per-node audit with tamper detection; compliance evidence bundles |
| Container sandbox | Not a feature | Not a feature | Native ExecutableUnitNode + Docker enforcement — unique in the space |
| Governance / policy | Limited capability enforcement | Not built-in | Native PolicyGuard with capability-based access control per node — Zeroth's key differentiator |

---

## Sources

- Codebase inspection: `src/zeroth/agent_runtime/`, `src/zeroth/guardrails/`, `src/zeroth/dispatch/`, `.planning/codebase/CONCERNS.md`, `.planning/codebase/ARCHITECTURE.md` — HIGH confidence
- Regulus SDK inspection: `econ_instrumentation/__init__.py`, `schemas.py` — HIGH confidence (direct source read)
- [LangSmith Deployment](https://www.langchain.com/langsmith/deployment) — production AI agent platform feature reference — MEDIUM confidence
- [Building Hierarchical Budget Controls for Multi-Tenant LLM Gateways](https://dev.to/pranay_batta/building-hierarchical-budget-controls-for-multi-tenant-llm-gateways-ceo) — budget enforcement patterns — MEDIUM confidence
- [FastAPI API Versioning](https://medium.com/geoblinktech/fastapi-with-api-versioning-for-data-applications-2b178b0f843f) — versioning patterns — MEDIUM confidence
- [How to sandbox AI agents in 2026](https://northflank.com/blog/how-to-sandbox-ai-agents) — container isolation landscape — MEDIUM confidence
- [Ask HN: What's your go-to message queue in 2025?](https://news.ycombinator.com/item?id=43993982) — queue selection rationale — MEDIUM confidence
- [AI Agent Retry Strategies: Exponential Backoff](https://getathenic.com/blog/ai-agent-retry-strategies-exponential-backoff) — retry/fallback patterns — MEDIUM confidence
- [Top 8 AI agent orchestration platforms](https://redis.io/blog/ai-agent-orchestration-platforms/) — ecosystem survey — MEDIUM confidence

---

*Feature research for: Zeroth v1.1 Production Readiness milestone*
*Researched: 2026-04-06*
