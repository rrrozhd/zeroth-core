# Feature Landscape

**Domain:** Platform extensions for production agentic workflow orchestration
**Researched:** 2026-04-12
**Overall confidence:** HIGH (cross-platform evidence from LangGraph, Temporal, Prefect, Airflow, Conductor, Semantic Kernel, n8n)

---

## Table Stakes

Features users expect from a production-grade agent orchestration platform. Missing = blocks real-world adoption (especially for teams migrating from LangGraph or similar).

| Feature | Why Expected | Complexity | Depends On (existing zeroth-core) | Notes |
|---------|-------------|------------|----------------------------------|-------|
| **Parallel fan-out/fan-in** | Every workflow platform supports this. LangGraph Send(), Temporal activities, Prefect map(), Airflow dynamic task mapping all provide it. Users cannot model real workloads (batch processing, multi-source retrieval, parallel agent evaluation) without it. | **HIGH** | RuntimeOrchestrator (sequential step loop must become parallel-aware), RunRepository (branch state tracking), BudgetEnforcer (per-branch cost isolation), AuditRepository (per-branch audit trails), guardrails (per-branch visit counting) | The single hardest feature. Requires fundamental changes to the orchestrator's step execution loop. |
| **Computed data mappings** | n8n expressions, Azure Data Factory derived columns, Airflow XCom transforms all provide this. Current zeroth mappings (passthrough/rename/constant/default) cannot derive new values from existing data -- a basic workflow need. | **LOW** | MappingExecutor, MappingOperation union type, _SafeEvaluator (conditions/evaluator.py -- reuse directly), EdgeMapping model | Lowest risk of the 7. The expression engine already exists in conditions/evaluator.py and supports arithmetic, comparisons, string ops, ternary. Just wire it as a new MappingOperation variant. |
| **Resilient HTTP client** | Agent nodes routinely call external APIs. Without managed resilience (retry, backoff, circuit breaking), any external dependency failure cascades into workflow failure. Every production system needs this. Temporal has it built into activities. Prefect uses httpx with retries. | **MEDIUM** | SecretResolver (API key injection), AuditRepository (HTTP call logging), PolicyGuard (capability-gated access), config/settings.py (circuit breaker thresholds) | Well-understood patterns. httpx + tenacity is the standard Python stack. The novel part is integrating with zeroth's governance: capability checks before HTTP calls, audit logging of requests/responses, secret injection for auth headers. |
| **Large payload externalization** | Temporal has a 2MB event history limit and uses Payload Codecs for offloading. Conductor enforces soft barriers at 3-5MB. Argo Workflows externalizes all artifacts to S3/GCS. Without this, graphs passing large LLM outputs, documents, or embeddings between nodes will hit memory/storage limits. | **MEDIUM** | RunRepository (replace inline payload storage with references), MappingExecutor (dereference artifact refs transparently), contracts/registry (artifact-aware validation), AuditRepository (store references not payloads) | The pattern is well-established: store payload in blob store, pass reference through the graph. Key design decision is transparency -- callers should not know whether a payload is inline or externalized. |

---

## Differentiators

Features that set zeroth apart. Not expected by every user, but highly valued by teams building production agent systems.

