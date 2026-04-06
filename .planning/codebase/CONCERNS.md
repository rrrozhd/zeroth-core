# Codebase Concerns

**Analysis Date:** 2026-04-05

## Tech Debt

**Local GovernAI path dependency:**
- Issue: `pyproject.toml` pins `governai @ file:///Users/dondoe/coding/governai` -- a hardcoded local filesystem path. This breaks CI, onboarding, and any machine that does not replicate this exact directory layout.
- Files: `pyproject.toml` (line 8)
- Impact: Cannot build or install zeroth on any other machine without manually cloning governai to the same path. No version pinning or integrity checking on the dependency.
- Fix approach: Publish governai to a private PyPI index or use a git+https reference with a tag/commit pin. At minimum, document the local setup requirement.

**In-memory memory connectors only:**
- Issue: All memory connectors (`RunEphemeralMemoryConnector`, `KeyValueMemoryConnector`, `ThreadMemoryConnector`) store data in plain Python dicts. Data is lost on process restart.
- Files: `src/zeroth/memory/connectors.py`
- Impact: Agent memory does not survive process restarts. Multi-worker deployments cannot share memory state. The code comments explicitly acknowledge this: "These are the MVP implementations; production versions would use a real database."
- Fix approach: Implement SQLite-backed or Redis-backed connectors that conform to the `MemoryConnector` protocol. The protocol-based design makes this straightforward.

**In-memory environment cache in sandbox:**
- Issue: `EnvironmentCacheManager` stores prepared sandbox environments in a plain dict. Cache is lost on restart, and cannot be shared across workers.
- Files: `src/zeroth/execution_units/sandbox.py` (lines 219-274)
- Impact: Every process restart rebuilds all sandbox environments from scratch. Not a correctness issue, but a performance concern at scale.
- Fix approach: Back the cache with SQLite or Redis for persistence across restarts.

**In-memory metrics collector:**
- Issue: `MetricsCollector` stores all counters, gauges, and histograms in thread-local Python dicts. No external metrics sink (Prometheus, Datadog, etc.) is integrated.
- Files: `src/zeroth/observability/metrics.py`
- Impact: Metrics are lost on process restart. No external monitoring or alerting capability. The Prometheus text renderer exists but nothing scrapes it.
- Fix approach: Integrate with a real metrics backend (Prometheus push gateway, StatsD, or OpenTelemetry exporter).

**Default InMemoryThreadStateStore in AgentRunner:**
- Issue: `AgentRunner.__init__` defaults to `InMemoryThreadStateStore()` when no store is provided. Thread conversation state is lost on restart.
- Files: `src/zeroth/agent_runtime/runner.py` (line 72), `src/zeroth/agent_runtime/models.py` (lines 164-194)
- Impact: Multi-turn agent conversations lose context on process restart unless the caller explicitly provides a persistent store. The `RepositoryThreadResolver` exists as the production alternative but must be manually wired.
- Fix approach: Make thread_state_store a required parameter or default to the SQLite-backed `RepositoryThreadStateStore` from `src/zeroth/agent_runtime/thread_store.py`.

## Known Bugs

**No known bugs identified from static analysis.** The codebase has thorough test coverage (78 test files) and the code paths examined are well-structured. Phase 8A sandbox hardening has one incomplete item (making hardened sandbox the default for untrusted units) but this is tracked in PROGRESS.md.

## Security Considerations

**No TLS termination in the service layer:**
- Risk: The FastAPI app served via uvicorn has no TLS configuration. All traffic including API keys and bearer tokens travels in plaintext unless a reverse proxy handles TLS.
- Files: `src/zeroth/service/app.py`, `src/zeroth/service/auth.py`
- Current mitigation: None in the application layer.
- Recommendations: Document that a TLS-terminating proxy (nginx, Caddy, cloud load balancer) is required for production. Optionally add uvicorn SSL config parameters.

