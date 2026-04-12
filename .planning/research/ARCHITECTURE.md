# Architecture Research -- v4.0 Platform Extensions Integration

**Domain:** Agent orchestration platform extensions (parallel execution, composition, artifacts, context, HTTP, templates, computed mappings)
**Researched:** 2026-04-12
**Confidence:** HIGH (existing codebase verified via direct source reading; patterns validated against LangGraph, Temporal, Anthropic context engineering, and LiteLLM references)

---

## Scope

This document analyzes how 7 architectural gaps integrate with the existing zeroth-core architecture. For each gap, it identifies: new components to create, existing components to modify, data flow changes, and integration points with the governance stack. It concludes with a dependency-aware build order.

---

## System Overview -- Current Architecture

```
                          RunWorker.poll_loop()
                                  |
                     _execute_leased_run(run_id)
                                  |
                       _drive_run(run_id)
                                  |
                    RuntimeOrchestrator._drive(graph, run)
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
   _enforce_loop_guards    _dispatch_node           _plan_next_nodes
   _enforce_policy         (AgentRunner /           (NextStepPlanner /
                            ExecUnitRunner)          BranchResolver)
        |                         |                         |
        |                   _record_history          _queue_next_nodes
        |                   (AuditRepository)        (MappingExecutor)
        |                         |                         |
        +-----------+-------------+-----------+-------------+
                    |                         |
              run.metadata["node_payloads"]   run.pending_node_ids
              run.metadata["last_output"]     run.execution_history
```

### Critical Architectural Constraints

1. **Sequential _drive loop**: `_drive()` pops one `pending_node_ids[0]` at a time, dispatches, plans next nodes, repeats. No parallel dispatch exists.
2. **Payload storage in Run.metadata**: Node payloads (inputs/outputs) are stored as dicts inside `run.metadata["node_payloads"]`. No size limit. Serialized with the Run on every checkpoint.
3. **Mapping operations are data-copy only**: `passthrough`, `rename`, `constant`, `default`. No computation/transformation.
4. **AgentRunner owns prompt assembly**: `PromptAssembler` builds messages from `AgentConfig.instruction` (a raw string). No template indirection.
5. **No external HTTP capability**: Nodes that need HTTP must implement it themselves. No shared client, no retry/circuit-breaker, no capability gating.
6. **Graph model has no subgraph references**: Nodes are leaf types only (agent/executable_unit/human_approval). No nesting.
7. **No context window tracking**: AgentRunner's prompt assembly has no token counting or truncation.

---

## Gap-by-Gap Integration Analysis

### Gap 1: Parallel Fan-Out / Fan-In Execution

**Problem**: `_drive()` is strictly sequential -- `pop(0)` from pending, dispatch, plan next, repeat. Multiple outgoing edges from a single node DO produce multiple entries in `pending_node_ids`, but they execute one at a time.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `ParallelDispatcher` | `src/zeroth/core/orchestrator/parallel.py` | Gather a batch of concurrently-eligible node IDs, dispatch via `asyncio.gather`, collect results |
| `FanOutNode` (model) | `src/zeroth/core/graph/models.py` | New node type `fan_out` with `branch_config: FanOutConfig` defining static/dynamic branches |
| `FanInBarrier` (model) | `src/zeroth/core/graph/models.py` | New node type `fan_in` (or annotation on any node) that waits until all incoming branches complete |
| `BranchContext` | `src/zeroth/core/orchestrator/parallel.py` | Per-branch isolation envelope: own payload namespace, cost accumulator, visit counter |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `RuntimeOrchestrator._drive()` | Detect when multiple nodes in `pending_node_ids` share the same "parallel group", dispatch them via `ParallelDispatcher` instead of one-at-a-time |
| `Run` model | Add `parallel_groups: dict[str, ParallelGroupState]` to track which branches are pending/completed for each fan-in barrier |
| `ExecutionSettings` | Add `max_parallel_branches: int` to bound parallelism |
| `_queue_next_nodes()` | Tag queued nodes with parallel group ID when source node has multiple outgoing edges |
| `AuditRepository` | Fan-out branches need per-branch audit records with a shared `parallel_group_id` for correlation |
| `BudgetEnforcer` | Must track cost per-branch and aggregate at fan-in |