| Feature | Value Proposition | Complexity | Depends On (existing zeroth-core) | Notes |
|---------|-------------------|------------|----------------------------------|-------|
| **Subgraph composition** | LangGraph subgraphs and Temporal child workflows support this, but most platforms implement it as a second-class citizen (Airflow deprecated SubDagOperator in favor of TaskGroups which are UI-only). Zeroth can differentiate by making subgraphs first-class: governance inheritance, thread continuity, approval propagation, and budget isolation across nested graphs. | **HIGH** | Graph model (needs SubgraphNode type), GraphRepository (version-resolved subgraph loading), RuntimeOrchestrator (recursive execution or flattening), PolicyGuard (governance inheritance), BudgetEnforcer (nested budget scoping), ApprovalService (approval propagation), thread state (continuity across parent/child) | Second hardest feature. LangGraph's approach (shared state keys vs isolated state) maps well to zeroth's mapping system, but governance inheritance is novel and zeroth-specific. |
| **Agent context window management** | Most frameworks punt on this entirely -- LangChain has ConversationSummaryBufferMemory but it is basic. Semantic Kernel added ChatHistoryReducer in v1.35.0 (2025). Anthropic's context engineering guide emphasizes this as critical. Zeroth already tracks tokens via InstrumentedProviderAdapter; adding managed context windows with pluggable strategies (truncation, summarization, observation masking) would be a strong differentiator. | **MEDIUM** | AgentRunner (token tracking per call), PromptAssembler (context budget enforcement), InstrumentedProviderAdapter (token usage data), thread state store (conversation history), MemoryConnectorResolver (external memory for summaries) | JetBrains research (Dec 2025) found observation masking outperforms LLM summarization while being 52% cheaper. Implement both strategies but default to the simpler one. |
| **Prompt template management** | Dedicated prompt versioning platforms (PromptLayer, Langfuse, Maxim AI) exist but are external services. Embedding versioned prompt templates directly in the orchestrator -- with variable rendering, audit redaction, and agent node integration -- gives teams a self-contained solution. No production agent framework has this built-in well. | **LOW-MEDIUM** | AgentNodeData.instruction (currently a raw string -- becomes template_ref), contracts/registry (pattern for versioned registry), AuditRepository (template version tracking), SecretResolver (pattern for variable resolution), PromptAssembler (render templates before assembly) | The registry pattern already exists in contracts/registry.py. Prompt templates are simpler than contracts (just strings with variables). The novel value is audit-aware rendering that redacts sensitive variables. |

---

## Anti-Features

Features to explicitly NOT build. Based on platform comparison and zeroth's architectural constraints.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **Distributed parallel execution across workers** | Temporal and Prefect distribute parallel branches across worker pools. This requires distributed state coordination, message queuing, and complex failure handling. Zeroth's modular monolith architecture (single-process RuntimeOrchestrator with durable dispatch) is not designed for this. | In-process asyncio.gather() for parallel branches within a single worker. The existing RunWorker already has max_concurrency=8 via semaphore. Parallel branches execute as concurrent tasks within the same worker process. Scale by running more workers, not distributing branches. |
| **Full expression language / DSL for computed mappings** | n8n uses JavaScript expressions. Azure Data Factory has a full expression language with 100+ functions. Building a rich DSL creates maintenance burden, security surface, and learning curve. | Reuse the existing _SafeEvaluator which supports arithmetic, comparisons, boolean logic, string concatenation, ternary expressions, list/dict construction. This is sufficient for data mappings. Add a small set of pure built-in functions (len, str, int, float, upper, lower, keys, values) if needed. |
| **External prompt management platform integration** | PromptLayer, Langfuse, and Maxim AI are dedicated prompt platforms. Building adapters for each creates coupling and maintenance burden. | Build a self-contained PromptTemplateRegistry. If users want to sync from external platforms, they can write a loader that populates the registry. The registry API is the integration point, not per-platform adapters. |
| **Automatic context summarization as default behavior** | LLM-based summarization adds latency (~13-15% longer runs per JetBrains research), costs tokens, and can obscure stopping signals. It is not always the right strategy. | Make context management pluggable with strategies. Default to observation masking (cheaper, faster, often better). Offer summarization as opt-in. Let users configure trigger thresholds and strategy per agent node. |
| **Artifact store as a general-purpose object store** | Conductor's external payload storage is specifically for workflow payloads, not arbitrary files. Building a general-purpose object store adds scope. | Artifact store handles only workflow intermediate data: node outputs that exceed a size threshold. It is not a file upload service. TTL-based cleanup prevents unbounded growth. |
| **Subgraph runtime flattening** | Airflow's SubDagOperator flattened subDAGs into the parent, causing scheduling issues (deprecated in Airflow 2.0+). LangGraph compiles subgraphs but preserves isolation. | Execute subgraphs as recursive RuntimeOrchestrator invocations with their own Run, preserving isolation. Parent sees subgraph as a single node with input/output contracts. This avoids the scheduling complexity that killed Airflow's SubDagOperator. |
| **HTTP response caching in the resilient client** | Caching external API responses requires cache invalidation logic, storage management, and correctness guarantees that are out of scope. | The resilient HTTP client handles retry/backoff/circuit-breaking only. If users need caching, they can wrap calls with their own cache layer or use httpx-cache. Caching is a separate concern from resilience. |

