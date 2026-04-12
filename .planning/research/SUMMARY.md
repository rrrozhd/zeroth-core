# Project Research Summary

**Project:** zeroth-core v4.0 Platform Extensions
**Domain:** Governed agent orchestration platform -- parallel execution, composition, artifact stores, context management, resilient HTTP, prompt templates, computed mappings
**Researched:** 2026-04-12
**Confidence:** HIGH

## Executive Summary

Zeroth v4.0 extends the existing governed agent orchestration platform with 7 capabilities that production teams expect: parallel fan-out/fan-in execution, subgraph composition, large payload externalization, agent context window management, a resilient HTTP client, prompt template management, and computed data mappings. Research across LangGraph, Temporal, Prefect, Airflow, Conductor, Semantic Kernel, and n8n confirms that the first four are table stakes for real-world adoption, while context management, subgraph governance inheritance, and built-in prompt versioning are differentiators no competing platform does well. The critical finding is that all 7 features can be built with zero new PyPI dependencies -- the existing stack (asyncio.TaskGroup, httpx, tenacity, litellm, Jinja2, redis, pydantic) already provides every library needed.

The recommended approach follows a dependency-aware build order across three phases. Computed mappings and the artifact store ship first because they are foundational (mappings enable fan-in aggregation; artifacts prevent payload bloat during parallel execution). The resilient HTTP client, prompt templates, and context window management are independent and can be built in parallel. Parallel fan-out/fan-in and subgraph composition ship last because they are the most complex, touch the core orchestrator loop, and benefit from all earlier infrastructure being in place. This ordering minimizes risk: the hardest features build on stable foundations rather than being built first and retrofitted.

The primary risks are concentrated in two areas. First, parallel execution introduces shared-state mutation hazards -- the current `_drive()` loop mutates the `Run` object in-place, and concurrent branches will corrupt state unless each branch operates on an isolated context with atomic fan-in merging. Second, subgraph composition creates governance scope leak risks where child graphs inherit parent permissions they should not have, and infinite recursion through self-referencing graphs. Both require careful upfront design (isolated branch contexts, governance scoping, depth limits, and publish-time cycle detection) rather than bolt-on fixes.

## Key Findings

### Recommended Stack

All 7 features build on zeroth's existing dependency tree with no new packages. The only change to `pyproject.toml` is promoting Jinja2 from a transitive dependency (via litellm) to an explicit one, which changes nothing at install time but prevents breakage if litellm ever drops it. This is the optimal outcome: no version conflicts, no new supply-chain risk.

**Core technologies (all existing):**
- `asyncio.TaskGroup` (stdlib, Python 3.12+): Parallel fan-out/fan-in -- structured concurrency with automatic cancellation on failure, superior to `asyncio.gather()` for fail-fast semantics
- `litellm.token_counter()` (already pinned): Context window management -- model-aware tokenization for OpenAI, Anthropic, Cohere without adding tiktoken as a direct dependency
- `httpx.AsyncClient` + `tenacity` (both already pinned): Resilient HTTP -- retry with backoff on HTTP status codes (not just connection errors), connection pooling, layered timeouts
- `Jinja2 SandboxedEnvironment` (transitive via litellm): Prompt template rendering -- prevents template injection attacks
- `_SafeEvaluator` (existing in conditions/evaluator.py): Computed mappings -- already supports arithmetic, string ops, comparisons, ternary, list/dict construction; zero new parsing code needed
- `redis` (already in dispatch extra): Artifact store backend with `SETEX` for TTL-based storage
- Custom circuit breaker (~60 LOC): All async circuit breaker libraries are unmaintained; the pattern is simple enough to implement in-house with `asyncio.Lock`

**Do NOT add:** `aiobreaker`/`pybreaker` (unmaintained), `celery`/`dramatiq` (overkill for in-process parallelism), `networkx` (existing graph validation suffices), `jsonpath-ng`/`jmespath` (existing evaluator suffices), `aiofiles` (use `asyncio.to_thread` instead).

### Expected Features

**Must have (table stakes):**
- Parallel fan-out/fan-in -- every workflow platform (LangGraph Send(), Temporal activities, Prefect map(), Airflow expand()) supports this; users cannot model batch processing, multi-source retrieval, or parallel agent evaluation without it
- Computed data mappings -- current mappings (passthrough/rename/constant/default) cannot derive new values; this is a basic workflow need that n8n, Azure Data Factory, and Airflow all provide
- Resilient HTTP client -- agent nodes routinely call external APIs; without managed retry/backoff/circuit-breaking, any external dependency failure cascades into workflow failure
- Large payload externalization -- Temporal has a 2MB event history limit, Conductor enforces 3-5MB barriers; without this, graphs passing large LLM outputs or documents between nodes hit memory/storage limits

