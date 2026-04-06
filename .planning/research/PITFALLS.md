# Pitfalls Research

**Domain:** Production infrastructure hardening of an existing governed multi-agent platform
**Researched:** 2026-04-06
**Confidence:** HIGH (codebase directly inspected; findings validated against current official docs and community sources)

---

## Critical Pitfalls

### Pitfall 1: Breaking the 280-Test Suite by Swapping the Synchronous Repository Layer for Async Postgres

**What goes wrong:**
Every repository in Zeroth (runs, leases, approvals, audit, contracts, graphs, deployments, guardrails) uses synchronous SQLite calls via a hand-rolled `SQLiteDatabase` wrapper. The async boundary lives at the orchestrator and HTTP layers. When Postgres is introduced teams often reach for `asyncpg` + async SQLAlchemy because FastAPI is already async — converting the entire repository layer to async to match. This instantly invalidates hundreds of tests that call repositories synchronously, triggers a cascading rewrite of the orchestrator's `_drive()` loop (already 774 lines, explicitly flagged fragile in CONCERNS.md), and produces subtle bugs where an `await` is missing inside the dispatch worker's tight lease-renewal concurrency.

**Why it happens:**
"Async all the way down" is the FastAPI community pattern. Developers see async SQLAlchemy and asyncpg as the modern choice and convert the whole stack in one step, not recognising that the existing sync/async split is intentional and load-bearing.

**How to avoid:**
Keep the repository layer **synchronous**. Use `psycopg2` (the sync driver) or `psycopg3` in sync mode behind a thin `PostgresDatabase` class that implements the same interface as `SQLiteDatabase`. The repositories themselves do not change. SQLite remains the test and development backend; Postgres is selected by an environment flag at bootstrap. This preserves all 280 tests with zero modifications because the test harness continues to use the SQLite backend. The async/sync boundary stays exactly where it is today.

**Warning signs:**
- Any PR that adds `async def` to a repository method
- Import of `asyncpg` or `sqlalchemy.ext.asyncio` into `src/zeroth/storage/`
- Test failures in `tests/runs/`, `tests/dispatch/`, or `tests/service/` during the Postgres phase

**Phase to address:** Postgres migration phase. Introduce `PostgresDatabase` behind the existing storage interface; never touch the async boundary.

---

### Pitfall 2: Telemetry SDK Auto-Instrumentation Corrupting Audit Records

**What goes wrong:**
Regulus's `econ_instrumentation` SDK uses monkey-patching to intercept OpenAI and Anthropic SDK calls at the module level. Zeroth's audit subsystem independently records `NodeAuditRecord` entries with a digest chain for tamper detection. If the telemetry SDK wraps the LLM client at the wrong layer, it can observe calls that Zeroth has already counted or inject additional network calls that produce usage entries without corresponding audit records, breaking the continuity verifier in `src/zeroth/audit/verifier.py`. Worse, any SDK that patches `asyncio` event loop internals can silently swallow context-var state, losing correlation IDs or tenant context mid-flight.

**Why it happens:**
Auto-instrumentation SDKs are designed to "just work" by patching at import time. The implicit assumption is that the host application has no concurrent audit mechanism. In Zeroth, audit is not just observability — it is a governance artefact with integrity guarantees.

**How to avoid:**
Do not use Regulus auto-instrumentation in the orchestrator hot path. Instead, instrument **explicitly** at the `AgentRunner.run()` return point: read `ProviderResponse.usage` (token counts), emit a Regulus span around that single call, and pass the span ID into the `NodeAuditRecord`. This keeps Regulus as a companion observer, not an interceptor. Validate in tests that `AuditContinuityReport` still passes with Regulus enabled.

**Warning signs:**
- Regulus import appearing before `GovernedLLMProviderAdapter` import in any module
- `asyncio_mode = "auto"` tests failing intermittently (context-var loss)
- `AuditContinuityReport.gaps` becoming non-empty after enabling telemetry

**Phase to address:** Regulus integration phase. Explicit instrumentation call at `AgentRunner` level, not at SDK import level.

---

### Pitfall 3: LLM Retry Logic Double-Counting Costs and Corrupting Audit Chains

