# Architecture Research

**Domain:** Governed multi-agent platform — production readiness integration
**Researched:** 2026-04-06
**Confidence:** HIGH (based on direct source inspection of both codebases)

---

## Standard Architecture

### System Overview

Current state (modular monolith, single process):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         HTTP / API Layer (FastAPI)                       │
│  run_api  approval_api  audit_api  contracts_api  admin_api              │
├─────────────────────────────────────────────────────────────────────────┤
│              Bootstrap / Composition Root (ServiceBootstrap)             │
├──────────┬──────────────────────────────────────┬───────────────────────┤
│ Dispatch │        Orchestration Engine           │  Service Domains      │
│  Worker  │  (RuntimeOrchestrator._drive loop)    │  graph / deployments  │
│  Lease   │                                       │  contracts / audit    │
│ Manager  │  agent_runtime  execution_units        │  approvals / runs     │
│          │  conditions     policy                 │  guardrails           │
│          │  mappings       secrets                │  observability        │
│          │  memory                                │                       │
├──────────┴──────────────────────────────────────┴───────────────────────┤
│                         Storage Layer                                    │
│  SQLiteDatabase (WAL)    RedisConfig    GovernAI Redis Stores            │
└─────────────────────────────────────────────────────────────────────────┘
```

Target state (v1.1, still modular monolith — new modules, new backends):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  HTTP / API Layer (FastAPI + versioning)                 │
│  run_api  approval_api  audit_api  contracts_api  admin_api              │
│  webhook_api  health_api (readiness/liveness)                            │
├─────────────────────────────────────────────────────────────────────────┤
│              Bootstrap / Composition Root (ServiceBootstrap)             │
│              + ConfigLoader (pydantic-settings)                          │
├──────────┬───────────────────────────────────────────────────────────────┤
│ Dispatch │        Orchestration Engine                                   │
│  Worker  │  (RuntimeOrchestrator._drive loop)                            │
│  +MQ     │                                                               │
│  Lease   │  agent_runtime  execution_units (+ Docker sandbox)            │
│ Manager  │    + LLM adapters  conditions  policy                         │
│  +PG     │    + Regulus hook  mappings     secrets                       │
│          │    + retry/backoff memory (+ Redis/vector connectors)         │
├──────────┴───────────────────────────────────────────────────────────────┤
│     New Modules       │  Modified Domains   │  External Services          │
│  llm_providers/       │  storage/ (+PG)     │  Regulus backend (econ_plane)│
│  econ/ (Regulus glue) │  dispatch/ (+MQ)    │  OpenAI API                 │
│  webhooks/            │  agent_runtime/     │  Anthropic API              │
│  health/              │  memory/            │  Redis (external)           │
│                       │  execution_units/   │  Postgres                   │
│                       │  service/ (+vers.)  │  Docker daemon              │
│                       │  observability/     │  Message queue (Redis/SQS)  │
├───────────────────────┴─────────────────────┴────────────────────────────┤
│                           Storage Layer                                   │
│  PostgresDatabase  SQLiteDatabase (dev/test)  Redis  VectorStore          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Component Classification: New vs Modified

### New Modules (create from scratch)

| Module | Path | Purpose |
|--------|------|---------|
| `llm_providers` | `src/zeroth/llm_providers/` | Concrete OpenAI and Anthropic `ProviderAdapter` implementations, retry policy, model fallback |
| `econ` | `src/zeroth/econ/` | Regulus SDK glue: `ExecutionEvent` builder, `join_key_context` wiring, budget cap enforcer, tenant cost attribution |
| `webhooks` | `src/zeroth/webhooks/` | Outgoing webhook dispatch for run completion and approval events |
| `health` | `src/zeroth/health/` | Readiness and liveness probe logic with dependency checks (DB, Redis, MQ) |

### Modified Modules (extend existing code)

| Module | What Changes | Scope |
|--------|-------------|-------|
| `storage/` | Add `PostgresDatabase` alongside `SQLiteDatabase`; repository protocol abstraction | Medium — new file, existing repos untouched initially |
| `agent_runtime/` | Add token/usage fields to `ProviderResponse`; add `InstrumentedProviderAdapter` wrapper; fix default thread store | Medium — provider.py + models.py |
| `dispatch/` | Add message queue backend option for distributed workers; `LeaseManager` with Postgres backend | Medium — worker.py + lease.py |
| `memory/` | Add `RedisMemoryConnector` and `VectorMemoryConnector` implementing existing `MemoryConnector` protocol | Low — protocol already defined |
| `execution_units/` | Harden Docker sandbox as default for untrusted units (Phase 8A completion) | Low — sandbox.py |
| `service/` | Add API versioning prefix (`/v1/`), register webhook and health routes, add `webhook_notifier` to bootstrap | Medium — app.py + bootstrap.py |
| `observability/` | Wire real Prometheus/OTEL exporter instead of in-memory only | Low — metrics.py |
| `approvals/` | Add SLA timeout policy, escalation rules | Low — new models + service extension |
| `secrets/` | Add secrets manager integration (AWS SSM / Vault option) alongside `EnvSecretProvider` | Low — provider.py |
| `guardrails/` | Add budget cap enforcement via Regulus backend query | Low — new check in rate_limit flow |

### Infrastructure / Packaging (new files at repo root)

| Artifact | Purpose |
|----------|---------|
| `Dockerfile` | Multi-stage build for production image |
| `docker-compose.yml` | Local dev stack: zeroth + postgres + redis + regulus |
| `.env.example` | Environment variable documentation |
| `src/zeroth/config.py` | Unified pydantic-settings `ZerothConfig` replacing scattered `from_env()` calls |

---

## Integration Points

### 1. Real LLM Providers → agent_runtime

The `ProviderAdapter` protocol is already the correct abstraction. No changes needed to orchestrator or runner — only the adapter implementation changes.

```
agent_runtime/runner.py (AgentRunner)
    → ProviderAdapter.ainvoke(ProviderRequest)
        → [NEW] OpenAIProviderAdapter  — wraps openai.AsyncOpenAI
        → [NEW] AnthropicProviderAdapter  — wraps anthropic.AsyncAnthropic
        → [EXISTING] GovernedLLMProviderAdapter  — kept for GovernAI model refs