**Should have (differentiators):**
- Subgraph composition with governance inheritance -- most platforms treat subgraphs as second-class; zeroth can differentiate with first-class governance inheritance, thread continuity, approval propagation, and budget isolation across nested graphs
- Agent context window management -- most frameworks punt on this; zeroth's pluggable strategy pattern (observation masking by default, summarization opt-in) with per-node configuration would be a strong differentiator
- Prompt template management -- no production agent framework has built-in versioned prompt templates with audit redaction; self-contained alternative to external platforms like PromptLayer/Langfuse

**Defer (anti-features -- explicitly do NOT build):**
- Distributed parallel execution across workers (use in-process asyncio; scale by running more workers)
- Full expression language / DSL for mappings (existing `_SafeEvaluator` is sufficient)
- Automatic context summarization as default (default to observation masking; summarization is opt-in)
- HTTP response caching in the resilient client (separate concern from resilience)
- Subgraph runtime flattening (Airflow tried this and deprecated it; use recursive invocation with isolation)
- General-purpose object store (artifact store handles only workflow intermediate data with TTL cleanup)

### Architecture Approach

The architecture follows four key patterns already established in zeroth-core: (1) Protocol + Pluggable Backend for new subsystems (ArtifactStore, CompactionStrategy, GraphResolver), (2) Optional Injection via RuntimeOrchestrator fields defaulting to None for backward compatibility, (3) Governance-by-Default for every new capability touching external resources, and (4) Atomic Superstep for parallelism where all branches in a group must complete before state updates apply.

**Major components (new modules):**
1. `zeroth.core.orchestrator.parallel` -- ParallelDispatcher, BranchContext; handles concurrent branch spawning via asyncio.TaskGroup, isolated per-branch state, and atomic fan-in merging
2. `zeroth.core.orchestrator.subgraph` -- SubgraphRunner, GraphResolver; recursive graph invocation with governance inheritance, budget scoping, and depth limiting
3. `zeroth.core.artifacts` -- ArtifactStore protocol with RedisArtifactStore and FilesystemArtifactStore; transparent externalization of payloads exceeding configurable threshold
4. `zeroth.core.agent_runtime.context` -- ContextWindowManager with pluggable CompactionStrategy (truncation, observation masking, summarization); enforces token budgets in PromptAssembler
5. `zeroth.core.http` -- ResilientHTTPClient wrapping httpx with tenacity retry, custom circuit breaker, capability gating, and audit logging
6. `zeroth.core.prompts` -- PromptTemplateRegistry (versioned, DB-backed), TemplateRenderer (Jinja2 SandboxedEnvironment); mirrors ContractRegistry pattern
7. `zeroth.core.mappings` (extension) -- TransformMappingOperation added to existing MappingOperation union; delegates to existing `_SafeEvaluator`

**Existing components requiring modification:**
- `RuntimeOrchestrator._drive()` -- detect parallel groups, dispatch via ParallelDispatcher; add SubgraphNode handler
- `Run` model -- add `parallel_groups`, `parent_run_id` fields
- `MappingOperation` union -- add TransformMappingOperation variant
- `AgentConfig` / `AgentNodeData` -- add context_window_config, template_ref fields
- `PromptAssembler` -- add template resolution and token budget enforcement
- `ZerothSettings` -- add ArtifactSettings, HTTPClientSettings, ContextWindowSettings

### Critical Pitfalls

1. **Parallel fan-out corrupts shared Run state** -- The `_drive()` loop mutates Run in-place (visit counts, payloads, history, pending nodes). Concurrent branches will corrupt state at every `await` yield point. Prevention: each branch must operate on an isolated BranchContext; merge atomically at fan-in. This is the foundational design decision -- get it wrong and everything else breaks.

2. **Parallel branches break budget enforcement (N-fold cost multiplication)** -- BudgetEnforcer's TTL cache means N branches all read the same stale spend value and all pass. Prevention: pre-reserve total estimated cost before fan-out, enforce per-branch sub-budgets, reconcile actual spend after fan-in.

3. **Expression injection in computed mappings via AST evaluator escape** -- The `_SafeEvaluator` allows `ast.Attribute` access that can chain to `__class__.__bases__[0].__subclasses__()` (see CVE-2025-68613, CVSS 9.9 in n8n). Prevention: convert all namespace values to plain dicts via `.model_dump()`; denylist dunder attributes; block all function calls in mapping expressions; add fuzz testing.

4. **Artifact references become dangling after TTL expiry** -- Runs paused for approval or replayed from checkpoint will find artifact data evicted. Prevention: refresh TTLs on checkpoint writes; pin artifacts for paused runs; fail with specific `ArtifactExpiredError` rather than passing stale references.