**What goes wrong:**
Provider-aware retry with exponential backoff is essential for production. But if a retry is treated as a continuation of the same audit record, token usage is under-reported (only the final attempt's tokens recorded) or over-reported (all attempts summed into one record with no attribution). Separately, a silent retry that succeeds at attempt 3 looks identical to a first-try success in the governance audit trail, making it impossible to detect systematic provider degradation or unexplained cost spikes.

**Why it happens:**
Retry logic is commonly bolted onto the provider adapter level without considering that the audit system expects one `NodeAuditRecord` per node execution. Developers add a simple `for attempt in range(max_retries): try/except` loop inside `GovernedLLMProviderAdapter.call()` without realising that token tracking and audit recording happen above that layer in `AgentRunner`.

**How to avoid:**
Each retry attempt must emit its own sub-span or usage record with attempt number, model, and token counts. The final `NodeAuditRecord` should include a `retry_count` field and aggregate usage across all attempts. Idempotency is not a concern for LLM calls (they are read-only from a state perspective), but the cost ledger must reflect all attempts, not just the successful one.

**Warning signs:**
- Regulus cost reports showing lower-than-expected token usage during known degraded periods
- Audit records with no `retry_count` field despite 429 errors visible in provider logs
- Test assertions checking `AgentRunResult.usage.total_tokens` that pass on first call but silently drop tokens on retry

**Phase to address:** LLM provider adapter phase. Define retry telemetry contract before writing the retry loop.

---

### Pitfall 4: Test Suite Contamination from Real LLM Providers

**What goes wrong:**
When real OpenAI/Anthropic adapters are introduced, developers add integration tests that make live API calls. These tests are slow (3-30 seconds each), flaky under rate limits, and cost money on every CI run. More critically, if `pytest-asyncio` fixtures are not carefully scoped, a test that imports the real provider adapter can accidentally activate it in unrelated unit tests that expect `DeterministicProviderAdapter`, causing cascading failures across the 280-test suite.

**Why it happens:**
The `DeterministicProviderAdapter` is currently wired via direct instantiation in test fixtures. The import path for the real adapter is in the same `src/zeroth/agent_runtime/provider.py` module. A misconfigured conftest or an accidental default can substitute the real provider.

**How to avoid:**
Gate all real-provider tests behind a `@pytest.mark.live` marker and an environment variable (`ZEROTH_LIVE_TESTS=1`). Ensure the real provider adapters are never imported at module level in conftest files. The `DeterministicProviderAdapter` remains the default for all non-live tests. Add a CI check that `uv run pytest -v` (without `ZEROTH_LIVE_TESTS`) runs in under 60 seconds and makes zero network calls.

**Warning signs:**
- Test runtime increasing from seconds to minutes
- Intermittent `openai.RateLimitError` or `anthropic.APIStatusError` in CI
- Any test file importing `openai` or `anthropic` without a `pytest.mark.live` guard

**Phase to address:** LLM provider adapter phase. Establish the marker convention before writing the first live adapter test.

---

### Pitfall 5: Container Sandbox Using Docker-in-Docker Shares the Host Kernel

**What goes wrong:**
Phase 8A's `DockerSandboxConfig` launches containers using the local Docker daemon. In a containerised deployment (the v1.1 milestone adds a Dockerfile), this means executing untrusted code in a container that shares its Docker socket with the Zeroth service container — a textbook container escape vector. An untrusted execution unit that mounts the Docker socket can list, stop, or exec into the Zeroth service container or any sibling container, defeating the entire governance model.

**Why it happens:**
The sandbox was designed for development (running Docker on the host machine). When the service is Dockerised, developers mount `/var/run/docker.sock` because "that's how Docker-in-Docker works" — not realising that socket mounting grants container root over the entire host Docker environment.

**How to avoid:**
For the production Dockerfile, the sandbox backend must use either (a) a separate privileged sidecar container that owns the Docker socket, with the Zeroth service communicating over a restricted API, or (b) a rootless Podman or `gVisor`-backed runtime. The simpler short-term option is to deploy the sandbox worker as a separate service that receives execution requests over a local socket, keeping the Zeroth API container completely free of Docker socket access. Document this as a hard deployment requirement.

**Warning signs:**
- `docker-compose.yml` that mounts `- /var/run/docker.sock:/var/run/docker.sock` on the zeroth service
- `SandboxManager` instantiated directly inside the FastAPI lifespan (same process as the API)
- Any test or documentation that says "just add the socket mount"

**Phase to address:** Container sandbox hardening phase AND Dockerfile phase. The two must be designed together, not sequentially.

---

### Pitfall 6: Message Queue Introduction Duplicating the Lease Manager's Guarantees

**What goes wrong:**
Zeroth has a carefully implemented lease-based dispatch: `LeaseManager` does atomic claim via `UPDATE ... WHERE status = 'PENDING' AND lease_id IS NULL`, with orphan recovery on startup. Adding a message queue (Redis Streams, SQS, RabbitMQ) as a "production upgrade" to the dispatch system creates two competing durability mechanisms. The common mistake is to publish a message to the queue when a run is created AND write `status=PENDING` to the database, then have workers consume from the queue. This produces dual-delivery: a worker that crashes between queue `ACK` and database state update leaves the run in `RUNNING` with no live worker, while the message has been consumed and is gone. Orphan recovery no longer works because the run never re-appears as `PENDING`.

**Why it happens:**
"Add a proper queue for production" sounds straightforward. Developers model it after standard task queue patterns (Celery, RQ) without accounting for the fact that Zeroth's state machine is in the database, not the queue. The queue becomes an unreliable shadow of database state.

**How to avoid:**
The database is the authoritative queue. The message queue is a **notification** mechanism only: when a run is written as `PENDING`, publish a lightweight "run_id ready" notification to Redis Pub/Sub or a lightweight channel. Workers that are listening wake up immediately and attempt a lease claim via the existing `LeaseManager.claim_pending()` — which already handles contention atomically. Workers that missed the notification (or are starting fresh) continue to poll on a slow interval. This gives near-instant dispatch latency without replacing the durable lease mechanism.

**Warning signs:**
- Any design where `status=PENDING` and queue publication are not in the same database transaction
- Workers that `ACK` a message before updating the run status to `RUNNING`
- Removing the `LeaseManager` polling loop because "the queue handles that now"

**Phase to address:** Durable dispatch enhancement phase. The queue supplements polling; it does not replace the lease.

---

### Pitfall 7: API Versioning Retrofit Breaking All Existing URL Paths

**What goes wrong:**
FastAPI versioning libraries (e.g., `fastapi-versioning`, `versionize`) prefix all registered routes with `/v1/` (or the configured version) when versioning is enabled globally. Zeroth's current clients call `/runs`, `/approvals`, `/audit`, etc. Enabling versioning retroactively turns every existing path into `/v1/runs`, `/v1/approvals`, etc. — breaking every deployed client integration and any approval UI that has the paths hardcoded.

**Why it happens:**
The library's README shows enabling versioning in one line on the FastAPI app and shows all routes versioned. Teams assume "adding v1 doesn't change v1 paths" because they are the same endpoints. But unversioned paths disappear by default.

**How to avoid:**
Introduce versioning by registering a new `APIRouter` under `/v1/` prefix and mounting it alongside the existing unversioned router. Keep the unversioned paths alive as aliases (or redirect them with a deprecation warning). Only new endpoints introduced in v1.1 need to live under `/v1/` from day one. Document the deprecation timeline for unversioned paths. Never remove unversioned paths in the same PR that introduces versioned ones.

**Warning signs:**
- Any test in `tests/service/` failing with 404 after versioning is enabled
- A `fastapi_versioning.VersionedFastAPI` wrapper applied at the app factory level
- Approval UI calling `/approvals/` that starts returning 404

**Phase to address:** API versioning phase. Dual-router approach required; do not use a global versioning wrapper.

---

### Pitfall 8: Webhook Delivery Conflating "Fire and Forget" with "At-Least-Once"

**What goes wrong:**
Webhook notifications for run completion and approval events are added as a `asyncio.create_task(httpx.post(...))` call inside the orchestrator's completion path. This approach has no durability: if the target endpoint is slow, the ASGI worker is restarted before the `httpx` call completes, or the call fails with a 5xx, the notification is silently dropped. Clients building on top of Zeroth cannot rely on webhooks for anything consequential (billing events, downstream workflow triggers).

**Why it happens:**
`asyncio.create_task` is the idiomatic way to do non-blocking work in FastAPI/asyncio applications. It works perfectly in development. The failure modes (process restart, slow receiver) only appear in production.

**How to avoid:**
Store webhook delivery attempts in the database as a `webhook_deliveries` table with `status` (PENDING, DELIVERED, FAILED), `next_retry_at`, and `attempt_count` columns. A background task (similar to the `RunWorker` polling loop) periodically reads PENDING deliveries and attempts HTTP delivery with exponential backoff. On success, mark DELIVERED. After N failures, mark FAILED and emit an audit event. This pattern guarantees at-least-once delivery with a recoverable dead-letter state.

**Warning signs:**
- `httpx.post` calls inside `RuntimeOrchestrator` or `RunWorker` without a surrounding persistence step
- Webhook tests that only verify the HTTP call was made, not that delivery state was persisted
- No dead-letter or retry table in the database schema

**Phase to address:** Webhook notification phase. Design the delivery table before writing the HTTP dispatch code.

---

### Pitfall 9: GovernAI Local Path Dependency Blocking Docker Build

**What goes wrong:**
`pyproject.toml` pins `governai @ file:///Users/dondoe/coding/governai`. A `docker build` copies the Zeroth source into the image and runs `uv sync` — which fails immediately because `/Users/dondoe/coding/governai` does not exist inside the container. This is already flagged in CONCERNS.md but becomes a hard blocker the moment any Dockerfile is written.

**Why it happens:**
The local path dependency works for local development. The blocker is invisible until a Docker build is attempted for the first time.

**How to avoid:**
Before the Dockerfile phase, change the GovernAI dependency to a git+https reference with the pinned commit: `governai @ git+https://github.com/rrrozhd/governai.git@7452de4`. This is a one-line change to `pyproject.toml` that unblocks Docker, CI, and any developer who does not replicate the exact directory layout. Verify with `uv sync` on a clean checkout before proceeding.

**Warning signs:**
- `docker build` failing with `FileNotFoundError` or `uv sync` error about missing local path
- CI pipeline failing at the dependency installation step
- Any new developer unable to `uv sync` without cloning governai to the exact path

**Phase to address:** Must be resolved as a prerequisite to the Dockerfile/containerisation phase, not as part of it.

---

### Pitfall 10: In-Memory Metrics Collector Masking Production Performance Problems

**What goes wrong:**
`MetricsCollector` stores all counters and histograms in thread-local Python dicts. In a multi-worker deployment (multiple `uvicorn` processes or Docker replicas), each process has its own isolated metrics state. The `/admin/metrics` endpoint reports only the metrics for the process serving that request. This means 7 of 8 workers could be at 100% concurrency while the one worker serving the metrics request reports 0 active runs. Capacity decisions made on this data are wrong.

**Why it happens:**
In-process metrics work correctly in single-process development. The problem is invisible until horizontal scaling is introduced.

**How to avoid:**
Before adding horizontal scaling support, route metrics to an external sink (Prometheus via push gateway, StatsD, or Redis-backed counters). The existing Prometheus text renderer in `MetricsCollector` is a stub — wire it to a real scrape endpoint or a push job. At minimum, store aggregate counters in Redis so all workers share the same metric state. Do not add a "scale to N workers" feature while metrics remain in-memory.

**Warning signs:**
- `/admin/metrics` showing inconsistent data between successive requests (different workers responding)
- Load tests showing high run throughput but metrics reporting low utilization
- Any horizontal scaling PR that does not address the metrics backend

**Phase to address:** Horizontal worker scaling phase. Fix metrics aggregation before advertising multi-worker support.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Keeping `run.metadata` as `dict[str, Any]` through all v1.1 phases | No refactoring effort | New production features (webhook state, retry counts, budget enforcement) add more untyped keys; key collisions become a production incident risk | Only if no new metadata keys are added — but v1.1 requires at least 5 new keys |
| Using SQLite for the lease manager even with Postgres elsewhere | Avoid refactoring lease logic | SQLite single-writer contention becomes the throughput ceiling exactly when Postgres is supposed to remove it | Never — the lease and run tables must migrate together |
| Auto-instrumentation for Regulus rather than explicit spans | Zero integration code | Breaks audit chain integrity, corrupts correlation IDs in async context | Never — Zeroth's audit is a governance artefact, not just observability |
| Docker socket mount for sandbox execution | Sandbox "just works" in Docker | Container escape vulnerability; untrusted code can control the host Docker daemon | Never in production; acceptable in single-tenant local dev only |
| `asyncio.create_task` for webhook delivery | Simple, no schema changes | Silent delivery failures; no dead-letter; no at-least-once guarantee | Never for consequential events; acceptable for fire-and-forget informational pings |
| `file://` local path for GovernAI in Dockerfile | Works if mounted correctly | Breaks `docker build`, CI, every other developer's machine | Never — change to git ref before writing any Dockerfile |
| Keeping `/runs` unversioned and adding `/v1/runs` as a duplicate | Backward compatible | Two code paths to maintain; easy to diverge; confuses OpenAPI spec | Acceptable as a transition state for one release cycle only |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| OpenAI adapter | Using `response.choices[0].message.content` only, discarding `usage.prompt_tokens` / `usage.completion_tokens` | Always read and surface `usage` object; it is the input to Regulus cost attribution and the audit token count |
| Anthropic adapter | Passing system prompt as the first human message (OpenAI compat layer behaviour) | Use native Anthropic SDK; the `system` parameter is top-level, not a message — the compat layer silently mangles this |
| asyncpg / psycopg3 behind pgbouncer | Prepared statement cache errors in transaction pooling mode | Disable prepared statements: `statement_cache_size=0` (asyncpg) or `prepare_threshold=None` (psycopg3) when pooling middleware is in use |
| Regulus SDK auto-instrumentation | Calling `econ_instrumentation.auto_instrument()` at module import | Call it only after the GovernAI/LLM client is instantiated and only outside the test environment; gate on `REGULUS_ENABLED` env var |
| Redis Pub/Sub for dispatch wakeup | Subscribing before the worker's lease claim loop is ready | Start the subscription listener after `LeaseManager` is initialised; messages received before the worker is ready must be discardable (the poller catches up) |
| Redis vector store | Using FLAT index for large embedding sets | Use HNSW for approximate nearest neighbour; FLAT is exhaustive (O(n)) and blocks the Redis thread for large corpora |
| Docker sandbox in composed deployment | Mounting `/var/run/docker.sock` on the API container | Separate sandbox execution into its own sidecar container; API container never touches the Docker socket |
| Webhook delivery | Retrying on 4xx responses | Only retry on 5xx and network errors; 400/401/403/404 are permanent failures — move to dead-letter immediately |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| SQLite single-writer under concurrent run checkpoints | Checkpoint writes queue up; lease renewal timeouts spike; runs appear stuck in RUNNING | Migrate `runs`, `run_checkpoints`, `audit_records`, and `lease` tables to Postgres together; keep graph/contract/deployment tables in SQLite | ~10 concurrent runs with checkpoint writes + audit + rate limiter writes |
| Synchronous JWKS fetch in `_load_jwks()` blocking auth path | Occasional 10-30s auth latency; upstream timeout complaints | Replace with `httpx` async fetch with TTL-based caching (JWKS rarely change; 1-hour TTL is fine) | Any time the JWKS endpoint is slow or unreachable |
| O(n) node lookup in orchestrator for large graphs | Step dispatch time grows linearly with graph size | Build `{node_id: node}` and `{(src, tgt): edge}` index dicts once at graph load in `bootstrap_service()` | Graphs with >50 nodes; already flagged in CONCERNS.md |
| In-memory sandbox environment cache rebuilt on every restart | Container cold-start adds N × sandbox-prep time before first run | Persist prepared environment digests to Redis; restore on startup | Any deployment with >5 sandbox execution unit types |
| Regulus companion service timing out during token metering in critical path | Agent steps take unexpectedly long; `AgentRunner` timeout fires before LLM call completes | Make Regulus span emission fire-and-forget (non-blocking); only budget enforcement (hard reject) should be synchronous | As soon as Regulus backend has any latency (network hop, cold start) |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Storing raw LLM API keys in `ZEROTH_SERVICE_API_KEYS_JSON` env var alongside provider API keys | All credentials in one blast radius; env vars visible in process listings and crash dumps | Use separate env vars per credential type; rotate LLM provider keys independently; consider a secrets manager (Vault, AWS SSM) for production |
| Passing tenant API keys to LLM providers without per-tenant key isolation | One compromised provider key exposes all tenant spend | Use separate OpenAI/Anthropic organisations or projects per tenant, or gate all provider calls through Regulus which enforces budget caps before the call reaches the provider |
| Container image built with `governai` git credentials baked in | Git credentials in Docker layer cache; leaks in image registry | Use Docker build secrets (`--secret`) for git auth during `uv sync`; never bake tokens into image layers |
| Running Docker sandbox with `--privileged` flag for capability compatibility | Full host privilege escalation from within a "sandboxed" execution unit | Use `--cap-drop ALL --cap-add` only the specific capabilities required; never `--privileged` in multi-tenant deployments |
| Webhook delivery without HMAC signature | Malicious party spoofs delivery confirmations or replays events | Sign all outbound webhook payloads with HMAC-SHA256 using a per-tenant secret; document verification instructions for receivers |
| SQLite `EncryptedField` with no key rotation | Compromise of the single Fernet key decrypts all historical secret values | Implement key rotation: dual-decrypt during transition, re-encrypt on next write, remove old key after rotation window closes |

---

## "Looks Done But Isn't" Checklist

- [ ] **LLM provider adapters:** Real API calls work in isolation — verify that `NodeAuditRecord.token_usage` is populated for every agent node execution, not just when Regulus is enabled.
- [ ] **Regulus integration:** Cost spans emit correctly — verify that `AuditContinuityReport` passes with Regulus enabled; a broken instrumentation layer that silently swallows spans looks like success.
- [ ] **Postgres migration:** Repositories pass tests — verify that lease contention works correctly under concurrent writes (run a parallel test with 8 workers competing for the same run); SQLite's serialised writes hide race conditions that appear under Postgres concurrent access.
- [ ] **Container sandbox:** Docker container launches in tests — verify that resource constraints (`--memory`, `--cpu-quota`) are actually enforced by running a container that exceeds limits and confirming it is killed, not just warned.
- [ ] **Message queue dispatch:** Runs are delivered faster — verify orphan recovery still works by killing a worker mid-execution and confirming the run resumes on restart; the queue notification path can mask a broken fallback poller.
- [ ] **Dockerfile:** `docker build` succeeds — verify `uv sync` resolves the GovernAI git dependency (not the local path), and that the final image starts the service without a local filesystem dependency.
- [ ] **Webhook notifications:** Delivery fires in tests — verify that a delivery attempted against a slow/unreachable endpoint is retried and eventually reaches dead-letter state; "fires once" is not the same as "at-least-once".
- [ ] **API versioning:** `/v1/runs` returns the right response — verify that the original `/runs` path still returns 200 (not 404) after versioning is introduced; breaking existing paths is invisible until a client reports it.
- [ ] **Health probes:** `/health/ready` returns 200 — verify it returns 503 when Postgres is unreachable, Redis is down, or the GovernAI dependency is unavailable; a probe that always returns 200 provides no protection.
- [ ] **Horizontal scaling:** N workers start — verify that two workers running simultaneously do not double-execute the same run (lease exclusivity test under Postgres).

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Async repository rewrite broken 280 tests | HIGH | Revert repository layer to sync; introduce a `PostgresDatabase` sync wrapper; re-run the full test suite before merging |
| Regulus instrumentation corrupted audit chain | MEDIUM | Disable auto-instrumentation; re-run affected runs with explicit span emission; audit records cannot be patched (append-only), so the gap must be documented in the `AuditContinuityReport` |
| LLM retry double-counted tokens in billing | MEDIUM | Replay audit records for the affected time window; emit correcting Regulus debit transactions; add retry-count field to future records |
| API versioning removed unversioned paths | HIGH | Add unversioned path aliases immediately; notify clients; cannot patch deployed client integrations retroactively |
| GovernAI local path blocking Docker build | LOW | One-line `pyproject.toml` change to git ref; `docker build` unblocked within minutes |
| Webhook fire-and-forget dropped events | MEDIUM | Query run status directly for the affected time window; emit compensating notifications manually; add the delivery table and backfill |
| Docker socket mounted on API container | HIGH | Rotate all credentials visible to any container on that host; redeploy with socket mount removed; audit Docker daemon logs for evidence of socket abuse |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Sync repository layer converted to async (Pitfall 1) | Postgres migration | `uv run pytest -v` passes with zero test changes |
| Telemetry SDK corrupting audit chain (Pitfall 2) | Regulus integration | `AuditContinuityReport` has zero gaps with Regulus enabled |
| Retry logic double-counting tokens (Pitfall 3) | LLM provider adapter | Every retry attempt produces an audit sub-entry with own token counts |
| Test suite contamination from real providers (Pitfall 4) | LLM provider adapter | CI run without `ZEROTH_LIVE_TESTS` completes in <60s with zero network calls |
| Docker sandbox container escape (Pitfall 5) | Container sandbox hardening + Dockerfile | No `--privileged` flag; no Docker socket mount on API container |
| Message queue replacing lease durability (Pitfall 6) | Durable dispatch enhancement | Orphan recovery test passes after simulated worker crash |
| API versioning breaking existing paths (Pitfall 7) | API versioning | All pre-existing `tests/service/` tests pass with zero URL changes |
| Webhook fire-and-forget delivery (Pitfall 8) | Webhook notification | Dead-letter test: slow receiver → retry → eventual DLQ entry |
| GovernAI local path blocking Docker (Pitfall 9) | Dockerfile (prerequisite) | `docker build` succeeds on a machine without `/Users/dondoe/coding/governai` |
| In-memory metrics masking multi-worker state (Pitfall 10) | Horizontal scaling | Aggregate metrics endpoint reflects counts from all N workers |

---

## Sources

- Zeroth codebase direct inspection: `src/zeroth/` (April 2026) — HIGH confidence
- `.planning/codebase/CONCERNS.md` (April 2026) — HIGH confidence
- `.planning/codebase/ARCHITECTURE.md` (April 2026) — HIGH confidence
- [Retries, fallbacks, and circuit breakers in LLM apps](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/) — MEDIUM confidence
- [Webhook Infrastructure Guide](https://hookdeck.com/webhooks/guides/webhook-infrastructure-guide) — MEDIUM confidence
- [Versioning REST APIs in FastAPI Without Breaking Old Clients](https://medium.com/@bhagyarana80/versioning-rest-apis-in-fastapi-without-breaking-old-clients-736f75e7dd6e) — MEDIUM confidence
- [asyncpg FAQ — pgbouncer prepared statement issues](https://magicstack.github.io/asyncpg/current/faq.html) — HIGH confidence
- [OpenTelemetry asyncio context propagation](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/asyncio/asyncio.html) — HIGH confidence
- [Docker Sandboxes — host kernel sharing](https://unit42.paloaltonetworks.com/making-containers-more-isolated-an-overview-of-sandboxed-container-technologies/) — MEDIUM confidence
- [Redis as Message Broker — durability limits](https://dev.to/nileshprasad137/redis-as-a-message-broker-deep-dive-3oek) — MEDIUM confidence
- [AI workload observability cost crisis](https://oneuptime.com/blog/post/2026-04-01-ai-workload-observability-cost-crisis/view) — MEDIUM confidence

---

*Pitfalls research for: Zeroth v1.1 Production Readiness — adding production infrastructure to an existing governed multi-agent platform*
*Researched: 2026-04-06*