**Static API keys stored in environment JSON:**
- Risk: `ServiceAuthConfig.from_env()` reads `ZEROTH_SERVICE_API_KEYS_JSON` which contains credential secrets as a JSON blob in an environment variable. This is a common pattern but environment variables can leak through process listings, crash dumps, or logging.
- Files: `src/zeroth/service/auth.py` (lines 69-76)
- Current mitigation: API key secrets are compared as plain strings, not hashed.
- Recommendations: Hash stored API key secrets (bcrypt/argon2). Consider a secrets manager integration for production credential storage.

**JWKS fetched via plain urlopen:**
- Risk: `JWTBearerTokenVerifier._load_jwks()` uses `urllib.request.urlopen` without timeout, certificate verification customization, or caching. A slow or malicious JWKS endpoint could block the auth path.
- Files: `src/zeroth/service/auth.py` (line 114)
- Current mitigation: JWKS can be passed inline via config to avoid network fetch.
- Recommendations: Add timeout, use httpx (already a dependency), cache JWKS with TTL.

**Sandbox local backend does not enforce resource constraints:**
- Risk: When running in `LOCAL` mode, the sandbox warns but does not enforce CPU, memory, disk, process count, or network access constraints. Untrusted code running locally has full host access.
- Files: `src/zeroth/execution_units/sandbox.py` (lines 635-657)
- Current mitigation: `SandboxStrictnessMode.STANDARD` and `STRICT` raise `SandboxPolicyViolationError` when constraints require hard isolation but Docker is unavailable.
- Recommendations: Complete Phase 8A to make Docker the default for untrusted workloads. Never run untrusted code in LOCAL mode with PERMISSIVE strictness.

**Encryption key for SQLiteDatabase is optional:**
- Risk: `SQLiteDatabase` accepts an optional `encryption_key` for `EncryptedField`. If omitted, no field-level encryption occurs for secrets stored in SQLite.
- Files: `src/zeroth/storage/sqlite.py` (line 56-59)
- Current mitigation: `SecretResolver` and `SecretRedactor` strip secrets from audit/checkpoint data.
- Recommendations: Require encryption_key in production bootstrap. Validate that no raw secret values leak into unencrypted columns.

## Performance Bottlenecks

**SQLite as sole persistence layer:**
- Problem: All repositories (runs, threads, graphs, deployments, approvals, audit, contracts, rate limits, quotas, leases) use SQLite. While WAL mode is enabled, SQLite has a single-writer constraint.
- Files: `src/zeroth/storage/sqlite.py`, `src/zeroth/runs/repository.py` (1102 lines), `src/zeroth/dispatch/lease.py`, `src/zeroth/guardrails/rate_limit.py`
- Cause: Write contention under concurrent workloads. The lease manager, rate limiter, quota enforcer, and run repository all compete for the write lock.
- Improvement path: For production: migrate critical-path tables (runs, leases, rate_limit_buckets) to PostgreSQL or Redis. SQLite remains fine for graph/contract/deployment storage which is read-heavy.

**Synchronous graph node lookup is O(n):**
- Problem: `RuntimeOrchestrator._node_by_id()` and `_edge_for()` iterate through all nodes/edges linearly for every lookup.
- Files: `src/zeroth/orchestrator/runtime.py` (lines 762-774)
- Cause: Nodes and edges are stored as lists on the Graph model, not indexed by ID.
- Improvement path: Build lookup dicts (`{node_id: node}`, `{(source, target): edge}`) once at graph load time. Low effort, high impact for large graphs.

**Run repository is the largest file (1102 lines):**
- Problem: `src/zeroth/runs/repository.py` contains all run and thread persistence logic, multiple migration scripts, and serialization helpers in one file. Complex to navigate and modify.
- Files: `src/zeroth/runs/repository.py`
- Cause: Organic growth across phases 1-9 without extraction.
- Improvement path: Extract migrations to a separate module. Split RunRepository and ThreadRepository into separate files. Extract serialization helpers.

## Fragile Areas