5. **Subgraph governance scope leaks** -- Child graphs inherit parent permissions they should not have; node ID collisions across nested graphs corrupt enforcement records. Prevention: "parent is ceiling, child is floor" inheritance rule; namespace all node IDs in nested execution; create cascading GovernanceScope.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Foundation (Computed Mappings + Artifact Store)
**Rationale:** Computed mappings have zero dependencies, the smallest scope (~150-250 LOC), and are a prerequisite for fan-in aggregation expressions. The artifact store is also dependency-free and must be in place before parallel execution multiplies intermediate data. Building these first exercises the PR/test pipeline on low-risk changes.
**Delivers:** Transform mapping operation in edge mappings; pluggable artifact storage with Redis and filesystem backends; transparent externalization/resolution of large payloads.
**Addresses:** Table stakes: computed data mappings, large payload externalization.
**Avoids:** Pitfall 7 (expression injection) -- must harden `_SafeEvaluator` with dunder denylist and dict-only namespaces BEFORE exposing computed mappings to user input. Pitfall 5 (artifact TTL) -- must implement TTL refresh on checkpoint writes from day one.
**Estimated scope:** ~550-850 LOC + ~70-100 tests.

### Phase 2: Independent Capabilities (HTTP Client + Prompt Templates + Context Management)
**Rationale:** These three features touch entirely different subsystems (new http module, new prompts module, agent_runtime) with no overlap. They can be developed in parallel by different contributors. None depends on Phase 1 outputs, but landing them before the complex Phase 3 work means the platform has a broader capability surface to stress-test.
**Delivers:** Governed resilient HTTP client with retry/backoff/circuit-breaker; versioned prompt template registry with Jinja2 sandboxed rendering; pluggable context window management with observation masking, truncation, and summarization strategies.
**Addresses:** Table stakes: resilient HTTP client. Differentiators: prompt template management, agent context window management.
**Avoids:** Pitfall 9 (circuit breaker tenant leak) -- scope breakers by (tenant_id, host), not globally. Pitfall 10 (prompt injection via templates) -- type variables as trusted/untrusted, wrap untrusted in delimiters. Pitfall 6 (summarization data loss) -- default to observation masking, implement pinned facts for critical values.
**Estimated scope:** ~1,100-1,700 LOC + ~140-200 tests.

### Phase 3: Parallel Execution + Subgraph Composition
**Rationale:** These are the two hardest features (~1,400-2,100 LOC combined), both touch the core `_drive()` loop, and they share critical integration surfaces (subgraph nodes inside parallel branches, tree-shaped budget carving, correlated audit trails). They benefit from all Phase 1-2 infrastructure being stable. They should be co-designed even if built sequentially (parallel first, then subgraph).
**Delivers:** Fan-out/fan-in parallel execution with per-branch isolation, budget scoping, and configurable failure policies; subgraph composition with governance inheritance, thread continuity, approval propagation, and recursion guards.
**Addresses:** Table stakes: parallel fan-out/fan-in. Differentiators: subgraph composition with governance inheritance.
**Avoids:** Pitfall 1 (shared state corruption) -- isolated BranchContext with atomic fan-in merge. Pitfall 2 (budget multiplication) -- pre-reserve before fan-out. Pitfall 8 (checkpoint corruption) -- ParallelExecutionGroup model, checkpoint only at fan-out start and fan-in complete. Pitfall 3 (infinite recursion) -- publish-time cycle detection + runtime depth guard. Pitfall 4 (governance scope leak) -- namespaced node IDs, GovernanceScope cascade.
**Estimated scope:** ~1,400-2,100 LOC + ~140-210 tests.

### Integration Phase
**Rationale:** After all features land, wire everything into service bootstrap, update OpenAPI spec, validate cross-feature interactions.
**Delivers:** End-to-end integration; updated API surface; cross-feature interaction testing.

### Phase Ordering Rationale