**Data Flow Change**:
```
Before:  A -> B -> C  (sequential pop from pending_node_ids)
After:   A -> [B1, B2, B3] -> FanIn -> C
         B1, B2, B3 dispatched concurrently via asyncio.gather
         FanIn waits for all branches, merges outputs
```

**Key Design Decision -- Superstep Model (recommended)**:
Follow LangGraph's superstep pattern. When `_plan_next_nodes` returns multiple IDs that originated from the same source node's fan-out edges, group them into a "superstep". The superstep is atomic: all branches succeed or none of their state updates apply. This avoids partial-failure complexity.

**Merge Strategy at Fan-In**:
- Default: dict merge with last-write-wins per key
- Configurable: `FanInConfig.merge_strategy` -- `"merge"`, `"list"` (collect as ordered list), `"custom"` (user-provided expression via condition evaluator)

**Governance Integration**:
- PolicyGuard runs per-branch (each branch is an independent policy evaluation)
- BudgetEnforcer checks per-branch spend against a per-branch budget ceiling (configurable, defaults to total budget / branch count)
- AuditRecord includes `parallel_group_id` and `branch_index` fields

---

### Gap 2: Subgraph Composition

**Problem**: Graph model only supports leaf node types. No way to reference a published graph as a nested node.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `SubgraphNode` | `src/zeroth/core/graph/models.py` | New node type `subgraph` with `SubgraphNodeData` containing `graph_ref: str`, `version_constraint`, and `input_mapping`/`output_mapping` |
| `SubgraphRunner` | `src/zeroth/core/orchestrator/subgraph.py` | Resolves graph_ref to a Graph, creates a child Run, drives it via a RuntimeOrchestrator instance, returns output |
| `GraphResolver` | `src/zeroth/core/graph/resolver.py` | Protocol + implementation for looking up published graphs by ref/version. Backed by the same storage as GraphRepository |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `RuntimeOrchestrator._dispatch_node()` | Add `isinstance(node, SubgraphNode)` branch that delegates to `SubgraphRunner` |
| `Graph._validate_references()` | Skip validation for `SubgraphNode` target (it references an external graph, not a node_id in the current graph) |
| `Node` union type | Add `SubgraphNode` to the discriminated union |
| `Run` model | Add `parent_run_id: str | None` for child runs created by subgraph execution |
| `AuditRepository` | Child run audit records reference `parent_run_id` for traceability |

**Thread Continuity**:
- By default, child run inherits `thread_id` from parent run
- `SubgraphNodeData.thread_mode`: `"inherit"` (default), `"isolated"` (new thread), `"shared_read"` (read parent thread, write to child thread)

**Governance Inheritance**:
- Child graph inherits parent's `policy_bindings` UNLESS `SubgraphNodeData.policy_override` is set
- Budget: child run gets a `budget_allocation` carved from parent's remaining budget
- Approval propagation: if child graph hits an approval gate, the parent run transitions to `WAITING_APPROVAL` with metadata linking to the child approval

**Recursion Guard**:
- `ExecutionSettings.max_subgraph_depth: int = 5` -- prevents infinite nesting
- Runtime tracks `subgraph_depth` in run metadata, incremented on each SubgraphRunner call

**Co-design with Gap 1**: A SubgraphNode can appear inside a parallel branch. The `ParallelDispatcher` must handle async subgraph execution naturally since `SubgraphRunner.run()` is async. No special handling needed beyond ensuring the child run's thread isolation prevents cross-branch contamination.

---

### Gap 3: Large Payload Externalization (Artifact Store)

