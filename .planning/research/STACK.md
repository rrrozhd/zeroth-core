# Stack Research: v4.0 Platform Extensions

**Domain:** Agent orchestration platform -- parallel execution, composition, artifact stores, context management, resilient HTTP, prompt templates, computed mappings
**Researched:** 2026-04-12
**Confidence:** HIGH

## Executive Summary

All 7 platform extensions can be built with **zero new PyPI dependencies**. The existing dependency tree already provides every library needed: `asyncio.TaskGroup` for parallel execution (stdlib, Python 3.12), `litellm.token_counter` for context window management (litellm already pinned), `httpx.AsyncClient` with `tenacity` for resilient HTTP (both already pinned), `Jinja2` for prompt templates (already a transitive dependency via litellm), and the existing AST-based `_SafeEvaluator` for computed mappings. Circuit breaking is lightweight enough to implement in-house (~60 lines) rather than pulling in an unmaintained library.

This is the optimal outcome. No version conflicts, no new supply-chain risk, no optional-extras proliferation.

## Existing Stack (DO NOT change)

These are already in `pyproject.toml` and verified in the environment:

| Technology | Installed Version | Purpose | Relevant to v4.0 |
|------------|-------------------|---------|-------------------|
| Python | 3.12+ | Runtime | `asyncio.TaskGroup` for parallel fan-out/fan-in |
| httpx | 0.28.1 | HTTP client | Resilient HTTP client (Feature 5) |
| tenacity | 9.1.4 | Retry logic | HTTP retry with backoff (Feature 5) |
| cachetools | 7.0.5 | In-memory caching | Response caching for HTTP client (Feature 5) |
| litellm | 1.83.0 | LLM routing | `token_counter()` / `acount_tokens()` for context management (Feature 4) |
| tiktoken | 0.12.0 | Tokenization | Transitive via litellm; local fallback tokenizer |
| Jinja2 | 3.1.6 | Template engine | Transitive via litellm; prompt template rendering (Feature 6) |
| redis | 5.3.1 | KV store | Artifact store backend (Feature 3) |
| pydantic | 2.12.5 | Data models | All new models and contracts |
| pydantic-settings | 2.13+ | Configuration | Settings for new subsystems |
| FastAPI | 0.135.1 | API framework | Any new endpoints |
| SQLAlchemy | 2.0.49 | ORM/DB | Persistence for templates, artifacts metadata |
| asyncio (stdlib) | 3.12 | Concurrency | `TaskGroup` for fan-out/fan-in |

## Feature-by-Feature Stack Mapping

### Feature 1: Parallel Fan-Out / Fan-In Execution

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Concurrent branch spawning | `asyncio.TaskGroup` (stdlib) | Built into Python 3.12; provides structured concurrency with automatic cancellation on failure; stronger guarantees than `asyncio.gather()` |
| Synchronization barrier | `asyncio.TaskGroup` exit + result collection | `async with` block ensures all branches complete before fan-in proceeds |
| Per-branch isolation | Dict-based context copying | Each branch gets a deep copy of the input payload; no shared mutable state |
| Budget tracking per branch | Existing `BudgetEnforcer` + `CostEstimator` | Already wired into `RuntimeOrchestrator._dispatch_node`; each branch runs through the same instrumented path |

**Integration points:**
- `RuntimeOrchestrator._drive()` loop needs a new code path: when a `ParallelNode` is encountered, spawn N branches via `TaskGroup` instead of processing sequentially
- `Run.pending_node_ids` needs extension to represent parallel branch sets (e.g., list-of-lists or a `ParallelBranchSet` model)
- `RunHistoryEntry` needs branch indexing for audit trail ordering
- Existing `_SafeEvaluator` handles merge expressions for fan-in result combination

**Why NOT `asyncio.gather()`:** `TaskGroup` cancels remaining tasks on first exception (fail-fast), which matches the existing `failure_policy: "fail_fast"` default in `ExecutionSettings`. `gather()` with `return_exceptions=True` collects all errors but lets failing branches waste resources.

### Feature 2: Subgraph Composition

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Nested graph invocation | `RuntimeOrchestrator.run_graph()` (existing) | Already supports running a graph with initial input; subgraph invocation is a recursive call |
| Graph resolution | `GraphRepository` (existing) | Published graphs can be looked up by reference |
| Governance inheritance | `PolicyGuard` + `policy_bindings` (existing) | Parent graph policies propagate to child via policy binding merge |
| Thread continuity | `thread_id` passthrough (existing) | `run_graph()` already accepts `thread_id`; pass parent's thread to child |