**Orchestrator runtime loop:**
- Files: `src/zeroth/orchestrator/runtime.py` (774 lines)
- Why fragile: The `_drive()` method is a complex state machine that handles node dispatch, policy enforcement, approval gates, side-effect gating, branch planning, history recording, and failure handling in a single while loop. Many code paths interact with run metadata as a mutable dict.
- Safe modification: Always test with the full orchestrator test suite (`tests/orchestrator/`). Be careful with `run.metadata` mutations -- multiple methods read/write the same keys ("node_payloads", "pending_approval", "approved_side_effect_nodes", "enforcement", "edge_visit_counts", "path").
- Test coverage: Good coverage through `tests/orchestrator/` but the metadata-as-dict pattern makes it easy to introduce key naming conflicts.

**Run metadata as untyped dict:**
- Files: `src/zeroth/runs/repository.py`, `src/zeroth/orchestrator/runtime.py`
- Why fragile: `run.metadata` is `dict[str, Any]` used for node_payloads, edge_visit_counts, path, enforcement, pending_approval, approved_side_effect_nodes, terminal_reason, audits, last_output, and more. No schema enforcement on keys.
- Safe modification: Grep for the metadata key name across the entire codebase before adding or renaming keys. Consider extracting typed metadata sub-models.
- Test coverage: Implicitly tested through orchestrator and run API tests, but no explicit metadata-shape validation tests.

**Bootstrap wiring:**
- Files: `src/zeroth/service/bootstrap.py` (200 lines)
- Why fragile: `bootstrap_service()` manually wires 20+ dependencies. Adding new subsystems requires touching this function and the `ServiceBootstrap` dataclass. Phase 9 additions are already marked as optional to avoid breaking existing tests.
- Safe modification: Add new fields as optional with `None` default. Always add corresponding wiring in `bootstrap_service()`.
- Test coverage: Tested indirectly through service integration tests in `tests/service/`.

## Scaling Limits

**Single-process worker model:**
- Current capacity: One `RunWorker` per process, bounded by `max_concurrency` (default 8 concurrent runs).
- Limit: Throughput capped by single-process SQLite write contention and Python GIL for CPU-bound operations.
- Scaling path: Deploy multiple worker processes (each gets its own worker ID and competes for leases via SQLite atomic updates). For higher scale, migrate lease storage to Redis or PostgreSQL for better write concurrency.

**SQLite write throughput:**
- Current capacity: ~50-200 writes/second depending on hardware (WAL mode).
- Limit: Every run step writes to runs table, run_checkpoints, and audit. Rate limiter and quota enforcer also write per-request. Under high concurrency, SQLite becomes the bottleneck.
- Scaling path: PostgreSQL for run/lease/audit tables. Keep SQLite for read-heavy graph/contract storage.

## Dependencies at Risk

**governai (local path dependency):**
- Risk: Unpublished, unversioned dependency. Any breaking change in the local governai repo immediately breaks zeroth. No way to pin to a known-good version.
- Impact: All core abstractions depend on governai: `GovernedLLM`, `RunState`, `RunStatus`, `GovernedFlowSpec`, `GovernedStepSpec`, `TransitionSpec`, `Tool`, `PythonTool`, `PythonHandler`, `RedisRunStore`, `RedisInterruptStore`, `RedisAuditEmitter`, `ExecutionPlacement`, `NormalizedToolCall`, `extract_tool_calls`, `build_tool_message`.
- Migration plan: Pin to a git commit hash at minimum. Publish to a private index for production use.

**cryptography (for Fernet encryption):**
- Risk: Low. Well-maintained, widely used. But Fernet has a 32-byte key requirement and its own versioning -- key rotation is not handled by the current `EncryptedField` implementation.
- Impact: `src/zeroth/storage/sqlite.py` uses it for field-level encryption.
- Migration plan: None needed, but add key rotation support before production use.

## Missing Critical Features