**Problem**: Node payloads are stored inline in `run.metadata["node_payloads"]` as Python dicts, serialized with every checkpoint. Large payloads (images, documents, big JSON) bloat the Run record and slow down checkpoint writes.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `ArtifactStore` (Protocol) | `src/zeroth/core/artifacts/store.py` | `async put(key, data, ttl) -> ArtifactRef`, `async get(ref) -> bytes`, `async delete(ref)` |
| `RedisArtifactStore` | `src/zeroth/core/artifacts/redis.py` | Redis-backed implementation with TTL via `SETEX` |
| `FilesystemArtifactStore` | `src/zeroth/core/artifacts/filesystem.py` | Local filesystem backend for dev/test, TTL via scheduled cleanup |
| `ArtifactRef` | `src/zeroth/core/artifacts/models.py` | Pydantic model: `store_type`, `key`, `content_hash`, `size_bytes`, `created_at`, `ttl_seconds` |
| `ArtifactConfig` | `src/zeroth/core/artifacts/models.py` | Threshold config: `externalize_above_bytes: int = 65536`, `default_ttl_seconds: int = 86400` |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `RuntimeOrchestrator._queue_next_nodes()` | After mapping, check payload size. If above threshold, externalize to ArtifactStore and replace with `ArtifactRef` in `node_payloads` |
| `RuntimeOrchestrator._payload_for()` | When payload value is an `ArtifactRef`, resolve from ArtifactStore before returning |
| `RuntimeOrchestrator._record_history()` | `input_snapshot`/`output_snapshot` in `RunHistoryEntry` store `ArtifactRef` instead of full payload when externalized |
| `RunHistoryEntry` model | Allow `input_snapshot`/`output_snapshot` to contain `ArtifactRef` objects (audit-safe: reference, not payload) |
| `ContractRegistry` | Validation must handle `ArtifactRef` fields transparently (resolve before validation, or validate against ref schema) |
| `NodeAuditRecord` | `input_snapshot`/`output_snapshot` reference externalized artifacts for audit reconstruction |

**Data Flow**:
```
_dispatch_node() produces output_data (potentially large)
    |
    v
_queue_next_nodes():
    1. Apply edge mapping
    2. Check size: if > threshold, artifact_store.put(payload) -> ArtifactRef
    3. Store ArtifactRef in node_payloads[target_node_id]
    |
    v
_payload_for(node_id):
    1. Pop from node_payloads
    2. If ArtifactRef, artifact_store.get(ref) -> bytes -> deserialize
    3. Return materialized dict
```

**TTL and Cleanup**:
- Artifacts get `run_id` and `node_id` metadata for lifecycle management
- On run completion/failure, optionally trigger cleanup of all artifacts for that run
- TTL provides safety net for orphaned artifacts

---

### Gap 4: Agent Context Window Management

**Problem**: `PromptAssembler` builds messages without tracking total token count. For long-running agents with tool loops, the context can overflow the model's context window, causing silent truncation or provider errors.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `TokenCounter` | `src/zeroth/core/agent_runtime/tokens.py` | Wraps `litellm.token_counter()` for model-aware counting. Already a dependency. |
| `ContextWindowManager` | `src/zeroth/core/agent_runtime/context.py` | Monitors token budget, triggers compaction strategy when threshold is exceeded |
| `CompactionStrategy` (Protocol) | `src/zeroth/core/agent_runtime/context.py` | `async compact(messages, budget) -> compacted_messages` |
| `SummarizationCompaction` | `src/zeroth/core/agent_runtime/context.py` | Uses LLM call to summarize older messages, keeps recent N messages verbatim |
| `TruncationCompaction` | `src/zeroth/core/agent_runtime/context.py` | Hard drop of oldest messages beyond budget. No LLM call. Fast and cheap. |
| `SlidingWindowCompaction` | `src/zeroth/core/agent_runtime/context.py` | Keeps system prompt + last N messages + summary of dropped messages |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `AgentConfig` | Add `context_window_config: ContextWindowConfig` with `max_context_tokens: int | None`, `compaction_trigger_ratio: float = 0.85`, `compaction_strategy: str = "truncation"`, `reserve_output_tokens: int = 4096` |
| `AgentRunner.run()` | Before each provider call, check token count via `ContextWindowManager`. If over threshold, apply compaction strategy to message history |
| `PromptAssembler` | Add `count_tokens(messages, model) -> int` method. Uses `TokenCounter` under the hood |
| `AgentAuditSerializer` | Record `context_tokens_before`, `context_tokens_after`, `compaction_applied: bool` in audit data |
| `AgentNodeData` | Surface `context_window_config` as optional field for per-node configuration |

**Token Counting Strategy**:
Use `litellm.token_counter()` because zeroth-core already depends on LiteLLM for provider routing. This gives model-aware tokenization for OpenAI (tiktoken), Anthropic, Cohere, and others without adding new dependencies.

**Compaction Trigger Flow**:
```
AgentRunner.run():
    messages = prompt_assembler.build(input_data, thread_state, ...)
    token_count = context_window_manager.count(messages, model)
    if token_count > max_context_tokens * compaction_trigger_ratio:
        messages = await context_window_manager.compact(messages)
    response = await provider.invoke(request)
```