---

## Detailed Feature Analysis

### 1. Parallel Fan-Out / Fan-In

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **LangGraph** | `Send()` API creates dynamic parallel tasks at runtime. State reducers (e.g., `operator.add`) merge results. Each parallel branch gets independent worker state. | Runtime-determined parallelism. Reducer functions define merge semantics. |
| **Temporal** | `workflow.ExecuteActivity()` returns futures. `Promise.all()` waits for all. Child Workflows partition large fan-outs (1000 children x 1000 activities = 1M activities). | Futures-based. Event history limits require child workflow partitioning for large fan-outs. |
| **Prefect** | `task.map()` over iterables. Dynamic children are first-class tasks (can succeed, fail, retry, pause independently). DaskExecutor for distributed parallelism. | First-class mapped tasks. Each branch has full lifecycle. |
| **Airflow** | `expand()` for dynamic task mapping (since 2.3). Creates mapped task instances at runtime. Concurrency controlled by `max_active_tasks`. | Runtime-determined. XCom for result passing. Recent issues with dynamically mapped TaskGroups. |

**Zeroth-specific design considerations:**
- Per-branch audit trails (unique to zeroth's governance model)
- Per-branch budget tracking via BudgetEnforcer (no platform does this)
- Per-branch policy enforcement via PolicyGuard
- Result aggregation via computed mappings (new transform operation)
- Deterministic ordering of results regardless of completion order
- Guardrail enforcement: max_total_steps and max_visits_per_node apply across all branches

**Recommended approach:** LangGraph's Send() pattern adapted for zeroth. A fan-out edge specifies an expression that produces a list; the orchestrator spawns one branch per list element via asyncio.gather(). A fan-in barrier node collects results using a computed mapping with a reduce expression. Each branch gets its own audit trail and budget scope.

---

### 2. Subgraph Composition

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **LangGraph** | Compiled subgraphs as nodes. Two communication patterns: shared state keys (automatic) or isolated state with explicit transformation. Subgraphs inherit parent checkpointer. | Shared vs isolated state is the key design decision. |
| **Temporal** | Child Workflows with full isolation (own event history). Parent cancellation propagates. Child can outlive parent (ParentClosePolicy). | Full isolation by default. Explicit parent-child lifecycle control. |
| **Prefect** | Subflows share parent flow run context. Parent settings inherited. Can be invoked synchronously or asynchronously. | Shared context, simple invocation model. |
| **Airflow** | SubDagOperator (deprecated) -- flattened subDAGs caused scheduling issues. TaskGroup -- UI-only grouping, no runtime isolation. | Airflow tried and failed with runtime subgraph flattening. TaskGroup is display-only. |

**Zeroth-specific design considerations:**
- SubgraphNode references a published Graph by graph_id + version
- Input/output contracts enforced at the subgraph boundary (via EdgeMapping)
- Governance inheritance: parent's policy_bindings propagate to child unless overridden
- Budget scoping: child gets a sub-budget from parent's remaining budget
- Approval propagation: child's HumanApprovalNodes are visible to parent's approval service
- Thread continuity: child can read parent's thread state (configurable)
- The child creates its own Run (Temporal's isolation model, not Airflow's flattening)

**Recommended approach:** Temporal's child workflow isolation model. SubgraphNode is a new node type. The orchestrator recursively invokes itself for the child graph, creating a child Run. The child Run's input comes from the parent's edge mapping; the child's output is returned to the parent via the subgraph node's output contract. Policy and capability bindings are inherited by default (can be overridden). Budget is scoped (child cannot exceed parent's remaining allocation).

---