**No real LLM provider integration:**
- Problem: The only production-ready provider adapter is `GovernedLLMProviderAdapter` which wraps a `GovernedLLM` instance. There are no concrete OpenAI, Anthropic, or other LLM SDK integrations. The `DeterministicProviderAdapter` is test-only. The `CallableProviderAdapter` is a generic wrapper.
- Files: `src/zeroth/agent_runtime/provider.py`
- Blocks: Running agents against real LLM APIs. The platform cannot execute real AI workloads without external wiring of a GovernedLLM instance.

**No cost/token tracking:**
- Problem: The platform has no mechanism to track LLM token usage, cost per run, or budget enforcement. The `ProviderResponse` model does not include usage/token fields. The audit records do not capture token counts.
- Files: `src/zeroth/agent_runtime/provider.py`, `src/zeroth/agent_runtime/models.py`
- Blocks: Economic viability tracking, per-tenant billing, cost-aware routing -- all stated platform goals.

**No Dockerfile or container packaging:**
- Problem: No Dockerfile, docker-compose.yml, or .dockerignore exists. The platform cannot be deployed as a container without manual packaging.
- Files: Project root (missing files)
- Blocks: Production deployment, CI/CD pipelines, reproducible environments.

**No CI/CD pipeline:**
- Problem: No `.github/workflows/`, no `Makefile`, no CI configuration of any kind.
- Files: Project root (missing files)
- Blocks: Automated testing, quality gates, deployment automation.

**No configuration management:**
- Problem: No config file loading (YAML/TOML), no environment-specific profiles, no config validation at startup. Auth config uses env var JSON blobs. Redis config uses env vars with a prefix convention. No unified config surface.
- Files: `src/zeroth/service/auth.py` (from_env), `src/zeroth/storage/redis.py` (from_env)
- Blocks: Clean production deployment, environment-specific configuration, config auditing.

**Studio authoring APIs not wired:**
- Problem: `src/zeroth/studio/` contains lease and workflow models for the Studio authoring UI, but no HTTP routes or API endpoints expose this functionality. The FastAPI app in `src/zeroth/service/app.py` only registers run, approval, audit, and contract routes.
- Files: `src/zeroth/studio/`, `src/zeroth/service/app.py`
- Blocks: Phase 10 Studio UI cannot call backend authoring APIs until these are wired as FastAPI routes.

## Test Coverage Gaps

**No integration tests with real LLM providers:**
- What's not tested: All agent runtime tests use `DeterministicProviderAdapter` with pre-canned responses. The `GovernedLLMProviderAdapter` is tested with mock objects, never with a real LLM call.
- Files: `tests/agent_runtime/`
- Risk: Provider adapter edge cases (rate limits, streaming, malformed responses, network failures) are untested against real APIs.
- Priority: Medium -- the adapter interface is simple, but real-world LLM responses have surprising shapes.

**No load/stress tests:**
- What's not tested: Concurrent run execution, SQLite write contention, lease contention under multiple workers, rate limiter accuracy under burst traffic.
- Files: No load test infrastructure exists.
- Risk: Performance bottlenecks and race conditions only discoverable under load.
- Priority: Medium -- important before production deployment.

**Sandbox Docker backend not tested in CI:**
- What's not tested: Docker-based sandbox execution is tested with mock command runners, but no CI job actually runs commands inside Docker containers.
- Files: `tests/execution_units/test_sandbox.py`, `tests/execution_units/test_sandbox_hardening.py`
- Risk: Docker path rewriting, volume mounting, and resource flag translation could have bugs that only manifest with real Docker.
- Priority: Low for MVP, High before running untrusted code in production.

**Run metadata key collisions not validated:**
- What's not tested: No test verifies that different orchestrator subsystems (branch planner, approval gates, policy enforcement) do not collide on `run.metadata` keys.
- Files: `src/zeroth/orchestrator/runtime.py`
- Risk: A new feature adding a metadata key that shadows an existing one could silently corrupt run state.
- Priority: Low -- unlikely with current codebase discipline, but risk grows with team size.

---

*Concerns audit: 2026-04-05*