**Governance Integration**:
- Compaction via `SummarizationCompaction` costs tokens -- route through the existing `InstrumentedProviderAdapter` so cost is tracked via Regulus
- Audit record captures compaction events: how many tokens dropped, strategy used, what was summarized

---

### Gap 5: Resilient External HTTP Client

**Problem**: Agent nodes or executable units that need external HTTP calls must bring their own clients. No shared retry, circuit breaking, caching, connection pooling, or capability gating.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `HttpClient` | `src/zeroth/core/http/client.py` | Async HTTP client wrapping `httpx.AsyncClient` with retry, circuit breaker, and audit hooks |
| `RetryConfig` | `src/zeroth/core/http/models.py` | `max_retries: int`, `backoff_base: float`, `backoff_max: float`, `retry_on_status: list[int]`, `jitter: bool` |
| `CircuitBreakerConfig` | `src/zeroth/core/http/models.py` | `failure_threshold: int`, `reset_timeout_seconds: float`, `half_open_max_requests: int` |
| `CircuitBreaker` | `src/zeroth/core/http/circuit.py` | State machine: CLOSED -> OPEN -> HALF_OPEN. Tracks failure counts per-host |
| `HttpCacheConfig` | `src/zeroth/core/http/models.py` | Optional response caching with TTL for idempotent GET requests |
| `HttpAuditHook` | `src/zeroth/core/http/audit.py` | Records request/response metadata to `AuditRepository` |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `PolicyGuard` | Add `http_request` capability. Nodes must have `"http"` in `capability_bindings` to use `HttpClient` |
| `RuntimeOrchestrator` | Add `http_client: HttpClient | None` field. Injected into agent runners and executable unit runners that need it |
| `ExecutionSettings` | Add `http_config: HttpClientConfig` for graph-level HTTP defaults (timeout, max connections) |
| `AgentConfig` | Optional `http_config` override for per-agent HTTP settings |

**Client Lifecycle**:
- One `httpx.AsyncClient` per `HttpClient` instance, created at bootstrap, shared across all nodes
- Connection pool via `httpx.Limits(max_connections=100, max_keepalive_connections=20)`
- Closed on graceful shutdown alongside `RunWorker`

**Capability Gating**:
```python
# In _dispatch_node, before passing http_client to runner:
if http_client is not None:
    policy_check = await policy_guard.evaluate(
        node, run, capability="http_request"
    )
    if not policy_check.allowed:
        raise PolicyDeniedError("http capability not granted")
```

**Audit Integration**:
Every HTTP request/response is logged as a lightweight audit record:
- `HttpAuditRecord(run_id, node_id, method, url, status_code, latency_ms, retry_count, circuit_state)`
- Sensitive headers (Authorization, cookies) redacted by default

---

### Gap 6: Prompt Template Management

**Problem**: `AgentNodeData.instruction` is a raw string. No support for parameterization, versioning, reuse across nodes, or A/B testing different prompts.

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `PromptTemplate` | `src/zeroth/core/prompts/models.py` | Pydantic model: `template_id`, `version`, `content` (Jinja2 template string), `required_variables: list[str]`, `metadata` |
| `PromptTemplateRegistry` | `src/zeroth/core/prompts/registry.py` | Versioned storage (backed by `AsyncDatabase`) -- register, get, list, resolve by ref |
| `TemplateRenderer` | `src/zeroth/core/prompts/renderer.py` | Jinja2 `SandboxedEnvironment` rendering. Validates required variables are present before rendering |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `AgentNodeData` | Add `instruction_template_ref: str | None` as alternative to raw `instruction`. When set, `instruction` is ignored and template is resolved + rendered at dispatch time |
| `PromptAssembler` | Accept either raw instruction or template ref. When template ref, resolve from `PromptTemplateRegistry`, render with node context variables |
| `RuntimeOrchestrator._dispatch_node()` | If agent node has `instruction_template_ref`, resolve template before passing to `AgentRunner`. Template variables: `{input, output_schema, thread_state, node_id, run_id, ...}` |
| `AgentAuditSerializer` | Record `template_ref`, `template_version`, `rendered_instruction` (optionally redacted) in audit data |

