# Phase 43: Parallel Subgraph Fan-Out & Merge Strategies - Context

**Gathered:** 2026-04-14
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase closes the gap between declared and implemented orchestration composition:

1. Lift the current block on `SubgraphNode` inside fan-out branches so fan-out can invoke child workflows concurrently.
2. Enable fan-out nodes inside subgraphs (nested composition through recursive `_drive()`).
3. Implement the full declared family of merge strategies: `collect` (exists), `reduce` (fold), `merge` (shallow dict merge), and user-supplied custom reducers.
4. Reject invalid merge-strategy / reducer configurations at graph publish time.

No new node types are added. Subgraph remains the "call another published Zeroth workflow" primitive from Phase 39.

</domain>

<decisions>
## Implementation Decisions

### Merge Strategies
- **D-01:** `reduce` strategy is a **sequential left-to-right fold**. The reducer callable receives `(accumulator, next_branch_output)` and returns the new accumulator. Initial accumulator is branch index 0's output. No pairwise tree reduce.
- **D-02:** `merge` strategy is a **shallow dict merge** in branch-index order using `dict.update()`. All branch outputs must be dicts; later branches overwrite earlier keys. No deep recursive merge.
- **D-03:** Custom reducers are referenced by **dotted Python import path** on a new `ParallelConfig.reducer_ref` field (e.g., `"myapp.reducers.sum_scores"`). Resolved via `importlib` at runtime. No separate ReducerRegistry.
- **D-04:** `ParallelConfig.merge_strategy` literal expands to `Literal["collect", "reduce", "merge", "custom"]`. `custom` requires `reducer_ref` to be set.

### SubgraphNode in Parallel
- **D-05:** Remove the `FanOutValidationError` block in `ParallelExecutor.split_fan_out()` for `SubgraphNode`. No opt-in flag — lifting is unconditional.
- **D-06:** **Force per-branch isolated threads** for subgraphs inside parallel branches, regardless of `SubgraphNodeData.thread_participation`. Prevents concurrent writes to a shared thread from parallel branches. The `inherit` setting only applies to non-parallel SubgraphNode invocations.
- **D-07:** **Inherit parent depth** across branches. All branches share the same `subgraph_depth` ceiling from the parent run. Each branch's child subgraph increments by 1.
- **D-08:** **Shared `GlobalStepTracker`** — all branches (and any nested subgraphs they invoke) decrement the same global step budget from the parent run.
- **D-09:** **Cost rolls up to parent branch.** Each branch's child subgraph cost is accumulated into that branch's `BranchResult.cost_usd`; `FanInResult.total_cost_usd` aggregates across branches.
- **D-10:** **Audit records use branch-prefixed namespaced IDs.** Child run audit node IDs get `branch:{i}:subgraph:{ref}:{depth}:` prefixes so traces from parallel subgraph branches are distinguishable. Parent linkage via `parent_run_id` is preserved.
- **D-11:** **Approval pause is run-wide.** If any subgraph child run inside a parallel branch hits `WAITING_APPROVAL`, cancel/pause remaining branches and mark the parent run `WAITING_APPROVAL`. Resume re-executes from the paused branch (others are not re-driven).

### Nested Composition
- **D-12:** **Shared parent step budget** — fan-out inside a subgraph decrements the same `GlobalStepTracker` as the parent. Total work across all nesting levels is bounded by a single limit.
- **D-13:** **Reuse existing `SubgraphNodeData.max_depth`** — fan-out nesting inside subgraphs does not add a new depth dimension. Fan-out is lateral; subgraph call is vertical. Keep the single depth counter.
- **D-14:** **Standard branch isolation applies to inner fan-outs.** A subgraph's internal fan-out creates `BranchContext`s exactly as in a top-level graph; the child run's `_drive()` handles it normally. No compound/nested branch IDs.

### Registration Validation
- **D-15:** Validation runs **at graph publish time** (`DRAFT → PUBLISHED` transition). Invalid strategies block publishing. Extend `graph/validation.py`.
- **D-16:** Custom reducer `reducer_ref` validation does **full import + callable check** via `importlib.import_module`. Catches typos and missing modules before runtime.
- **D-17:** Merge-strategy **type compatibility is checked against node output contracts**:
  - `merge` requires the node's output contract to resolve to a dict-like schema
  - `reduce` / `custom` validate that the reducer callable is import-resolvable (signature is trusted)
  - `collect` has no contract requirement