### 3. Large Payload Externalization

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **Conductor** | ExternalPayloadStorage with soft barriers (5MB) and hard barriers (10MB). Three backends: S3, Azure Blob, PostgreSQL. Fully transparent to workers. | Two-tier threshold (externalize vs reject). Transparency is critical. |
| **Temporal** | Payload Codec (DataDog large-payload-codec). Codecs transform bytes-to-bytes in the serialization pipeline. Applied after Payload Converter. Transparent to workflow code. | Codec layer in the serialization pipeline. Third-party solution, not built-in. |
| **Argo Workflows** | Artifact repository (S3/GCS/Azure/HTTP). All outputs are artifacts by default. Explicit artifact declarations in workflow spec. | Artifacts are first-class, not transparent. Requires explicit declaration. |

**Zeroth-specific design considerations:**
- Pluggable backends: Redis (default for dev), filesystem (default for tests), S3-compatible (production)
- Configurable threshold (default: 1MB soft barrier, 10MB hard barrier)
- ArtifactRef replaces inline payload in Run state -- transparent to MappingExecutor
- TTL-based cleanup with configurable retention
- Audit references: audit trail records ArtifactRef (not the full payload) with hash for integrity verification
- Contract compatibility: validation happens on the original data before externalization

**Recommended approach:** Conductor's transparent externalization model. An ArtifactStore protocol with pluggable backends. The RunRepository transparently externalizes payloads exceeding the soft threshold. MappingExecutor transparently dereferences ArtifactRef values. Hard threshold rejects payloads entirely. TTL cleanup prevents unbounded growth.

---

### 4. Agent Context Window Management

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **Semantic Kernel** | ChatHistoryReducer (v1.35.0+). Two built-in reducers: truncation and summarization. Configurable target_count, threshold_count, auto_reduce. System messages always preserved. | Pluggable strategy with sensible defaults. System message preservation is critical. |
| **LangChain** | ConversationSummaryBufferMemory. Fixed window + summary of older messages. Token counting via tiktoken. | Basic but functional. Single strategy, not pluggable. |
| **Anthropic** | Context engineering principles: compaction, structured note-taking, sub-agent architectures, just-in-time retrieval. Emphasizes "smallest set of high-signal tokens." | Strategic guidance, not library code. Key insight: less context is often better. |
| **JetBrains Research** | Compared observation masking vs LLM summarization. Observation masking achieved 2.6% higher solve rates while being 52% cheaper. Summarization extended run times by 13-15%. | Simpler approach (masking) outperforms sophisticated approach (summarization). |

**Zeroth-specific design considerations:**
- Token tracking already exists via InstrumentedProviderAdapter
- PromptAssembler currently does not enforce token budgets
- Strategies: truncation (drop oldest), observation masking (replace old outputs with placeholders), summarization (LLM-based compression)
- Per-agent-node configuration: each AgentNode can have its own context strategy and thresholds
- System messages and tool definitions always preserved (Semantic Kernel pattern)
- Integration with thread state: reducer operates on thread history before prompt assembly

**Recommended approach:** Semantic Kernel's ChatHistoryReducer pattern adapted for zeroth. A ContextWindowManager with pluggable strategies. Default to observation masking (cheapest, fastest, empirically best per JetBrains research). Offer truncation as simplest fallback and summarization as opt-in. Configure per AgentNode via execution_config. Token budget enforced in PromptAssembler before LLM call.

---

### 5. Resilient External HTTP Client

**How production systems do it:**

| Pattern | Implementation | Key Insight |
|---------|---------------|-------------|
| **Retry with exponential backoff + jitter** | tenacity library (Python standard). Configurable retry conditions, wait strategies, stop conditions. | Jitter prevents synchronized retry storms across workers. |
| **Circuit breaker** | Three states: closed (normal), open (failing, reject immediately), half-open (testing recovery). Opens after N consecutive failures. | Prevents hammering a down service. Critical for agent nodes that call the same API repeatedly. |
| **Connection pooling** | httpx.AsyncClient with connection limits. Reuse connections across requests within a worker. | Reduces TCP/TLS handshake overhead for repeated calls to same host. |
| **Timeout layering** | Connect timeout (short), read timeout (medium), total timeout (long). httpx supports all three. | Different timeout types for different failure modes. |