**Template Rendering Pipeline**:
```
_dispatch_node(agent_node, run, input_payload):
    if agent_node.agent.instruction_template_ref:
        template = await template_registry.get(agent_node.agent.instruction_template_ref)
        variables = {
            "input": input_payload,
            "node_id": agent_node.node_id,
            "run_id": run.run_id,
            "thread_id": run.thread_id,
            ...
        }
        instruction = template_renderer.render(template, variables)
        # Temporarily override instruction for this execution
        agent_node_copy = agent_node.model_copy(...)
        agent_node_copy.agent.instruction = instruction
```

**Security**: Use Jinja2 `SandboxedEnvironment` to prevent template injection. No `eval`, `exec`, or filesystem access from templates. This matches the pattern used by MLflow's prompt registry and Microsoft Semantic Kernel.

**Versioning**: Follows the same pattern as `ContractRegistry` -- name + version, auto-increment, latest resolution. Templates can be registered, listed, and pinned to specific versions per-node.

---

### Gap 7: Computed Data Mappings (Transform Operation)

**Problem**: `MappingExecutor` supports `passthrough`, `rename`, `constant`, and `default` operations. No way to compute derived values (e.g., concatenate strings, extract subfields, do arithmetic).

**New Components**:

| Component | Location | Responsibility |
|-----------|----------|----------------|
| `TransformMappingOperation` | `src/zeroth/core/mappings/models.py` | New operation type `transform` with `expression: str` and `source_paths: list[str]` |

**Existing Components Modified**:

| Component | Change |
|-----------|--------|
| `MappingOperation` union | Add `TransformMappingOperation` to the discriminated union |
| `MappingExecutor._apply_operation()` | Add `case TransformMappingOperation()` that resolves source paths, builds namespace, evaluates expression via `_SafeEvaluator` |
| `MappingValidator` | Add validation for `transform` operations (expression is parseable, source paths are valid) |

**Implementation -- Reuse `_SafeEvaluator`**:
The existing `ConditionEvaluator._SafeEvaluator` already supports all needed operations: arithmetic, string concatenation, comparisons, list/dict construction, ternary expressions. The `transform` operation reuses this engine:

```python
case TransformMappingOperation():
    namespace = {}
    for source_path in operation.source_paths:
        exists, value = _get_path(payload, source_path)
        if exists:
            # Flatten dotted paths to simple names for expression access
            simple_name = source_path.replace(".", "_")
            namespace[simple_name] = value
    evaluator = _SafeEvaluator(namespace)
    result = evaluator.evaluate(operation.expression)
    _set_path(output, operation.target_path, result)
```

**Example Usage**:
```json
{
    "operation": "transform",
    "source_paths": ["user.first_name", "user.last_name"],
    "expression": "user_first_name + ' ' + user_last_name",
    "target_path": "full_name"
}
```

**Side-Effect Safety**: The `_SafeEvaluator` is already constrained to pure expressions (no function calls, no imports, no assignments). Transform mappings inherit this safety by construction.

---

## New Module Layout

```
src/zeroth/core/
    artifacts/              # Gap 3: NEW
        __init__.py
        models.py           # ArtifactRef, ArtifactConfig
        store.py            # ArtifactStore protocol
        redis.py            # RedisArtifactStore
        filesystem.py       # FilesystemArtifactStore
    http/                   # Gap 5: NEW
        __init__.py
        models.py           # RetryConfig, CircuitBreakerConfig, HttpClientConfig
        client.py           # HttpClient (wraps httpx.AsyncClient)
        circuit.py          # CircuitBreaker state machine
        audit.py            # HttpAuditHook
    prompts/                # Gap 6: NEW
        __init__.py
        models.py           # PromptTemplate model
        registry.py         # PromptTemplateRegistry (async DB-backed)
        renderer.py         # TemplateRenderer (Jinja2 sandboxed)
    orchestrator/
        runtime.py          # MODIFIED: parallel dispatch, subgraph dispatch, artifact resolution
        parallel.py         # Gap 1: NEW -- ParallelDispatcher, BranchContext
        subgraph.py         # Gap 2: NEW -- SubgraphRunner, GraphResolver protocol
    graph/
        models.py           # MODIFIED: FanOutNode, FanInNode, SubgraphNode additions
        resolver.py         # Gap 2: NEW -- GraphResolver implementation
    mappings/
        models.py           # MODIFIED: TransformMappingOperation added to union
        executor.py         # MODIFIED: transform case in _apply_operation
    agent_runtime/
        runner.py           # MODIFIED: context window check before provider calls
        context.py          # Gap 4: NEW -- ContextWindowManager, compaction strategies
        tokens.py           # Gap 4: NEW -- TokenCounter wrapping litellm
        models.py           # MODIFIED: ContextWindowConfig in AgentConfig
        prompt.py           # MODIFIED: token counting integration
    runs/
        models.py           # MODIFIED: parallel_groups, parent_run_id fields
    conditions/
        evaluator.py        # UNCHANGED (reused by Gap 7 transform mappings)
```