- **Dependency chain drives order:** Computed mappings enable fan-in aggregation (Phase 1 before Phase 3). Artifact store prevents payload bloat during parallel execution (Phase 1 before Phase 3).
- **Independence enables parallelism:** HTTP client, prompt templates, and context management touch different subsystems with no overlap (Phase 2 can be parallelized across contributors).
- **Risk is back-loaded intentionally:** The hardest features (parallel execution, subgraph composition) ship last, building on stable foundations. This avoids the antipattern of building the riskiest feature first and retrofitting everything else around it.
- **Pitfall avoidance is front-loaded:** Expression injection hardening (Pitfall 7) ships with computed mappings in Phase 1. TTL management (Pitfall 5) ships with the artifact store in Phase 1. These are security-critical and must not be deferred.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Parallel Execution):** The superstep model, branch isolation, checkpoint extension, and budget reservation require careful design. The orchestrator surgery is the highest-risk change in the entire v4.0 scope. Recommend `/gsd-research-phase` for the parallel dispatch design before implementation.
- **Phase 3 (Subgraph Composition):** Governance inheritance rules (parent-ceiling/child-floor), cross-graph cycle detection at publish time, and approval propagation across nested runs are novel -- no existing platform does this well. Recommend `/gsd-research-phase` for governance inheritance design.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Computed Mappings):** Well-documented pattern -- add one model variant and one case to the executor. Existing `_SafeEvaluator` is already battle-tested.
- **Phase 1 (Artifact Store):** Conductor and Temporal patterns are thoroughly documented. Protocol + pluggable backend is an established zeroth pattern.
- **Phase 2 (HTTP Client):** httpx + tenacity is the standard Python resilience stack. Circuit breaker is ~60 LOC.
- **Phase 2 (Prompt Templates):** Registry pattern already proven in ContractRegistry. Jinja2 SandboxedEnvironment is well-documented.
- **Phase 2 (Context Management):** Semantic Kernel's ChatHistoryReducer pattern is well-documented. litellm token counting API is stable.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All dependencies verified in local environment; version compatibility confirmed; zero new packages needed |
| Features | HIGH | Cross-platform evidence from 7 production platforms (LangGraph, Temporal, Prefect, Airflow, Conductor, Semantic Kernel, n8n); clear table-stakes vs differentiator classification |
| Architecture | HIGH | Existing codebase verified via direct source reading; integration points mapped to specific files and methods; patterns validated against production platforms |
| Pitfalls | HIGH | 10 pitfalls identified with specific code-level corruption vectors; cross-feature integration pitfalls documented; CVE-2025-68613 provides concrete evidence for expression injection risk |

**Overall confidence:** HIGH

### Gaps to Address

- **Parallel branch failure policy design:** The research identifies fail-fast, fail-after-all, and continue-with-partial as options, but the optimal default and configuration UX need design work during Phase 3 planning.
- **Budget reservation mechanics:** Pre-reserving cost before fan-out requires estimating per-branch cost. The estimation accuracy for different LLM providers and the reconciliation protocol with Regulus need specification.
- **Subgraph approval propagation UX:** When a child subgraph hits an approval gate, how does this surface in the parent run's API and Studio UI? The approval service integration needs design.
- **Jinja2 vs str.format_map for templates:** STACK.md recommends Jinja2 (SandboxedEnvironment, conditionals, loops); FEATURES.md recommends str.format_map (simpler). Resolution: use Jinja2 SandboxedEnvironment -- the security properties (sandbox) and expressiveness (conditionals for complex prompts) justify the negligible complexity increase over str.format_map.
- **Observation masking implementation details:** JetBrains research shows observation masking outperforms summarization (2.6% higher solve rate, 52% cheaper), but the specific masking strategy (replace with placeholder, replace with summary, replace with hash) needs specification during Phase 2 planning.

## Sources

### Primary (HIGH confidence)
- Zeroth codebase direct inspection: `runtime.py`, `evaluator.py`, `models.py`, `budget.py`, `prompt.py`
- [Python asyncio.TaskGroup docs](https://docs.python.org/3/library/asyncio-task.html)
- [LiteLLM Token Counting docs](https://docs.litellm.ai/docs/count_tokens)
- [httpx Resource Limits / Transports docs](https://www.python-httpx.org/advanced/resource-limits/)
- [Temporal Child Workflows docs](https://docs.temporal.io/child-workflows)
- [Conductor External Payload Storage docs](https://conductor-oss.github.io/conductor/documentation/advanced/externalpayloadstorage.html)
- [Semantic Kernel ChatHistoryReducer](https://devblogs.microsoft.com/semantic-kernel/semantic-kernel-python-context-management/)
- [JetBrains Research: Efficient Context Management (Dec 2025)](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [Airflow Dynamic Task Mapping docs](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)
- [LangGraph Subgraphs docs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs)
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### Secondary (MEDIUM confidence)
- [CVE-2025-68613: n8n Expression Injection RCE (CVSS 9.9)](https://nvd.nist.gov/vuln/detail/CVE-2025-68613)
- [OWASP LLM01:2025 Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
- [n8n Expression Engine docs](https://docs.n8n.io/data/expressions/)
- [LangGraph Map-Reduce with Send() API](https://medium.com/ai-engineering-bootcamp/map-reduce-with-the-send-api-in-langgraph-29b92078b47d)
- [Scaling LangGraph Agents: Parallelization, Subgraphs, Map-Reduce](https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization)
- [Braintrust: Best Prompt Versioning Tools 2025](https://www.braintrust.dev/articles/best-prompt-versioning-tools-2025)
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot)

### Tertiary (LOW confidence)
- None -- all findings corroborated by multiple sources.

---
*Research completed: 2026-04-12*
*Ready for roadmap: yes*