```

The `ProviderResponse` model needs a new `usage` field added:

```python
class ProviderResponse(BaseModel):
    ...
    usage: TokenUsage | None = None   # NEW: prompt_tokens, completion_tokens, model
```

`AgentRunner` records this usage into the `NodeAuditRecord` and passes it to the econ module.

### 2. Regulus SDK → econ module → agent_runtime + audit

Regulus uses a fire-and-forget telemetry pattern: `TelemetryTransport` batches events to the Regulus backend over a background thread. Integration is non-blocking.

```
AgentRunner.run()
    → provider.ainvoke(request) → ProviderResponse (with usage)
    → [NEW] econ.record_node_execution(run_id, node_id, tenant_id, usage)
        → econ_instrumentation.track_execution(ExecutionEvent)
            → TelemetryTransport (background thread → Regulus backend HTTP)
    → audit.record_node_execution(...)  [existing, unchanged]
```

The `econ` module is a thin adapter. It maps Zeroth concepts to Regulus concepts:

| Zeroth concept | Regulus concept |
|----------------|-----------------|
| `run_id` | `join_key` |
| `node_id` + `deployment_ref` | `capability_id` |
| `model_provider` + `model_name` | `implementation_id` |
| `tenant_id` | included in `ExecutionEvent.metadata` |
| `TokenUsage` from ProviderResponse | `token_cost_usd` (after pricing lookup) |

Budget enforcement is a separate concern from telemetry. The `guardrails` module is the right home for a pre-execution budget cap check. It calls the Regulus backend `GET /capabilities/{id}/budget` (or uses a locally-cached spend figure) before allowing a node to execute.

The `InstrumentedProviderAdapter` pattern (wrapper around real adapters) is cleaner than patching:

```python
class InstrumentedProviderAdapter:
    """Wraps any ProviderAdapter, capturing usage for Regulus telemetry."""
    def __init__(self, inner: ProviderAdapter, econ_client: EconClient): ...
    async def ainvoke(self, request: ProviderRequest) -> ProviderResponse:
        response = await self.inner.ainvoke(request)
        await self.econ_client.record(request, response)
        return response