---

## Architectural Patterns

### Pattern 1: Protocol + Pluggable Backend

**What**: Define a Protocol (abstract interface) for each new subsystem, with concrete implementations swappable at bootstrap time.
**Used by**: ArtifactStore (Redis/Filesystem), CompactionStrategy (Summarize/Truncate/SlidingWindow), GraphResolver
**Why**: Zeroth already uses this pattern for `ProviderAdapter`, `SecretProvider`, `ThreadStateStore`. Consistency and testability.

```python
class ArtifactStore(Protocol):
    async def put(self, key: str, data: bytes, *, ttl: int | None = None) -> ArtifactRef: ...
    async def get(self, ref: ArtifactRef) -> bytes: ...
    async def delete(self, ref: ArtifactRef) -> None: ...
```

### Pattern 2: Optional Injection via RuntimeOrchestrator Fields

**What**: New capabilities are optional dataclass fields on `RuntimeOrchestrator`, defaulting to `None`. When `None`, the feature is transparently disabled.
**Used by**: All 7 gaps (artifact_store, http_client, template_registry, context_window_manager, etc.)
**Why**: Backward compatibility. Existing users who construct a `RuntimeOrchestrator` without new fields get identical behavior to today. This is the pattern already used for `regulus_client`, `cost_estimator`, `memory_resolver`, `budget_enforcer`.

```python
@dataclass(slots=True)
class RuntimeOrchestrator:
    # ... existing fields ...
    artifact_store: ArtifactStore | None = None
    http_client: HttpClient | None = None
    template_registry: PromptTemplateRegistry | None = None
    subgraph_runner: SubgraphRunner | None = None
```

### Pattern 3: Governance-by-Default for New Capabilities

**What**: Every new feature that touches external resources (HTTP calls, artifact storage, subgraph execution) goes through the existing governance stack: PolicyGuard for capability checks, AuditRepository for audit trails, BudgetEnforcer for cost.
**Why**: Core value proposition of zeroth is governed execution. New features without governance undermine the platform's reason for existing.

### Pattern 4: Atomic Superstep for Parallelism

**What**: Group concurrently-executable nodes into a superstep. All branches in a superstep must complete before any state updates are applied. If any branch fails, none are applied.
**Why**: Simplifies failure recovery (no partial state), matches LangGraph's proven model, and integrates cleanly with checkpoint/resume.

---

## Data Flow Changes

### Before (v3.0 Sequential)
```
pending_node_ids: [B, C, D]
_drive loop:
    pop B -> dispatch -> plan_next -> queue [E] -> checkpoint
    pop C -> dispatch -> plan_next -> queue [F] -> checkpoint
    pop D -> dispatch -> plan_next -> queue [] -> checkpoint
    pop E -> ...
```