**Zeroth-specific design considerations:**
- Capability-gated: agent must have `http:call` capability to use the client
- Audit-logged: every external HTTP call recorded in audit trail (URL, method, status, duration)
- Secret-injected: auth headers resolved via SecretResolver before the call
- Redaction-aware: request/response bodies redacted in audit per existing AuditSerializer patterns
- Circuit breaker state per (host, port) pair, shared across nodes in a run
- Budget-aware: HTTP calls with token cost (e.g., calling paid APIs) can debit budget

**Recommended approach:** Wrap httpx.AsyncClient with tenacity for retry/backoff and a simple circuit breaker. The client is a service-level singleton (one per worker process) with per-host circuit breaker state. Exposed to agent nodes and executable units via a ResilientHTTPClient that takes a capability check, secret resolution, and audit logging as constructor dependencies. Not a general-purpose HTTP library -- a governed, audited HTTP client.

---

### 6. Prompt Template Management

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **PromptLayer** | Versioned prompt registry with A/B testing, analytics, and deployment environments. | External SaaS. Overkill for embedded use. |
| **Langfuse** | Open-source prompt management with versioning, environment labels, and observability integration. | Good model but requires separate deployment. |
| **LangChain** | PromptTemplate with variable substitution. Hub for sharing. No built-in versioning. | Template rendering without lifecycle management. |
| **Maxim AI** | Full prompt lifecycle: authoring, versioning, evaluation, deployment with environment promotion. | Enterprise-grade. Indicates the direction the market is heading. |

**Zeroth-specific design considerations:**
- PromptTemplateRegistry follows the exact pattern of contracts/registry.py (versioned, named, Pydantic models)
- Templates are strings with `{variable}` placeholders (Jinja2 is overkill; Python str.format_map is sufficient)
- AgentNodeData.instruction becomes a template_ref pointing to a registered template
- Rendering happens in PromptAssembler before LLM call
- Audit trail records template_ref + version + rendered variables (with redaction)
- Templates are immutable once published (same lifecycle as Graph: draft -> published -> archived)

**Recommended approach:** A PromptTemplateRegistry mirroring contracts/registry.py. Templates are Pydantic models with name, version, body (template string), variables (list of expected variable names), and lifecycle status. Rendering uses Python's str.format_map with variable values resolved from the node's input payload and runtime context. Audit records template version used and variables provided (redacted per existing patterns). No Jinja2 dependency -- keep it simple.

---

### 7. Computed Data Mappings

**How platforms do it:**

| Platform | Mechanism | Key Insight |
|----------|-----------|-------------|
| **n8n** | Expression engine using Tournament templating + JavaScript. Inline expressions in `{{ }}` brackets. Data transformation functions for common operations. | Inline expressions in node parameters. Immediate preview. |
| **Azure Data Factory** | Mapping Data Flow with Expression Builder. 100+ built-in functions. Local variables for intermediate computations. Derived column transformation. | Rich expression language with dedicated IDE. |
| **Airflow** | XCom for inter-task data passing. Jinja2 templates for dynamic values. TaskFlow API with explicit return values. | Template-based with explicit data passing. |
| **Salesforce Flow** | Transform element for data mapping. Field-to-field mapping with formula support. | UI-driven mapping with formula expressions. |

**Zeroth-specific design considerations:**
- Add a `transform` operation to the existing MappingOperation discriminated union
- TransformMappingOperation has: target_path (where to write), expression (string), source_paths (list of input paths to bind as variables)
- The expression is evaluated by the existing _SafeEvaluator from conditions/evaluator.py
- Side-effect-free: the evaluator only supports pure expressions (no function calls, no imports, no assignments)
- The existing evaluator already supports: arithmetic (+, -, *, /, %), comparisons, boolean logic, ternary (if/else), list/dict/set literals, subscript access, attribute access
- May need to add a small set of pure built-in functions to the evaluator's namespace (len, str, int, float, upper, lower, keys, values, items, sorted, min, max, sum)