```

This keeps LLM providers and Regulus instrumentation fully decoupled.

### 3. Postgres → storage module

The repository pattern is already abstraction-ready. The path of least resistance is:

1. Add `src/zeroth/storage/postgres.py` with a `PostgresDatabase` that exposes the same `transaction()` context manager and `apply_migrations()` interface as `SQLiteDatabase`.
2. Migrate high-contention tables first: `runs`, `run_checkpoints`, `lease_records`, `rate_limit_buckets`, `quota_records`.
3. Keep `GraphRepository`, `ContractRegistry`, `DeploymentRepository` on SQLite (read-heavy, rarely mutated).
4. Add a database selector in `ZerothConfig`: `storage.backend = sqlite | postgres`.

The bootstrap factory (`bootstrap_service()`) passes the database instance to repositories. Switching the database there switches all downstream repos cleanly — no repository code changes needed for read/write operations that already go through the `transaction()` context manager.

SQLAlchemy Core (not ORM) is the right Postgres client: it supports async, matches the existing "raw SQL" style in repositories, and avoids ORM overhead on a data model this specialized.

### 4. Message Queue → dispatch module

The current `RunWorker` poll loop uses SQLite `LeaseManager` for distributed coordination. For multi-worker horizontal scaling with a real MQ:

```
[API] POST /runs
    → RunRepository.create(status=PENDING)
    → [NEW] MQPublisher.publish("run.pending", run_id)  [optional: fire-and-forget]

[Worker] MQConsumer.subscribe("run.pending")
    → claim run via LeaseManager (still needed for crash recovery)
    → RuntimeOrchestrator._drive()
```

The MQ is an acceleration layer, not a replacement for the lease system. Leases remain authoritative for crash recovery and deduplication. Redis Streams is the lowest-friction choice given Redis is already a dependency. SQS/RabbitMQ are valid alternatives with the same integration shape.

The `RunWorker` interface stays unchanged. A `MQRunWorker` subclass overrides `poll_loop()` to consume from the queue instead of polling SQLite.

### 5. Redis/Vector Memory Connectors → memory module

The `MemoryConnector` protocol is already defined. Adding durable connectors is straightforward:

```python
class RedisKeyValueConnector(MemoryConnector):
    def __init__(self, redis_client, key_prefix: str): ...

class VectorMemoryConnector(MemoryConnector):
    def __init__(self, vector_client, collection: str): ...
```

Wire in `bootstrap_service()` based on `ZerothConfig.memory.backend`. The `InMemoryConnectorRegistry` stays for development/test.

### 6. Docker Sandbox → execution_units module

Phase 8A left hardened Docker as non-default. The change is contained in `sandbox.py`: make `SandboxStrictnessMode.STRICT` the default for `ExecutableUnitNode` with no explicit strictness config. This is a single-line policy change guarded behind `ZerothConfig.sandbox.require_docker = true`.

### 7. Webhooks → webhooks module + service layer

```
RuntimeOrchestrator._drive() completes run
    → run.status = COMPLETED
    → [NEW] webhook_notifier.notify(RunCompletedEvent, tenant_id)
        → WebhookRepository.get_subscriptions(tenant_id, event_type)
        → async HTTP POST to each registered endpoint

ApprovalService.create_pending()
    → [NEW] webhook_notifier.notify(ApprovalPendingEvent, tenant_id)
```

The notifier is injected into the orchestrator and approval service via bootstrap. Webhook subscriptions are stored in a new `webhook_subscriptions` table. Delivery uses exponential backoff with a dead-letter store (max 5 retries).

### 8. Health Probes → health module + service layer

```
GET /health/ready
    → health.check_all()
        → db.ping() (Postgres or SQLite)
        → redis.ping()
        → mq.ping() (if enabled)
        → regulus_backend.ping() (optional, non-blocking)
    → 200 OK or 503 Service Unavailable

GET /health/live
    → shallow check (process alive, worker running)
    → always 200 unless worker stopped
```

### 9. Configuration → config module

Replace the current scattered `from_env()` methods with a single `ZerothConfig` using `pydantic-settings`:

```python
class ZerothConfig(BaseSettings):
    storage: StorageConfig       # backend, pg_dsn, sqlite_path
    redis: RedisConfig           # existing config promoted here
    auth: AuthConfig             # api_keys, bearer — replaces JSON blobs
    guardrails: GuardrailConfig  # existing config promoted here
    regulus: RegulusConfig       # base_url, enabled, budget_enforcement
    mq: MessageQueueConfig       # backend, connection_string
    sandbox: SandboxConfig       # require_docker, strictness_default
    webhooks: WebhookConfig      # delivery_timeout, max_retries