### After (v4.0 with Parallel + Subgraph + Artifacts)
```
pending_node_ids: [B1, B2, B3]  (parallel group "pg-1")
_drive loop:
    detect parallel group "pg-1" for [B1, B2, B3]
    parallel_dispatcher.dispatch([B1, B2, B3]):
        for each branch:
            resolve payload (possibly from ArtifactStore)
            if SubgraphNode: subgraph_runner.run(child_graph, payload)
            else: _dispatch_node(node, run, payload)
            if payload > threshold: externalize output to ArtifactStore
        wait all branches (asyncio.gather)
    merge results at fan-in barrier
    plan_next -> queue [C] -> checkpoint
    pop C -> ...
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Parallel Dispatch via Separate Worker Processes

**What people might do**: Use RunWorker's existing multi-run concurrency (semaphore-bounded) to dispatch parallel branches as separate runs.
**Why it's wrong**: Branches are NOT independent runs. They share a parent run's state, governance context, and budget. Treating them as separate runs breaks audit correlation, budget accounting, and fan-in synchronization.
**Do this instead**: In-process `asyncio.gather` within a single `_drive()` call. The `RunWorker` semaphore bounds run-level concurrency; within a run, `ParallelDispatcher` bounds branch-level concurrency.

### Anti-Pattern 2: Storing Large Artifacts in Redis Without TTL

**What people might do**: Use RedisArtifactStore with no TTL, letting artifacts accumulate.
**Why it's wrong**: Redis is memory-bound. Orphaned artifacts from failed runs will consume RAM indefinitely.
**Do this instead**: Always set TTL. Default 24h. Clean up on run completion. Filesystem backend for truly large artifacts (> 10MB).

### Anti-Pattern 3: Recursive Subgraph Without Depth Limit

**What people might do**: Allow subgraphs to reference graphs that themselves contain subgraph nodes, with no depth limit.
**Why it's wrong**: Infinite recursion. Stack overflow. Resource exhaustion.
**Do this instead**: `max_subgraph_depth` in ExecutionSettings (default 5). Runtime tracks depth and fails fast when exceeded.

### Anti-Pattern 4: Compaction Using the Same Provider as the Agent

**What people might do**: Summarization compaction calls the same expensive model (GPT-4, Claude Opus) as the agent itself.
**Why it's wrong**: Compaction should be cheap. Using an expensive model for summarization doubles the cost without proportional benefit.
**Do this instead**: `CompactionConfig.model: str` defaults to a fast, cheap model (e.g., `gpt-4o-mini`, `claude-3-haiku`). Configurable per-agent.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Redis | `RedisArtifactStore` uses existing `RedisConfig` from `storage.redis` | Reuse connection pool. Separate key prefix (`zeroth:artifacts:`) |
| LiteLLM | `TokenCounter` uses `litellm.token_counter()` | Already a dependency. No new deps needed |
| Jinja2 | `TemplateRenderer` uses `jinja2.sandbox.SandboxedEnvironment` | New dependency. Lightweight, well-maintained |
| httpx | `HttpClient` wraps `httpx.AsyncClient` | Already an indirect dependency via LiteLLM. Pin explicitly |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Orchestrator <-> ParallelDispatcher | Direct method call | ParallelDispatcher is owned by orchestrator, not a separate service |
| Orchestrator <-> SubgraphRunner | Direct method call | SubgraphRunner creates child runs via the same RunRepository |
| Orchestrator <-> ArtifactStore | Protocol-based injection | Transparently disabled when None |
| AgentRunner <-> ContextWindowManager | Direct method call, pre-provider-call hook | Manager is a field on AgentRunner, not orchestrator |
| MappingExecutor <-> _SafeEvaluator | Direct import from conditions.evaluator | Reuses existing safe expression engine |
| PromptTemplateRegistry <-> AsyncDatabase | Same pattern as ContractRegistry | Shares DB connection, separate table |
| HttpClient <-> PolicyGuard | Orchestrator checks capability before passing client | Client itself is policy-unaware |

---

## Scalability Considerations

| Concern | At 100 runs/hour | At 10K runs/hour | At 1M runs/hour |
|---------|-------------------|-------------------|-------------------|
| Parallel branches | asyncio.gather within process, fine | Still fine -- branches are I/O bound (LLM calls), not CPU bound | Consider branch-level worker distribution (future) |
| Artifact storage | Filesystem or Redis both adequate | Redis with TTL. Monitor memory. Consider S3 backend | S3/GCS mandatory. Redis for metadata only |
| Context compaction | Rare -- most runs under context limit | Common for long tool loops. LLM compaction cost is amortized | Pre-compute compaction in background, cache summaries |
| Subgraph depth | Minimal overhead | Depth 2-3 typical. 5 max adequate | Flatten deeply nested graphs during publish |
| HTTP client pool | Single shared client, plenty | Connection pool at 100, increase if needed | Multiple client instances sharded by target host |

---

## Suggested Build Order

Dependencies between gaps determine the optimal sequencing:

```
Phase A:  Gap 7 (Transform Mappings)    -- zero dependencies, smallest scope
          Gap 3 (Artifact Store)         -- zero dependencies, foundational for Gap 1