### Error Propagation
- **D-18:** A failed subgraph child run inside a branch surfaces as `SubgraphExecutionError`, captured as the branch's error (fail_fast or best_effort mode decides cancellation semantics — same as any other branch failure).
- **D-19:** **No partial output** on child failure. Failed branch output is `None`; the child run's audit trail preserves partial work for debugging.

### Testing Strategy
- **D-20:** **In-memory `SubgraphResolver` stub** for composition tests. Test resolver returns pre-built `Graph` objects from a dict registry. Extends the pattern already used in Phase 39 tests. No SQLite/DeploymentService setup.
- **D-21:** Required composition scenarios:
  1. `SubgraphNode` inside fan-out branch (basic)
  2. Fan-out inside subgraph (nested)
  3. Three-level: fan-out → subgraph → inner fan-out
  4. Approval pause inside parallel subgraph branch (pause + resume)

### Backward Compatibility
- **D-22:** `ParallelConfig.merge_strategy` default stays `"collect"`. New strategies are opt-in via explicit config. Existing graphs require no migration.
- **D-23:** The current `FanOutValidationError` for SubgraphNode is **removed entirely** (not behind an opt-in flag). Existing graphs that never had SubgraphNode in fan-out branches are unaffected; any graph that previously triggered the error now works.

### Claude's Discretion
- Reducer callable signature convention: accumulator + next value → new accumulator. Planner/researcher can decide whether to accept 2-arg or also support 3-arg `(acc, branch_output, branch_index)`.
- Exact location of validation code (new `graph/merge_validation.py` vs extending existing validators) — planner decides.
- Whether to emit telemetry metrics for composition scenarios — optional, not a requirement.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Roadmap & Requirements
- `.planning/ROADMAP.md` §"Phase 43: Parallel Subgraph Fan-Out & Merge Strategies" — goal, success criteria, plan split
- `.planning/REQUIREMENTS.md` §"Orchestration Composition (ORCH)" — ORCH-01 through ORCH-04

### Existing Implementations (must be extended, not replaced)
- `src/zeroth/core/parallel/models.py` — `ParallelConfig` (extend `merge_strategy` literal + add `reducer_ref`), `BranchContext`, `BranchResult`, `FanInResult`, `GlobalStepTracker`
- `src/zeroth/core/parallel/executor.py` — `ParallelExecutor.split_fan_out()` (remove SubgraphNode block), `collect_fan_in()` (dispatch to strategy implementations)
- `src/zeroth/core/subgraph/executor.py` — `SubgraphExecutor.execute()` (handle parallel-branch context, force isolated threads when invoked from a branch)
- `src/zeroth/core/subgraph/resolver.py` — `merge_governance`, `namespace_subgraph` (audit ID prefix needs branch index when applicable)
- `src/zeroth/core/subgraph/models.py` — `SubgraphNodeData` (no changes, but referenced)
- `src/zeroth/core/orchestrator/runtime.py` — `RuntimeOrchestrator._drive()` (lines ~271-395) and subgraph dispatch branch, plus fan-out dispatch wiring
- `src/zeroth/core/graph/models.py` — `SubgraphNode`, `AgentNode`, `ExecutableUnitNode`, `HumanApprovalNode`, node type discriminators
- `src/zeroth/core/graph/validation.py` — graph structural validation (extend with merge-strategy validation)

### Codebase Maps
- `.planning/codebase/ARCHITECTURE.md` §"Orchestration Layer" — orchestrator loop, `_drive()` flow, dispatch pattern
- `.planning/codebase/ARCHITECTURE.md` §"Graph Domain" — graph lifecycle DRAFT → PUBLISHED → ARCHIVED
- `.planning/codebase/CONVENTIONS.md` — coding patterns to follow
- `.planning/codebase/TESTING.md` — test patterns and fixtures