```

Bootstrap reads `ZerothConfig.from_env()` once and threads values through to all modules. This eliminates the "multiple scattered env var readers" problem and enables config validation at startup.

---

## Data Flow Changes

### Run Execution with Telemetry (v1.1)

```
POST /runs
    ↓ auth + RBAC + guardrails (rate limit, quota, [NEW] budget cap)
    ↓ RunRepository.create(PENDING)
    ↓ [NEW, optional] MQPublisher.publish("run.pending")
    ↓
RunWorker.poll_loop() (or MQ consumer)
    ↓ LeaseManager.claim_pending()
    ↓ RuntimeOrchestrator._drive()
        per AgentNode:
            ↓ PolicyGuard.evaluate()
            ↓ AgentRunner.run()
                ↓ PromptAssembler.assemble()
                ↓ [NEW] InstrumentedProviderAdapter.ainvoke()
                    ↓ OpenAIProviderAdapter / AnthropicProviderAdapter
                    ↓ → ProviderResponse (with TokenUsage)
                    ↓ [NEW] econ.record_node_execution() → Regulus backend (async)
            ↓ AuditRepository.append(NodeAuditRecord + token_usage)
            ↓ RunRepository.put(checkpoint)
    ↓ run.status = COMPLETED
    ↓ [NEW] webhook_notifier.notify(RunCompletedEvent)
    ↓ LeaseManager.release_lease()
```

### Approval Flow with Webhooks (v1.1)

```
Orchestrator hits HumanApprovalNode
    ↓ ApprovalService.create_pending()
    ↓ [NEW] webhook_notifier.notify(ApprovalPendingEvent)
    ↓ run.status = WAITING_APPROVAL
    ↓ [NEW, if SLA configured] SLATimeoutPolicy schedules escalation

External reviewer: POST /approvals/{id}/resolve
    ↓ ApprovalService.resolve()
    ↓ run re-queued via schedule_continuation
    ↓ [NEW] webhook_notifier.notify(ApprovalResolvedEvent)