**Recommended approach:** Add TransformMappingOperation to the MappingOperation union. The executor resolves source_paths from the input payload, builds a namespace dict, and passes the expression to _SafeEvaluator. Example: `target_path: "full_name"`, `expression: "first + ' ' + last"`, `source_paths: {"first": "user.first_name", "last": "user.last_name"}`. This reuses 100% of the existing expression engine with zero new parsing code.

---

## Feature Dependencies

```
                    Computed Mappings
                          |
                          v
Parallel Fan-Out -----> [needs: computed mappings for fan-in aggregation]
       |
       v
Subgraph Composition --> [needs: parallel fan-out for parallel subgraph invocation]
       |
       v
Large Payload Store ---> [benefits from being in place before parallel/subgraph
                          since those multiply intermediate data]

Agent Context Mgmt ----> [independent; token tracking already exists]

Resilient HTTP ---------> [independent; can be built in any order]

Prompt Templates -------> [independent; can be built in any order]
```

**Dependency chain (must-respect ordering):**
1. Computed Mappings (no deps, enables aggregation expressions for fan-in)
2. Parallel Fan-Out/Fan-In (uses computed mappings for result aggregation)
3. Subgraph Composition (builds on parallel execution for parallel subgraph invocation)

**Independent features (can be built in any order, before or after the chain):**
- Resilient HTTP Client
- Prompt Template Management
- Agent Context Window Management
- Large Payload Externalization (though benefits from being available before subgraphs produce large intermediate data)

---

## Cross-Platform Comparison

### Parallel Execution

| Platform | Mechanism | Dynamic? | State Merge | Governance |
|----------|-----------|----------|-------------|------------|
| **LangGraph** | Send() API + state reducers | Yes (runtime) | operator.add reducer on annotated state fields | None built-in |
| **Temporal** | workflow.ExecuteActivity() futures + Promise.all() | Yes | Manual aggregation in workflow code | Activity-level timeouts, retries |
| **Prefect** | task.map() over iterables | Yes | Automatic via Prefect state | Task-level retries, timeouts |
| **Airflow** | expand() dynamic task mapping | Yes (since 2.3) | XCom push/pull | Pool-based concurrency limits |
| **Zeroth (proposed)** | Fan-out edge type or Send-style API | Yes (runtime) | Computed mapping with reduce expression | Per-branch budget, audit, policy -- **unique** |

### Subgraph Composition

| Platform | Mechanism | State Isolation | Governance Inheritance |
|----------|-----------|----------------|----------------------|
| **LangGraph** | Compiled subgraphs as nodes | Shared keys or isolated state | None |
| **Temporal** | Child Workflows | Full isolation (own event history) | Parent cancellation propagates |
| **Prefect** | Subflows | Shared flow run context | Parent flow settings inherited |
| **Airflow** | TaskGroup (UI-only) / deprecated SubDagOperator | TaskGroup: none. SubDag: full | None |
| **Zeroth (proposed)** | SubgraphNode referencing published Graph | Isolated Run with mapped I/O | Policy, capability, budget inheritance -- **unique** |

### Large Payload Externalization

| Platform | Mechanism | Threshold | Transparent? |
|----------|-----------|-----------|-------------|
| **Conductor** | ExternalPayloadStorage (S3/Azure/Postgres) | Configurable soft (5MB) / hard (10MB) barriers | Yes -- fully transparent to workers |
| **Temporal** | Payload Codec (DataDog large-payload-codec) | 2MB event history limit | Yes -- codec layer is transparent |
| **Argo Workflows** | Artifact repository (S3/GCS/Azure/HTTP) | All outputs are artifacts by default | No -- explicit artifact declarations |
| **Zeroth (proposed)** | ArtifactStore with pluggable backends (Redis/filesystem) | Configurable threshold | Yes -- transparent via ArtifactRef in run state |

### Context Window Management