Phase B:  Gap 4 (Context Window)         -- depends on nothing, but benefits from Gap 3 for large context summaries
          Gap 5 (HTTP Client)            -- depends on nothing
          Gap 6 (Prompt Templates)       -- depends on nothing

Phase C:  Gap 1 (Parallel Fan-Out/In)    -- benefits from Gap 3 (externalized branch payloads)
          Gap 2 (Subgraph Composition)   -- benefits from Gap 1 (subgraphs inside parallel branches)

Integration: Wire all gaps into service bootstrap, update OpenAPI spec, update docs
```

**Rationale for this ordering**:

1. **Gap 7 first** because it is the smallest, most self-contained change (one new model + one case in executor). Ships fast, builds confidence, and exercises the PR/test pipeline.

2. **Gap 3 early** because the artifact store is a prerequisite for Gap 1 (parallel branches produce large merged payloads) and useful for Gap 4 (externalized context summaries). It also prevents a known pain point -- large payload bloat in Run checkpoints.

3. **Gaps 4, 5, 6 are independent** and can be built in parallel by different developers. They each touch different subsystems (agent_runtime, new http module, new prompts module) with no overlap.

4. **Gap 1 and Gap 2 last** because they are the most complex, touch the core `_drive()` loop, and benefit from all earlier gaps being in place (artifact store for payloads, templates for subgraph nodes, HTTP client for external API nodes within parallel branches).

5. **Gap 1 before Gap 2** (or co-developed) because subgraph nodes will commonly appear inside parallel branches, and the parallel dispatch infrastructure must be stable before adding another dispatch variant.

---

## Co-Design Requirements: Gap 1 + Gap 2

These two gaps share critical integration surfaces and should be designed together even if built sequentially:

1. **ParallelDispatcher must handle SubgraphNode**: When a parallel branch contains a subgraph node, the dispatcher must create a child run, drive it, and collect its output -- all within the branch's `asyncio.gather` task.

2. **Budget carving**: Both parallel branches and subgraph execution need to carve budget from the parent. The budget model must support: parent -> N branches, and within each branch, parent -> child subgraph. This is a tree, not a flat split.

3. **Audit correlation**: `parallel_group_id` (from Gap 1) and `parent_run_id` (from Gap 2) must coexist in audit records. A node execution inside a subgraph inside a parallel branch has: `run_id`, `parent_run_id`, `parallel_group_id`, and `branch_index`.

4. **Thread isolation**: Parallel branches need isolated payloads (already via per-branch `BranchContext`). Subgraph nodes within branches need thread isolation (via `SubgraphNodeData.thread_mode`). These must compose correctly: a `"shared_read"` subgraph inside a parallel branch reads the parent branch's thread, not the other branches' threads.

---

## Sources

- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [LangGraph: Branching and parallel execution](https://www.baihezi.com/mirrors/langgraph/how-tos/branching/index.html)
- [LangGraph: Subgraph composition](https://latenode.com/blog/ai-frameworks-technical-infrastructure/langgraph-multi-agent-orchestration/langgraph-ai-framework-2025-complete-architecture-guide-multi-agent-orchestration-analysis)
- [LiteLLM: Token counting](https://docs.litellm.ai/docs/count_tokens)
- [LiteLLM: Completion token usage and cost](https://docs.litellm.ai/docs/completion/token_usage)
- [httpx: Async support](https://www.python-httpx.org/async/)
- [httpx: Resource limits](https://www.python-httpx.org/advanced/resource-limits/)
- [Jinja2 prompt templating (Instructor)](https://python.useinstructor.com/concepts/templating/)
- [MLflow Prompt Registry](https://mlflow.org/docs/latest/genai/prompt-registry/)
- [Microsoft Agent Framework: Compaction](https://learn.microsoft.com/en-us/agent-framework/agents/conversations/compaction)
- [Dapr: Workflow patterns including fan-out/fan-in](https://docs.dapr.io/developing-applications/building-blocks/workflow/workflow-patterns/)
- [Scaling LangGraph Agents: Parallelization, Subgraphs, and Map-Reduce](https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization)

---
*Architecture research for: zeroth-core v4.0 platform extensions*
*Researched: 2026-04-12*