### Prior Phase Work
- Phase 38 (Parallel Fan-Out/Fan-In) — current fan-out implementation foundation
- Phase 39 (Subgraph Composition) — SubgraphNode, SubgraphExecutor, resolver, governance merging, depth tracking
- Phase 41 (D-05) — the original "SubgraphNode-in-parallel rejected with clear validation error" decision this phase reverses

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **`ParallelExecutor`** — three-phase split/execute/collect pattern is sound. Phase 43 extends `collect_fan_in()` to dispatch to strategy handlers and drops one validation line in `split_fan_out()`.
- **`SubgraphExecutor`** — already handles depth tracking, cycle detection, governance merging, namespace isolation, child Run creation, recursive `_drive()`. Phase 43 threads a branch context through `execute()` so it knows to force isolated threads and prefix audit IDs.
- **`GlobalStepTracker`** — async-safe counter with `asyncio.Lock` already designed for shared use across branches. Works unchanged for nested composition (D-08, D-12).
- **`namespace_subgraph()`** in `subgraph/resolver.py` — prefixes node IDs with `subgraph:{ref}:{depth}:`. Extend to accept an optional `branch_index` for branch-prefixed variant.
- **`merge_governance()`** — parent-ceiling merge already works correctly for nested composition; no changes expected.
- **`_get_path` / `_set_path`** in `mappings/executor.py` — dot-path extraction used by fan-out split. Reuse for reducer target paths if needed.

### Established Patterns
- **Discriminated unions for node types** (`node_type: Literal[...]`). Merge strategy literal should follow the same Pydantic pattern.
- **Extra-forbid Pydantic models** (`ConfigDict(extra="forbid")`). Any new config fields on `ParallelConfig` must respect this.
- **Async-first orchestration** — all new composition code is `async`. `asyncio.gather`, `asyncio.Lock`, `return_exceptions=True` patterns are established.
- **Validation errors are typed and module-scoped** — create new error classes like `MergeStrategyValidationError` in `parallel/errors.py` or `graph/validation_errors.py`.
- **Audit namespacing via string prefix** — simple, consistent, query-friendly.
- **In-memory stub resolvers for tests** — Phase 39 already uses this pattern, extend for composition scenarios.

### Integration Points
- **`RuntimeOrchestrator._drive()`** — the dispatch point where node type is switched. Fan-out dispatch and subgraph dispatch both live here (lines ~271-395). Nested composition means each can invoke the other within the same `_drive()` loop.
- **`ServiceBootstrap`** in `service/bootstrap.py` — wires `ParallelExecutor` and `SubgraphExecutor` onto the orchestrator. No changes expected unless new config surface emerges.
- **Graph publish path** (`GraphRepository.publish()`) — where registration validation hooks in. Call new merge-strategy validator before status transition.

</code_context>

<specifics>
## Specific Ideas

- The user explicitly confirmed that "subgraph" means "callable other async Zeroth workflow" — the current Phase 39 `SubgraphNode` + `SubgraphResolver` is exactly the right primitive. No new node types are added in Phase 43.
- Sequential fold over pairwise tree reduce — simplicity and predictability preferred.
- Shallow dict merge over deep merge — simpler reasoning, explicit overrides.
- Dotted import path over named registry — consistent with "explicit over indirected" preference for new surface.
- Force-isolated threads in parallel branches — safety over flexibility; rules out an entire class of concurrent-thread-write bugs.
- Branch-prefixed audit IDs over metadata-only tagging — makes traces grep-friendly and self-describing.

</specifics>

<deferred>
## Deferred Ideas

- **Pairwise tree reduce** — rejected for now. Could be added later as an optional mode if a workload demonstrates need.
- **Deep recursive merge strategy** — not in scope. If demand emerges, add as a separate `merge_deep` strategy rather than changing `merge`.
- **Named ReducerRegistry** — not adopted. If cross-project reducer sharing becomes important, this can be layered on top of dotted-path resolution later.
- **Allow-subgraph-branches opt-in flag** — not adopted. Lifting the block is unconditional; no permanent config surface added.
- **Partial output capture from failed child runs** — not in scope. Could be added if users request it for best_effort aggregation.
- **Per-subgraph step budgets** — not in scope. Shared parent budget is the single source of truth.
- **Fan-out nesting depth cap** — not needed. Existing `SubgraphNodeData.max_depth` covers the vertical dimension; fan-out is lateral.
- **Reducer signature with branch index** (`(acc, output, branch_index)`) — planner/researcher decides whether to support.

</deferred>

---

*Phase: 43-parallel-subgraph-fan-out-merge-strategies*
*Context gathered: 2026-04-14*