| Platform | Mechanism | Strategies | Pluggable? |
|----------|-----------|-----------|------------|
| **Semantic Kernel** | ChatHistoryReducer (v1.35.0+) | Truncation, Summarization | Yes -- custom reducers |
| **LangChain** | ConversationSummaryBufferMemory | Summary buffer (fixed window + summary) | Limited |
| **Anthropic (guidance)** | Context engineering principles | Compaction, sub-agents, just-in-time retrieval | Framework-level guidance |
| **Zeroth (proposed)** | ContextWindowManager with pluggable strategies | Truncation, observation masking, summarization | Yes -- strategy pattern with per-node config |

---

## MVP Recommendation

### Phase 1: Foundation (build first, unblocks everything)
1. **Computed data mappings** -- Lowest complexity, highest leverage. Adds `transform` operation to MappingOperation union. Reuses _SafeEvaluator. Unblocks fan-in aggregation expressions.
2. **Prompt template management** -- Low-medium complexity, independent. Versioned registry pattern already proven in contracts/registry.py. Immediate value for agent authoring.
3. **Resilient HTTP client** -- Medium complexity, independent. Well-understood patterns (httpx + tenacity). Immediate value for any agent calling external APIs.

### Phase 2: Parallel Execution (hardest, most impactful)
4. **Parallel fan-out/fan-in** -- High complexity. Requires RuntimeOrchestrator changes, per-branch isolation, result aggregation via computed mappings.
5. **Large payload externalization** -- Medium complexity. Should land alongside or before parallel execution since parallel branches multiply intermediate data.

### Phase 3: Composition and Intelligence
6. **Subgraph composition** -- High complexity, builds on parallel execution patterns. Governance inheritance is the differentiator.
7. **Agent context window management** -- Medium complexity, but benefits from all other features being stable. Token tracking already exists; this adds strategy layer.

**Defer:** Context management can move earlier if there is user demand -- it is independent of the dependency chain.

---

## Complexity Budget

| Feature | Est. LOC | Est. Tests | Risk Level | Confidence |
|---------|---------|-----------|------------|------------|
| Computed data mappings | 150-250 | 30-40 | LOW | HIGH -- expression engine exists |
| Prompt template management | 300-500 | 40-60 | LOW | HIGH -- registry pattern exists |
| Resilient HTTP client | 400-600 | 50-70 | LOW | HIGH -- httpx+tenacity well-proven |
| Parallel fan-out/fan-in | 800-1200 | 80-120 | HIGH | MEDIUM -- orchestrator surgery required |
| Large payload externalization | 400-600 | 40-60 | MEDIUM | HIGH -- Conductor/Temporal patterns clear |
| Subgraph composition | 600-900 | 60-90 | HIGH | MEDIUM -- governance inheritance is novel |
| Agent context window management | 400-600 | 50-70 | MEDIUM | HIGH -- Semantic Kernel patterns clear |

**Total estimate:** ~3,050-4,650 LOC source + ~350-510 tests

---

## Sources