```

---

## Recommended Project Structure (additions only)

```
src/zeroth/
├── config.py                    # [NEW] ZerothConfig (pydantic-settings)
├── llm_providers/               # [NEW] Real LLM provider adapters
│   ├── __init__.py
│   ├── openai.py                # OpenAIProviderAdapter (async)
│   ├── anthropic.py             # AnthropicProviderAdapter (async)
│   ├── retry.py                 # Exponential backoff + model fallback
│   └── errors.py
├── econ/                        # [NEW] Regulus SDK integration
│   ├── __init__.py
│   ├── client.py                # EconClient wrapping econ_instrumentation
│   ├── models.py                # TokenUsage, CostEvent, BudgetResult
│   ├── adapter.py               # InstrumentedProviderAdapter
│   └── budget.py                # BudgetEnforcer (pre-execution cap check)
├── webhooks/                    # [NEW] Outgoing webhook notifications
│   ├── __init__.py
│   ├── models.py                # WebhookSubscription, WebhookDelivery, event types
│   ├── repository.py            # SQLite/PG-backed subscription store
│   ├── notifier.py              # WebhookNotifier (async, with retry)
│   └── errors.py
├── health/                      # [NEW] Readiness / liveness probes
│   ├── __init__.py
│   ├── checks.py                # HealthCheck protocol + implementations
│   └── service.py               # HealthService orchestrates checks
├── storage/
│   ├── sqlite.py                # [EXISTING, unchanged]
│   ├── postgres.py              # [NEW] PostgresDatabase (SQLAlchemy Core async)
│   ├── redis.py                 # [EXISTING, minor: promote to ZerothConfig]
│   └── migrations/              # [NEW] Shared migration registry
├── agent_runtime/
│   ├── provider.py              # [MODIFY] ProviderResponse + TokenUsage field
│   └── runner.py                # [MODIFY] Record usage in audit, inject econ client
├── dispatch/
│   ├── worker.py                # [MODIFY] MQRunWorker subclass option
│   └── lease.py                 # [MODIFY] PostgresLeaseManager option
├── memory/
│   ├── connectors.py            # [MODIFY] Add RedisMemoryConnector, VectorMemoryConnector
├── service/
│   ├── app.py                   # [MODIFY] Add /v1 prefix, register health + webhook routes
│   ├── bootstrap.py             # [MODIFY] Add webhook_notifier, econ_client, config loader
│   └── health_api.py            # [NEW] /health/ready + /health/live
│   └── webhook_api.py           # [NEW] CRUD for webhook subscriptions
```

---

## Architectural Patterns

### Pattern 1: Protocol-Boundary Extension

**What:** All new capabilities plug into existing protocol boundaries — `ProviderAdapter`, `MemoryConnector`, `SecretProvider` — without modifying consumers.
**When to use:** LLM providers, memory connectors, secret backends — anything with an existing protocol.
**Trade-offs:** Existing tests remain valid. New implementations are independently testable. No orchestrator or runner changes needed.

### Pattern 2: Decorator / Wrapper Adapter for Cross-Cutting Concerns

**What:** `InstrumentedProviderAdapter` wraps any `ProviderAdapter` to add telemetry without the LLM adapter knowing about Regulus, and without the runner knowing about telemetry.
**When to use:** Regulus integration, retry policy, circuit breaker — any concern that applies orthogonally to all providers.
**Trade-offs:** Keeps provider adapters and instrumentation fully decoupled. Can stack multiple decorators (retry → instrument → actual provider).

```python
# In bootstrap_service():
raw_adapter = OpenAIProviderAdapter(client=openai_client)
retry_adapter = RetryProviderAdapter(raw_adapter, policy=node.retry_policy)
final_adapter = InstrumentedProviderAdapter(retry_adapter, econ_client=econ_client)
```

### Pattern 3: Configuration-Driven Backend Selection

**What:** `ZerothConfig` selects storage backend, MQ backend, memory connector backend. Bootstrap reads config once and wires the right implementation. No conditional logic in domain modules.
**When to use:** Postgres vs SQLite, Redis MQ vs SQS, Redis memory vs in-memory.
**Trade-offs:** Clean separation between "what to build" and "which backend to use". Enables SQLite for test, Postgres for production without code changes.

### Pattern 4: Fire-and-Forget Telemetry

**What:** Regulus `TelemetryTransport` runs a background daemon thread with an in-process queue. `track_execution()` enqueues and returns immediately. The background thread batches and flushes to the Regulus backend.
**When to use:** The Regulus integration is inherently this pattern — don't fight it.
**Trade-offs:** Non-blocking, no LLM latency impact. If Regulus backend is down, events are dropped after the buffer fills (not a correctness concern for Zeroth). Telemetry is best-effort, not transactional.

---

## Anti-Patterns

### Anti-Pattern 1: Embedding Regulus Backend Inside Zeroth

**What people do:** Deploy the `econ_plane` FastAPI server as part of the Zeroth process or as an embedded ASGI sub-app.
**Why it's wrong:** Regulus is a separate service with its own database (Postgres), its own worker (connector workers), and its own scaling requirements. Embedding it creates a single-failure-domain dependency on a service that should be independently operable.
**Do this instead:** Regulus runs as a companion Docker container. Zeroth uses the SDK (`econ_instrumentation`) to emit telemetry. SDK transport handles connection failures gracefully (drops events rather than blocking Zeroth).

### Anti-Pattern 2: Adding Provider Logic to the Orchestrator

**What people do:** Put OpenAI/Anthropic retry logic, rate limit handling, or token counting directly in `RuntimeOrchestrator._drive()`.
**Why it's wrong:** The orchestrator is already the most complex module (774 lines, fragile state machine). Adding provider concerns mixes execution scheduling with IO concerns and makes both harder to test.
**Do this instead:** `RetryProviderAdapter` and `InstrumentedProviderAdapter` wrap the raw provider adapter in the bootstrap. The orchestrator only knows about `ProviderAdapter.ainvoke()`.

### Anti-Pattern 3: Replacing SQLite Repositories Wholesale

**What people do:** Rewrite all 20 repositories to target Postgres, changing repository method signatures and migration schemas everywhere.
**Why it's wrong:** Breaks 280 tests, requires coordinated changes across every domain module, and is unnecessary — most repositories are read-heavy and SQLite performs fine for graph/contract/deployment data.
**Do this instead:** Migrate only high-contention tables (runs, leases, rate limits) to Postgres. Keep the `PostgresDatabase` and `SQLiteDatabase` behind the same interface so the repository layer is unchanged.

### Anti-Pattern 4: Synchronous Regulus Budget Checks on Every Node

**What people do:** Call the Regulus backend on every node execution to check the current spend against budget, blocking the orchestrator until the HTTP call returns.
**Why it's wrong:** Adds 50-200ms of HTTP latency per node. The orchestrator runs synchronously through the `_drive()` loop — a slow external call stalls the entire run.
**Do this instead:** Cache budget state locally (TTL 60 seconds) in `BudgetEnforcer`. Refresh asynchronously in the background. Use the cached value for per-node checks. This turns a blocking HTTP call into a memory lookup.

---

## Suggested Build Order

Dependencies drive this order. Each phase delivers a working, tested increment.

### Phase 1: Foundation (config + storage backend)
**Rationale:** Everything downstream depends on unified config and the ability to use Postgres.

1. `src/zeroth/config.py` — `ZerothConfig` with pydantic-settings; replaces scattered `from_env()` calls
2. `src/zeroth/storage/postgres.py` — `PostgresDatabase` with same interface as `SQLiteDatabase`
3. Migrate `RunRepository` and `LeaseManager` to support both backends (backend injected, not hardcoded)
4. Migrate `guardrails/rate_limit.py` tables to Postgres backend
5. Update `bootstrap_service()` to read from `ZerothConfig`

**Outputs:** Postgres-capable storage layer; unified config; 280 tests still green on SQLite.

### Phase 2: Real LLM Providers + Retry
**Rationale:** Core platform value — nothing works without real LLMs. Retry is essential for production reliability.

1. `src/zeroth/llm_providers/openai.py` — `OpenAIProviderAdapter`
2. `src/zeroth/llm_providers/anthropic.py` — `AnthropicProviderAdapter`
3. `src/zeroth/llm_providers/retry.py` — `RetryProviderAdapter` (exponential backoff, model fallback)
4. Add `TokenUsage` to `ProviderResponse` in `agent_runtime/provider.py`
5. Update `AgentRunner` to propagate usage to audit records
6. Wire via bootstrap; no orchestrator changes

**Outputs:** Real LLM execution; retry with backoff; token usage in audit records.

### Phase 3: Regulus Integration
**Rationale:** Depends on Phase 2 (need `TokenUsage` from real providers).

1. `src/zeroth/econ/models.py` — `TokenUsage`, `CostEvent`
2. `src/zeroth/econ/client.py` — `EconClient` wrapping `econ_instrumentation`
3. `src/zeroth/econ/adapter.py` — `InstrumentedProviderAdapter`
4. `src/zeroth/econ/budget.py` — `BudgetEnforcer` with local cache
5. Wire `InstrumentedProviderAdapter` in bootstrap (stacks on top of retry adapter)
6. Add budget cap pre-execution check to `GuardrailChecker` in `guardrails/`

**Outputs:** Token/cost telemetry flowing to Regulus; budget enforcement per tenant.

### Phase 4: Memory Connectors + Container Sandbox
**Rationale:** Independent from phases 2-3; unblocks persistent agent memory and safe code execution.

1. `src/zeroth/memory/redis_connector.py` — `RedisKeyValueConnector`, `RedisThreadConnector`
2. `src/zeroth/memory/vector_connector.py` — `VectorMemoryConnector` (pluggable vector client)
3. Fix `AgentRunner` default thread store (use `RepositoryThreadStateStore` by default)
4. Complete Phase 8A: make Docker sandbox default in `execution_units/sandbox.py` when `ZerothConfig.sandbox.require_docker = true`

**Outputs:** Durable agent memory; container-isolated code execution.

### Phase 5: Webhooks + Approval SLA
**Rationale:** Depends on stable run lifecycle (phases 1-2). Independent from Regulus.

1. `src/zeroth/webhooks/models.py`, `repository.py`, `notifier.py`
2. Wire `WebhookNotifier` into `RuntimeOrchestrator` (on run completion) and `ApprovalService` (on pending/resolved)
3. Add SLA timeout policy to `ApprovalService` (schedule escalation timer)
4. Register `service/webhook_api.py` routes for subscription management

**Outputs:** Outgoing webhooks for run and approval events; approval escalation.

### Phase 6: MQ + Horizontal Scaling
**Rationale:** Depends on Postgres lease backend (Phase 1) and stable dispatch semantics.

1. `MQPublisher` and `MQConsumer` in `dispatch/` (Redis Streams as default backend)
2. `MQRunWorker` subclass that consumes from queue instead of polling
3. Wire via `ZerothConfig.mq.backend`
4. Test horizontal worker scaling (multiple workers competing for leases)

**Outputs:** Queue-backed dispatch; horizontal worker scaling support.

### Phase 7: Health Probes + Deployment Packaging
**Rationale:** Can start any time; must be last before production deployment.

1. `src/zeroth/health/` — readiness/liveness checks
2. Register `/health/ready` and `/health/live` routes
3. `Dockerfile` — multi-stage build
4. `docker-compose.yml` — zeroth + postgres + redis + regulus-backend
5. API versioning prefix (`/v1/`) in `app.py`
6. OpenAPI spec generation verification

**Outputs:** Container image; deployable stack; health probes; versioned API.

---

## Integration Points Summary

| Boundary | Direction | Pattern | Notes |
|----------|-----------|---------|-------|
| `llm_providers` → `agent_runtime` | Plugin | `ProviderAdapter` protocol | No orchestrator changes |
| `econ` → `agent_runtime` | Wrapper | `InstrumentedProviderAdapter` decorator | Fire-and-forget telemetry |
| `econ` → Regulus backend | Outbound HTTP | `TelemetryTransport` (background thread) | Best-effort, non-blocking |
| `econ` → `guardrails` | In-process call | `BudgetEnforcer` with TTL cache | Pre-execution cap check |
| `storage/postgres` → all repositories | Backend swap | Same `transaction()` interface | Injected via bootstrap |
| `dispatch` → MQ backend | Consumer/publisher | `MQRunWorker` subclass | Redis Streams default |
| `memory` → Redis/vector | Plugin | `MemoryConnector` protocol | Protocol already defined |
| `webhooks` → external HTTP | Outbound | Async POST with retry | Triggered by orchestrator + approvals |
| `health` → all backends | Probe | `HealthCheck` protocol | DB + Redis + MQ + Regulus |
| `service` → `webhooks` + `health` | Route registration | FastAPI router | Same pattern as existing APIs |

### Internal Dependency Additions

```
econ/ → agent_runtime/ (reads TokenUsage from ProviderResponse)
econ/ → guardrails/ (BudgetEnforcer plugs into guardrail checks)
webhooks/ → runs/ (subscribe to run state transitions)
webhooks/ → approvals/ (subscribe to approval events)
health/ → storage/ + redis + dispatch/ (probe dependencies)
config.py → everything (replaces scattered from_env() calls)
```

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Single process (dev) | SQLite + in-memory MQ (poll loop) + InMemory connectors |
| 2-8 worker processes | Postgres + SQLite lease backend + Redis Streams MQ |
| 10+ workers / multi-host | Postgres for all state + Redis Streams + external vector store |
| High tenant count | Per-tenant Postgres schema or row-level isolation; Regulus budget checks critical |

**First bottleneck:** SQLite write contention under multiple concurrent runs. Postgres migration (Phase 1) resolves this.

**Second bottleneck:** Single-process worker poll interval. MQ consumer (Phase 6) eliminates polling latency and enables true horizontal scaling.

**Third bottleneck:** Synchronous per-node budget checks. Resolved by `BudgetEnforcer` TTL cache (Phase 3).

---

## Sources

- Direct source inspection: `src/zeroth/agent_runtime/provider.py`, `src/zeroth/service/bootstrap.py`, `src/zeroth/orchestrator/runtime.py`, `src/zeroth/dispatch/worker.py`, `src/zeroth/storage/sqlite.py`, `src/zeroth/guardrails/`
- Direct source inspection: `/Users/dondoe/coding/regulus/sdk/python/econ_instrumentation/` — `__init__.py`, `schemas.py`, `client.py`, `transport.py`, `integrations/openai.py`, `integrations/anthropic.py`
- Codebase planning documents: `.planning/codebase/ARCHITECTURE.md`, `STACK.md`, `INTEGRATIONS.md`, `CONCERNS.md`, `STRUCTURE.md`
- Project context: `.planning/PROJECT.md` (v1.1 milestone requirements)

---

*Architecture research for: Zeroth v1.1 production readiness — integration of new capabilities into existing modular monolith*
*Researched: 2026-04-06*