**Integration points:**
- New `SubgraphNode` type alongside `AgentNode`, `ExecutableUnitNode`, `HumanApprovalNode`
- `RuntimeOrchestrator._dispatch_node()` gains a `SubgraphNode` handler that calls `run_graph()` recursively
- Contract validation: subgraph's entry node input contract must be compatible with the parent edge's output
- Governance: merge parent `policy_bindings` with child graph's own bindings (parent wins on conflict)

### Feature 3: Large Payload Externalization (Artifact Store)

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Redis backend | `redis` 5.3.1 (existing) | Already in `dispatch` optional extra; `SETEX` provides TTL natively |
| Filesystem backend | `pathlib` + `asyncio.to_thread()` (stdlib) | Local dev/test; no network dependency |
| Artifact reference model | `pydantic` (existing) | `ArtifactRef` model with `store`, `key`, `ttl`, `content_hash` |
| Content hashing | `hashlib` (stdlib) | SHA-256 for integrity verification; zero-dependency |

**Integration points:**
- `ArtifactStore` protocol with `store(key, data, ttl) -> ArtifactRef` and `retrieve(ref) -> bytes`
- `RedisArtifactStore` uses existing `RedisConfig` / `RedisSettings` -- shared connection pool
- `FilesystemArtifactStore` for local dev (writes to `$ZEROTH_DATA_DIR/artifacts/`)
- Edge mapping system extended: when payload exceeds threshold, auto-externalize and replace with `ArtifactRef`
- Audit trail records artifact references (not the large payload itself)
- New `ArtifactSettings` in `ZerothSettings` for thresholds and backend selection

**Note on `aiofiles`:** Not needed. Python's `pathlib.Path.write_bytes()` is sync but non-blocking for typical artifact sizes (< 100MB). Use `asyncio.to_thread()` for the write path to avoid blocking the event loop.

### Feature 4: Agent Context Window Management

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Token counting | `litellm.token_counter()` / `litellm.acount_tokens()` | Already pinned; supports OpenAI, Anthropic, Llama, Cohere tokenizers; falls back to tiktoken for unknown models |
| Model context limits | `litellm.model_cost` / `litellm.get_max_tokens()` | litellm maintains a model cost/limits database; no need to hardcode limits |
| Summarization strategy | `AgentRunner` + LLM call | Use the same `ProviderAdapter` infrastructure to call a summarizer model |
| Token budget tracking | Custom `ContextWindowManager` class | Lightweight class that wraps `token_counter()` and enforces per-node or per-thread budgets |