### Parallel Fan-Out/Fan-In
- [LangGraph Map-Reduce with Send() API](https://medium.com/ai-engineering-bootcamp/map-reduce-with-the-send-api-in-langgraph-29b92078b47d) -- MEDIUM confidence
- [Temporal Fan-Out Child Workflows](https://community.temporal.io/t/long-running-workflow-with-significant-fan-out-of-child-workflows/17975) -- HIGH confidence
- [Temporal Child Workflow Docs](https://docs.temporal.io/child-workflows) -- HIGH confidence
- [Prefect Task Mapping: Scaling to Thousands](https://www.prefect.io/blog/beyond-loops-how-prefect-s-task-mapping-scales-to-thousands-of-parallel-tasks) -- MEDIUM confidence
- [Airflow Dynamic Task Mapping Docs](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html) -- HIGH confidence
- [LangGraph Parallelization Techniques](https://deepwiki.com/langchain-ai/langchain-academy/7.3-parallelization-techniques) -- MEDIUM confidence

### Subgraph Composition
- [LangGraph Subgraphs Official Docs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs) -- HIGH confidence
- [Temporal Child Workflows Docs](https://docs.temporal.io/child-workflows) -- HIGH confidence
- [Airflow TaskGroup vs SubDagOperator](https://www.astronomer.io/docs/learn/task-groups) -- HIGH confidence
- [Scaling LangGraph: Subgraphs Trade-Offs](https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization) -- MEDIUM confidence
- [Airflow SubDagOperator Deprecation (AIP-34)](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-34+TaskGroup:+A+UI+task+grouping+concept+as+an+alternative+to+SubDagOperator) -- HIGH confidence

### Large Payload Externalization
- [Conductor External Payload Storage Docs](https://conductor-oss.github.io/conductor/documentation/advanced/externalpayloadstorage.html) -- HIGH confidence
- [Temporal Large Payload Codec (DataDog)](https://github.com/DataDog/temporal-large-payload-codec) -- HIGH confidence
- [Temporal Payload Codec Docs](https://docs.temporal.io/payload-codec) -- HIGH confidence
- [Temporal Data Conversion Docs](https://docs.temporal.io/dataconversion) -- HIGH confidence

### Agent Context Window Management
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) -- HIGH confidence
- [Semantic Kernel Python Context Management](https://devblogs.microsoft.com/semantic-kernel/semantic-kernel-python-context-management/) -- HIGH confidence
- [Semantic Kernel ChatHistoryReducer API](https://learn.microsoft.com/en-us/python/api/semantic-kernel/semantic_kernel.contents.history_reducer.chat_history_reducer.chathistoryreducer) -- HIGH confidence
- [JetBrains Research: Efficient Context Management (Dec 2025)](https://blog.jetbrains.com/research/2025/12/efficient-context-management/) -- HIGH confidence
- [Agenta: Top Techniques for Context Length](https://agenta.ai/blog/top-6-techniques-to-manage-context-length-in-llms) -- MEDIUM confidence
- [LLM Context Window Limitations 2026 (Atlan)](https://atlan.com/know/llm-context-window-limitations/) -- MEDIUM confidence

### Resilient HTTP
- [Tenacity Library (GitHub)](https://github.com/jd/tenacity) -- HIGH confidence
- [resilient-httpx (httpx + tenacity)](https://github.com/galthran-wq/resilient-httpx) -- MEDIUM confidence
- [pyresilience (retry + circuit breaker)](https://pypi.org/project/pyresilience/) -- MEDIUM confidence
- [Building Resilient HTTP Clients with Tenacity](https://medium.com/@ansh.chaturmohta/building-resilient-http-clients-a-deep-dive-into-retry-logic-with-pythons-tenacity-513bc927042b) -- MEDIUM confidence

### Prompt Template Management
- [Braintrust: Best Prompt Versioning Tools 2025](https://www.braintrust.dev/articles/best-prompt-versioning-tools-2025) -- MEDIUM confidence
- [Maxim AI: Prompt Versioning Best Practices 2025](https://www.getmaxim.ai/articles/prompt-versioning-and-its-best-practices-2025/) -- MEDIUM confidence
- [LaunchDarkly: Prompt Versioning Guide](https://launchdarkly.com/blog/prompt-versioning-and-management/) -- MEDIUM confidence
- [ZenML: Best Prompt Management Tools](https://www.zenml.io/blog/best-prompt-management-tools) -- MEDIUM confidence

### Computed Data Mappings
- [n8n Expression Engine Docs](https://docs.n8n.io/data/expressions/) -- HIGH confidence
- [n8n Data Transformation Approaches](https://docs.n8n.io/data/transforming-data/) -- HIGH confidence
- [n8n Expressions for Data Transformation](https://docs.n8n.io/data/expressions-for-transformation/) -- HIGH confidence
- [Azure Data Factory Expression Builder](https://learn.microsoft.com/en-us/azure/data-factory/concepts-data-flow-expression-builder) -- MEDIUM confidence

---
*Feature research for: v4.0 Platform Extensions for Production Agentic Workflows*
*Researched: 2026-04-12*
*Confidence: HIGH -- all 7 features have clear precedent in production platforms; zeroth's governance integration is the novel differentiator.*