**Integration points:**
- `PromptAssembler.assemble()` gains a `max_context_tokens` parameter; truncates or summarizes thread state / memory when budget exceeded
- `AgentConfig` gets `context_window_strategy: Literal["truncate", "summarize", "error"]` field
- `AgentConfig` gets `max_context_tokens: int | None` field (defaults to model's max minus output reservation)
- Token count is recorded in `AgentRunResult.audit_record` for observability
- Summarization uses a dedicated summarizer agent config (lightweight model, e.g. `openai/gpt-4o-mini`)

**Why NOT tiktoken directly:** `litellm.token_counter()` already uses tiktoken internally but adds model-specific tokenizer selection. Calling tiktoken directly would duplicate logic and miss provider-specific tokenizers (Anthropic, Cohere). Since litellm is already pinned, use its higher-level API.

### Feature 5: Resilient External HTTP Client

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Async HTTP | `httpx.AsyncClient` 0.28.1 (existing) | Already a core dependency; async, connection pooling, timeouts built-in |
| Retry with backoff | `tenacity` 9.1.4 (existing) | Already pinned; `@retry` decorator with exponential backoff, jitter, custom predicates |
| Response caching | `cachetools.TTLCache` 7.0.5 (existing) | Already pinned; simple in-memory TTL cache for GET responses |
| Circuit breaking | Custom implementation (~60 LOC) | All async circuit breaker libraries are unmaintained (aiobreaker last release >12 months ago); pattern is simple enough to implement in-house with `asyncio.Lock` |
| Connection pooling | `httpx.Limits` (existing) | Built into httpx; `max_connections`, `max_keepalive_connections`, `keepalive_expiry` |

**Integration points:**
- New `ResilientHTTPClient` class wrapping `httpx.AsyncClient` with retry, circuit breaker, and caching layers
- Configuration via new `HTTPClientSettings` in `ZerothSettings`: `max_retries`, `backoff_base`, `backoff_max`, `circuit_breaker_threshold`, `circuit_breaker_reset_timeout`, `cache_ttl`, pool limits
- Capability-gated: tool calls and executable units that make external HTTP requests go through this client
- Audit integration: every HTTP request/response is logged to `AuditRepository` (URL, status, latency, retry count)
- Governor policy can restrict allowed domains via capability bindings

**Why NOT add a circuit breaker library:**
- `aiobreaker` 1.2.0: Last release >12 months ago, no recent GitHub activity, project appears abandoned
- `pybreaker` 1.2.1: Synchronous-first, Tornado-based async support -- wrong model for native asyncio
- `circuitbreaker` 2.0.0: Decorator-based, doesn't fit the class-based client wrapper pattern
- Circuit breaker is ~60 lines of code: failure counter, threshold check, half-open probe, reset timer. Adding a dependency for this creates more risk (unmaintained transitive) than value.

**httpx retry transport vs tenacity:** httpx's `AsyncHTTPTransport(retries=N)` only retries on `ConnectError` and `ConnectTimeout`. It does NOT retry on HTTP 429/500/502/503/504 responses. `tenacity` retries on arbitrary predicates (status codes, exception types), making it the right tool for resilient HTTP.

### Feature 6: Prompt Template Management

**New dependencies: NONE** (Jinja2 promoted to explicit dep, but already installed)

| Component | Technology | Why |
|-----------|-----------|-----|
| Template rendering | `Jinja2` 3.1.6 (transitive via litellm) | Industry standard; already installed; sandboxed rendering mode prevents code injection |
| Template storage | `SQLAlchemy` + existing DB (existing) | Versioned template records alongside existing graph/run tables |
| Variable validation | `pydantic` (existing) | Template variable schema as Pydantic model |
| Audit redaction | Existing `AgentAuditSerializer._redact_value()` | Already handles secret redaction in prompts |

**Integration points:**
- New `PromptTemplate` Pydantic model: `template_id`, `version`, `content` (Jinja2 string), `variable_schema` (JSON Schema), `metadata`
- `PromptTemplateRegistry` for CRUD + version lookup (similar pattern to `GraphRepository`)
- `PromptAssembler.assemble()` extended: when `AgentNodeData` references a template instead of inline instruction, resolve and render it
- `AgentNodeData.instruction` becomes `instruction: str | None` and gains `template_ref: str | None` -- exactly one must be set
- Jinja2 `SandboxedEnvironment` used for rendering (prevents `{{ config.__class__.__init__ }}` attacks)
- Rendered templates are redacted before audit logging using existing `_redact_value()`

**Why Jinja2 and not Mustache/Mako/string.Template:**
- Jinja2 is already installed (transitive dep); adding it to `dependencies` makes it explicit but changes nothing at install time
- `SandboxedEnvironment` is purpose-built for untrusted template input -- critical for user-authored prompts
- Supports conditionals, loops, filters -- needed for complex prompt templates (e.g., "include tool descriptions only if tools are attached")
- Industry standard in Python ecosystem; lowest learning curve

### Feature 7: Computed Data Mappings

**New dependencies: NONE**

| Component | Technology | Why |
|-----------|-----------|-----|
| Expression evaluation | Existing `_SafeEvaluator` in `conditions/evaluator.py` | Already handles arithmetic, string ops, comparisons, subscript, dict/list construction -- exactly what computed mappings need |
| Mapping operation | New `TransformMappingOperation` model | Extends existing `MappingOperation` discriminated union |
| Execution | Extend `MappingExecutor._apply_operation()` | Add `TransformMappingOperation` case to the existing match statement |

**Integration points:**
- New `TransformMappingOperation(MappingOperationBase)` with `operation: Literal["transform"]`, `expression: str`, `source_paths: list[str]`
- `MappingExecutor._apply_operation()` gains one new `case TransformMappingOperation():` branch
- Expression namespace populated from source paths resolved against the input payload
- `MappingOperation` discriminated union extended: adds `TransformMappingOperation` to existing 4-variant union
- Validation: `MappingValidator` checks that `expression` parses as valid AST and `source_paths` exist in the input contract

**Why reuse `_SafeEvaluator`:** It already supports the exact operations needed for data transformation: arithmetic (`+`, `-`, `*`, `/`, `%`), string concatenation via `+`, dict/list/tuple/set construction, subscript access, attribute access, comparisons, boolean logic, and ternary expressions. It blocks function calls, imports, and attribute mutation -- exactly the safety boundary needed for edge mappings.

## What NOT to Add

| Avoid | Why | What to Use Instead |
|-------|-----|---------------------|
| `aiobreaker` / `pybreaker` / `circuitbreaker` | All unmaintained or wrong async model; simple pattern doesn't justify a dependency | Custom ~60 LOC circuit breaker using `asyncio.Lock` + failure counter |
| `tiktoken` as direct dependency | Already a transitive dep via litellm; `litellm.token_counter()` wraps it with model-specific routing | `litellm.token_counter()` / `litellm.acount_tokens()` |
| `aiofiles` | Overkill for artifact store file writes; `asyncio.to_thread(pathlib.write_bytes)` is sufficient | `pathlib` + `asyncio.to_thread()` |
| `celery` / `dramatiq` for parallel execution | Massive dependency; ARQ already handles distributed dispatch; parallel branches are in-process concurrent tasks, not distributed jobs | `asyncio.TaskGroup` for in-process parallelism; existing ARQ for distributed dispatch |
| `networkx` for subgraph composition | Graph validation is already implemented in `Graph._validate_references()`; subgraph resolution is a simple recursive call | Existing graph models + `RuntimeOrchestrator.run_graph()` |
| `jsonpath-ng` / `jmespath` for computed mappings | Existing `_SafeEvaluator` + `_path_lookup()` already handles dot-path resolution and expression evaluation | Existing condition engine |
| `Mako` / `Mustache` / `string.Template` | Jinja2 already installed, has `SandboxedEnvironment`, and is the Python ecosystem standard | `Jinja2` with `SandboxedEnvironment` |

## Dependency Changes to pyproject.toml

### Explicit Dependencies to Add

None of these introduce new packages to install -- they promote transitive dependencies to explicit ones for stability:

```toml
# In [project] dependencies, ADD:
"Jinja2>=3.1.2,<4.0",    # Prompt template rendering; already transitive via litellm
```

**Rationale:** Jinja2 is currently only a transitive dependency (via litellm). If litellm ever drops it, our prompt template feature would break silently. Pinning it explicitly in `dependencies` costs nothing (it's already installed) and ensures the dependency survives upstream changes.

**Everything else stays as-is:** `httpx`, `tenacity`, `cachetools`, `litellm`, `redis`, `pydantic`, `pydantic-settings` are all already explicit dependencies at appropriate version ranges.

### No Changes Needed

The following are already correctly pinned:
- `httpx>=0.27` -- covers 0.28.1 with `Limits`, `AsyncHTTPTransport`
- `tenacity>=8.2` -- covers 9.1.4 with `AsyncRetrying`
- `cachetools>=5.5` -- covers 7.0.5 with `TTLCache`
- `litellm>=1.83,<2.0` -- covers `token_counter()`, `acount_tokens()`, `get_max_tokens()`
- `redis>=5.0.0` (in `dispatch` extra) -- covers `SETEX` for artifact TTL

## Existing Components to Extend (Not Replace)

| Component | File | Extension Needed |
|-----------|------|-----------------|
| `Graph` model | `graph/models.py` | Add `ParallelNode`, `SubgraphNode` to `Node` union |
| `Edge` model | `graph/models.py` | No change; edges already connect to any node type |
| `EdgeMapping` | `mappings/models.py` | Add `TransformMappingOperation` to `MappingOperation` union |
| `MappingExecutor` | `mappings/executor.py` | Add `case TransformMappingOperation()` branch |
| `_SafeEvaluator` | `conditions/evaluator.py` | No change; already supports all needed operations |
| `RuntimeOrchestrator` | `orchestrator/runtime.py` | Add `ParallelNode` and `SubgraphNode` handlers in `_dispatch_node()` and `_drive()` |
| `Run` model | `runs/models.py` | Add parallel branch tracking fields |
| `AgentConfig` | `agent_runtime/models.py` | Add `context_window_strategy`, `max_context_tokens`, `template_ref` fields |
| `AgentNodeData` | `graph/models.py` | Add `template_ref: str | None` field |
| `PromptAssembler` | `agent_runtime/prompt.py` | Add template resolution and context window truncation |
| `ZerothSettings` | `config/settings.py` | Add `ArtifactSettings`, `HTTPClientSettings`, `ContextWindowSettings` |
| `RedisConfig` | `storage/redis.py` | Shared for artifact store Redis backend (no changes needed) |

## New Modules to Create

| Module | Purpose | Dependencies |
|--------|---------|-------------|
| `zeroth.core.parallel` | `ParallelNode`, `ParallelBranchSet`, fan-out/fan-in orchestration | `asyncio`, existing orchestrator |
| `zeroth.core.subgraph` | `SubgraphNode`, governance inheritance, recursive invocation | Existing orchestrator, graph repo |
| `zeroth.core.artifacts` | `ArtifactStore` protocol, `RedisArtifactStore`, `FilesystemArtifactStore`, `ArtifactRef` | `redis`, `pathlib`, `hashlib` |
| `zeroth.core.context` | `ContextWindowManager`, summarization strategies, token budget tracking | `litellm` |
| `zeroth.core.http_client` | `ResilientHTTPClient`, circuit breaker, retry wrapper, cache layer | `httpx`, `tenacity`, `cachetools` |
| `zeroth.core.templates` | `PromptTemplate`, `PromptTemplateRegistry`, Jinja2 rendering | `jinja2`, `sqlalchemy` |

## Version Compatibility Matrix

| Package | Pinned Range | Installed | Required Feature | Compatible |
|---------|-------------|-----------|-----------------|------------|
| Python | >=3.12 | 3.12+ | `asyncio.TaskGroup` (3.11+) | YES |
| httpx | >=0.27 | 0.28.1 | `Limits`, `AsyncHTTPTransport`, `AsyncClient` | YES |
| tenacity | >=8.2 | 9.1.4 | `AsyncRetrying`, `retry_if_exception_type` | YES |
| cachetools | >=5.5 | 7.0.5 | `TTLCache` | YES |
| litellm | >=1.83,<2.0 | 1.83.0 | `token_counter()`, `acount_tokens()`, `get_max_tokens()` | YES |
| tiktoken | (transitive) | 0.12.0 | Used internally by litellm | YES |
| Jinja2 | (transitive) | 3.1.6 | `SandboxedEnvironment`, `Template` | YES |
| redis | >=5.0.0 | 5.3.1 | `SETEX`, async support | YES |
| pydantic | >=2.10 | 2.12.5 | Discriminated unions, `model_copy` | YES |

## Sources

- [LiteLLM Token Counting docs](https://docs.litellm.ai/docs/count_tokens) -- verified `token_counter()` and `acount_tokens()` APIs, model support (HIGH confidence)
- [httpx Resource Limits docs](https://www.python-httpx.org/advanced/resource-limits/) -- verified `Limits` class, connection pooling (HIGH confidence)
- [httpx Transports docs](https://www.python-httpx.org/advanced/transports/) -- verified `AsyncHTTPTransport` retry limitation (connection-level only) (HIGH confidence)
- [Jinja2 PyPI](https://pypi.org/project/Jinja2/) -- verified v3.1.6 latest stable (HIGH confidence)
- [tiktoken GitHub](https://github.com/openai/tiktoken) -- verified v0.12.0 (HIGH confidence)
- [tenacity PyPI](https://pypi.org/project/tenacity/) -- verified v9.1.4 latest (HIGH confidence)
- [aiobreaker Snyk analysis](https://snyk.io/advisor/python/aiobreaker) -- verified unmaintained status (MEDIUM confidence)
- [Python asyncio.TaskGroup docs](https://docs.python.org/3/library/asyncio-task.html) -- verified structured concurrency API (HIGH confidence)
- Local environment verification via `uv run python` -- all versions confirmed installed (HIGH confidence)

---
*Stack research for: zeroth-core v4.0 platform extensions*
*Researched: 2026-04-12*
