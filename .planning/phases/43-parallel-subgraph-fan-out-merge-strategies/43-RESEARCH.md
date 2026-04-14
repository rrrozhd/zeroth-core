# Phase 43: Parallel Subgraph Fan-Out & Merge Strategies — Research

**Researched:** 2026-04-14
**Domain:** Orchestration composition — fan-out × subgraph × reducer strategies
**Confidence:** HIGH (all findings grounded in current repo code via direct Read/Grep)

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Merge Strategies**
- **D-01:** `reduce` strategy is a sequential left-to-right fold. Reducer callable receives `(accumulator, next_branch_output)` and returns the new accumulator. Initial accumulator is branch index 0's output. No pairwise tree reduce.
- **D-02:** `merge` strategy is a shallow dict merge in branch-index order using `dict.update()`. All branch outputs must be dicts; later branches overwrite earlier keys. No deep recursive merge.
- **D-03:** Custom reducers are referenced by dotted Python import path on a new `ParallelConfig.reducer_ref` field (e.g., `"myapp.reducers.sum_scores"`). Resolved via `importlib` at runtime. No separate ReducerRegistry.
- **D-04:** `ParallelConfig.merge_strategy` literal expands to `Literal["collect", "reduce", "merge", "custom"]`. `custom` requires `reducer_ref` to be set.

**SubgraphNode in Parallel**
- **D-05:** Remove the `FanOutValidationError` block in `ParallelExecutor.split_fan_out()` for `SubgraphNode`. No opt-in flag — lifting is unconditional.
- **D-06:** Force per-branch isolated threads for subgraphs inside parallel branches, regardless of `SubgraphNodeData.thread_participation`. `inherit` only applies to non-parallel SubgraphNode invocations.
- **D-07:** Inherit parent depth across branches. All branches share the same `subgraph_depth` ceiling from the parent run. Each branch's child subgraph increments by 1.
- **D-08:** Shared `GlobalStepTracker` — all branches (and any nested subgraphs they invoke) decrement the same global step budget.
- **D-09:** Cost rolls up to parent branch. Each branch's child subgraph cost accumulates into that branch's `BranchResult.cost_usd`; `FanInResult.total_cost_usd` aggregates across branches.
- **D-10:** Audit records use branch-prefixed namespaced IDs: `branch:{i}:subgraph:{ref}:{depth}:`. Parent linkage via `parent_run_id` preserved.
- **D-11:** Approval pause is run-wide. If any subgraph child run inside a parallel branch hits `WAITING_APPROVAL`, cancel/pause remaining branches and mark the parent run `WAITING_APPROVAL`. Resume re-executes from the paused branch (others are not re-driven).

**Nested Composition**
- **D-12:** Shared parent step budget — fan-out inside a subgraph decrements the same `GlobalStepTracker` as the parent.
- **D-13:** Reuse existing `SubgraphNodeData.max_depth` — no new depth dimension for inner fan-outs. Fan-out is lateral; subgraph call is vertical.
- **D-14:** Standard branch isolation applies to inner fan-outs. Subgraph's internal fan-out creates `BranchContext`s normally; child run's `_drive()` handles it.

**Registration Validation**
- **D-15:** Validation runs at graph publish time (`DRAFT → PUBLISHED`). Extend `graph/validation.py`.
- **D-16:** Custom `reducer_ref` validation does full `importlib.import_module` + callable check. Catches typos/missing modules before runtime.
- **D-17:** Merge-strategy type compatibility checked against node output contracts:
  - `merge` requires node's output contract resolves to dict-like schema
  - `reduce`/`custom` validate reducer callable is import-resolvable (signature trusted)
  - `collect` has no contract requirement.

**Error Propagation**
- **D-18:** Failed subgraph child run inside a branch surfaces as `SubgraphExecutionError`, captured as the branch's error (fail_fast vs best_effort controls cancellation — same as any other branch failure).
- **D-19:** No partial output on child failure. Failed branch output is `None`; child run audit trail preserves partial work.

**Testing**
- **D-20:** In-memory `SubgraphResolver` stub for composition tests. Returns pre-built `Graph` objects from a dict registry. No SQLite/DeploymentService setup.
- **D-21:** Required composition scenarios:
  1. `SubgraphNode` inside fan-out branch (basic)
  2. Fan-out inside subgraph (nested)
  3. Three-level: fan-out → subgraph → inner fan-out
  4. Approval pause inside parallel subgraph branch (pause + resume)

**Backward Compatibility**
- **D-22:** `ParallelConfig.merge_strategy` default stays `"collect"`. New strategies opt-in via explicit config. Existing graphs require no migration.
- **D-23:** Current `FanOutValidationError` for SubgraphNode is removed entirely (not behind an opt-in flag).

### Claude's Discretion
- Reducer callable signature: 2-arg `(acc, value)` or also 3-arg `(acc, value, branch_index)` — planner/researcher recommends convention.
- Validation code location: extend `graph/validation.py` vs new `graph/merge_validation.py`.
- Telemetry metrics for composition scenarios — optional, not a requirement.

### Deferred Ideas (OUT OF SCOPE)
- Pairwise tree reduce
- Deep recursive merge strategy
- Named ReducerRegistry
- Allow-subgraph-branches opt-in flag
- Partial output capture from failed child runs
- Per-subgraph step budgets
- Fan-out nesting depth cap
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ORCH-01 | Fan-out node can invoke subgraphs concurrently | Unblock `split_fan_out` (remove SubgraphNode reject), remove `SubgraphNode._reject_parallel_config` model validator, thread `BranchContext` into `SubgraphExecutor.execute()` for thread/namespace override |
| ORCH-02 | Subgraphs can contain fan-out nodes (nested fan-out) | Already works mechanically — `SubgraphExecutor` uses `orchestrator._drive()` recursively, which hits the same parallel dispatch at lines ~415-459; need only to verify shared `GlobalStepTracker` threading (D-08/D-12) |
| ORCH-03 | All declared reduce merge strategies implemented | Expand `ParallelConfig.merge_strategy` literal, add `reducer_ref` field, dispatch registry in `collect_fan_in()` with functions for `collect`, `reduce`, `merge`, `custom` |
| ORCH-04 | Merge strategy validation at graph registration time | New validation in `graph/validation.py` for `ParallelConfig` + wire `GraphValidator.validate_or_raise()` into `GraphRepository.publish()` (currently not called in production!) |
</phase_requirements>

## Summary

Phase 43 closes the gap between declared and implemented orchestration composition by (a) unblocking `SubgraphNode` inside fan-out branches, (b) implementing the full family of reduce-merge strategies, and (c) enforcing validation at graph publish time. The existing Phase 38/39 machinery is structurally sound — fan-out and subgraph both dispatch inside `RuntimeOrchestrator._drive()`, `SubgraphExecutor` already uses recursive `_drive()` so fan-out-inside-subgraph works for free once the outer block is lifted, and `GlobalStepTracker` is already async-safe for cross-branch sharing. The net work is: one model-validator deletion, one dispatch-branch deletion, one optional `BranchContext` parameter threaded through `SubgraphExecutor.execute()` + `namespace_subgraph()`, a strategy-dispatch registry in `collect_fan_in()`, new `ParallelConfig` fields with publish-time validation, and a new wiring call from `GraphRepository.publish()` to `GraphValidator.validate_or_raise()`.

**Primary recommendation:** Split into two plans along the current seam (43-01 = composition unlock + branch-aware subgraph wiring; 43-02 = merge-strategy dispatch + publish-time validation). The two are mostly independent and can be implemented in parallel. One cross-cutting risk (validator-not-wired-at-publish) needs a shared touchpoint in 43-02.

## Key Findings per Research Question

### Q1. `namespace_subgraph()` — minimal change for optional `branch_index`

**Current signature** (`src/zeroth/core/subgraph/resolver.py:65-85`):
```python
def namespace_subgraph(graph: Graph, graph_ref: str, depth: int) -> Graph:
    prefix = f"subgraph:{graph_ref}:{depth}:"
```

**Minimal change:** Add an optional `branch_index: int | None = None` parameter. When `None`, the prefix stays identical (`subgraph:{ref}:{depth}:`) so non-parallel subgraph traces are byte-identical — **zero regression** on Phase 39 tests. When set, prefix becomes `branch:{i}:subgraph:{ref}:{depth}:` (matches D-10 exactly).

```python
def namespace_subgraph(
    graph: Graph,
    graph_ref: str,
    depth: int,
    *,
    branch_index: int | None = None,
) -> Graph:
    if branch_index is None:
        prefix = f"subgraph:{graph_ref}:{depth}:"
    else:
        prefix = f"branch:{branch_index}:subgraph:{graph_ref}:{depth}:"
    ...
```

All three call sites (`SubgraphExecutor.execute()` line 113, `RuntimeOrchestrator._drive()` Path B line 304, the new parallel-branch path) already have the info they need. Path B (resume after approval) must re-pass the stored `branch_index` from `run.metadata["pending_subgraph"]` so the re-namespaced graph matches the originally-namespaced one — **this is a subtle bug-trap the planner must call out explicitly**.

### Q2. Parallel-branch SubgraphExecutor wiring — `BranchContext` threading

**Current signature** (`src/zeroth/core/subgraph/executor.py:49-57`):
```python
async def execute(
    self,
    orchestrator: RuntimeOrchestrator,
    parent_graph: Graph,
    parent_run: Run,
    node: SubgraphNode,
    node_id: str,
    input_payload: dict[str, Any],
) -> Run:
```

**Cleanest change:** Add `branch_context: BranchContext | None = None` as a keyword-only argument. The non-parallel call path at `runtime.py:351-358` passes nothing — byte-identical to today. The new parallel path passes the branch's `BranchContext`.

Inside `execute()`, four things conditionally change when `branch_context is not None`:
1. **Line 113 (namespace_subgraph):** pass `branch_index=branch_context.branch_index`
2. **Line 119-122 (thread_id resolution):** force `child_thread_id = ""` unconditionally (D-06), ignoring `subgraph_data.thread_participation`
3. **Line 96 (depth reading):** unchanged — already reads from `parent_run.metadata["subgraph_depth"]` (D-07 satisfied naturally because the parent `Run` is the same object across all branches)
4. **Step tracker:** no direct plumbing needed in `execute()`; the shared `GlobalStepTracker` lives in the closure of `branch_coro_factory` in `_execute_parallel_fan_out()` — the subgraph child run's `_drive()` does NOT touch it today (nested inner fan-outs create their *own* tracker at line 522-526, which is **wrong for D-12** — see Q9 below).

**Alternative rejected:** passing a full new "execution context" dataclass. Overkill — the two pieces of info actually needed are `branch_index` (for audit namespacing) and a flag "force isolated thread" which is equivalent to `branch_context is not None`.

### Q3. Run-wide approval pause semantics and branch cancellation

**Current non-parallel subgraph approval propagation** (`runtime.py:283-381`):
1. First pass (Path A, lines 349-381): `SubgraphExecutor.execute()` drives the child to completion. If child returns with `status == WAITING_APPROVAL`, parent stashes `pending_subgraph` metadata (`child_run_id`, `node_id`, `graph_ref`, `version`), sets parent to `WAITING_APPROVAL`, and re-queues the subgraph node for resume.
2. Second pass (Path B, lines 283-347): The resume call re-enters `_drive()` on the parent, sees `pending_subgraph` matching the current node, calls `resume_graph(subgraph, child_run_id)` instead of creating a new child run. When the child finally completes, it clears `pending_subgraph` and continues.

**Parallel-branch equivalent (D-11):**

The parallel executor runs branches via `asyncio.gather` (either `_execute_best_effort` or `_execute_fail_fast` in `parallel/executor.py:127-189`). The cleanest "run-wide pause" approach:

1. **Detection inside the branch coroutine:** When a branch's `SubgraphExecutor.execute()` returns a child run with `WAITING_APPROVAL`, the branch coroutine raises a new typed exception `BranchApprovalPauseSignal(branch_index, child_run_id, graph_ref, version, node_id)`. This exception is *not* a failure — it's a pause signal.
2. **Collection at fan-in:** Treat `BranchApprovalPauseSignal` specially in both `_execute_best_effort` and `_execute_fail_fast` — when any branch raises it, cancel remaining in-flight tasks via the existing fail-fast cancellation pattern (lines 173-180), then return a sentinel `FanInResult` with a new `pause_state` field capturing which branch paused and the child run details.
3. **Parent propagation:** `_execute_parallel_fan_out` in `runtime.py:473-618` catches the pause signal, stashes a new `pending_parallel_subgraph` metadata dict on the parent run (mirroring the existing `pending_subgraph` pattern), sets parent to `WAITING_APPROVAL`, and returns the run. The other branches' partial work is **discarded** per D-19 (no partial output on child failure — and a pause mid-run is functionally equivalent to "not done" for fan-in aggregation).
4. **Resume:** When the parent is resumed, it re-enters `_drive()`, hits the fan-out node again, sees `pending_parallel_subgraph` for that node, and re-drives **only the paused branch** (per D-11: "others are not re-driven"). This means the parent must re-invoke `SubgraphExecutor` resume logic for the paused branch specifically — not re-run the full parallel split.

**Key subtlety — cancellation semantics:** `asyncio.CancelledError` in Python 3.11+ is a `BaseException`, not `Exception`. The existing `_execute_fail_fast` handler catches `Exception` (line 172), so cancellation propagation works naturally. For best-effort mode, `asyncio.gather(return_exceptions=True)` converts `CancelledError` to a regular exception in the list — but the current code (lines 137-160) only special-cases `BaseException`, so this path needs verification. Recommendation: in best-effort mode, still cancel remaining tasks immediately on pause signal (D-11 says "run-wide pause" — best-effort doesn't mean "keep running other branches after a pause").

**Partial result collection on cancellation:** Per D-19, no partial output. The pause state only needs to record which branch/subgraph paused; everything else is discarded and re-computed on resume. This is simpler than trying to preserve sibling-branch outputs.

### Q4. Merge-strategy dispatch in `collect_fan_in`

**Recommendation: module-level registry dict of callables.** Consistent with existing patterns in the codebase (`ExecutableUnitRegistry` in `execution_units/runner.py`, dispatch via dict lookup in `RuntimeOrchestrator._dispatch_node`).

```python
# parallel/reducers.py (new file)

from collections.abc import Callable
from typing import Any

ReducerFn = Callable[[list[dict[str, Any] | None]], Any]

def _reduce_collect(outputs: list[dict[str, Any] | None]) -> list[dict[str, Any] | None]:
    return outputs  # already in branch-index order

def _reduce_merge(outputs: list[dict[str, Any] | None]) -> dict[str, Any]:
    merged: dict[str, Any] = {}
    for out in outputs:
        if out is None:
            continue
        if not isinstance(out, dict):
            raise MergeStrategyError(f"merge requires dict outputs, got {type(out).__name__}")
        merged.update(out)
    return merged

def _reduce_fold(outputs: list[dict[str, Any] | None], reducer: Callable) -> Any:
    # Filter None per D-19, then fold
    non_null = [o for o in outputs if o is not None]
    if not non_null:
        return None
    acc = non_null[0]
    for nxt in non_null[1:]:
        acc = reducer(acc, nxt)
    return acc

_STRATEGY_REGISTRY: dict[str, ...] = {
    "collect": _reduce_collect,
    "merge":   _reduce_merge,
    # "reduce" and "custom" dispatch to _reduce_fold with a resolved callable
}
```

**Why module-level dict over match statement:** easier to extend, testable in isolation, mirrors `ExecutableUnitRegistry` pattern. Match statement would bind logic to `collect_fan_in` — harder to unit-test in isolation.

**Why not strategy classes:** the strategies are pure functions, no state. Classes would add ceremony without benefit.

**Integration with existing `collect_fan_in`** (`parallel/executor.py:191-231`):

Current body builds `output_list` and calls `_set_path(merged_output, merge_path, output_list)` unconditionally. The refactor keeps `_set_path` for `collect` (unchanged) but for `reduce`/`merge`/`custom`, the reduced value goes at `split_path` instead of a list. Alternatively, introduce a new `merge_path` field (deferred) — for Phase 43, reuse `split_path` as the write target.

### Q5. Dict-like output contract detection (D-17)

**Discovery:** `ContractVersion.json_schema` in `src/zeroth/core/contracts/registry.py:88` stores the output contract as a raw dict JSON schema. Nodes reference contracts via `node.output_contract_ref` (string ref, see `NodeBase.output_contract_ref` in `graph/models.py:102`).

**Dict-like check at publish time is nuanced because:**
1. Publish-time validation runs on a `Graph` object, but the `ContractRegistry` is an async-DB-backed service — validators are currently synchronous (`GraphValidator.validate()` in `graph/validation.py:85`).
2. Not all output contracts are registered in the DB at graph-publish time (contracts can be attached after publish in some flows — verify).
3. Even if we had the `json_schema`, "dict-like" means `type == "object"` at the top level, which is a simple one-liner once we have the schema.

**Recommendation for D-17 implementation:**
- Validation accepts a `ContractRegistry` dependency (inject it into `GraphValidator` or into a new `MergeStrategyValidator`). Make the validator `async` for the new checks — or add a separate `async validate_merge_strategies(graph)` method that runs alongside `GraphValidator.validate()`.
- For nodes with `parallel_config.merge_strategy == "merge"`, load `node.output_contract_ref` via the registry and assert `json_schema.get("type") == "object"`. If the contract isn't resolvable (missing ref, not registered), raise `MergeStrategyValidationError` with a clear "output contract must be registered and dict-shaped" message.
- **Pragmatic alternative:** if cross-cutting async validator refactor is too much scope, defer the `merge` output-contract check to runtime (first call to `_reduce_merge` will reject non-dict at runtime with a clear error). Publish-time check degrades to "merge_strategy=merge on a node that has `output_contract_ref is None` is an immediate error." This is weaker than D-17 but unblocks planning.
- **My recommendation:** do the strong version — inject `ContractRegistry` and make the merge-strategy validator async. The async-validator refactor is a 10-line change and matches the declared D-17 intent.

### Q6. importlib callable validation at publish time (D-16)

**Pattern:**
```python
def _resolve_reducer(reducer_ref: str) -> Callable:
    if ":" in reducer_ref:
        module_path, attr = reducer_ref.rsplit(":", 1)
    else:
        module_path, _, attr = reducer_ref.rpartition(".")
    if not module_path or not attr:
        raise MergeStrategyValidationError(f"invalid reducer_ref '{reducer_ref}'")
    try:
        module = importlib.import_module(module_path)
    except ImportError as exc:
        raise MergeStrategyValidationError(f"reducer module '{module_path}' not importable: {exc}") from exc
    if not hasattr(module, attr):
        raise MergeStrategyValidationError(f"reducer '{attr}' not found in module '{module_path}'")
    fn = getattr(module, attr)
    if not callable(fn):
        raise MergeStrategyValidationError(f"reducer_ref '{reducer_ref}' is not callable")
    return fn
```

**Security note:** `importlib.import_module` executes the target module's top-level code. This is a real risk if untrusted graph authors can set `reducer_ref` — a malicious reducer_ref like `os` or `shutil` would import real modules with side effects. Mitigation options:
1. **Whitelist prefix:** require `reducer_ref` to start with a configured allowed prefix (e.g., `myapp.reducers.`). Simple, strong.
2. **Accept risk:** document that graph authors are trusted (aligned with Zeroth's existing graph-author trust model — they can already specify `ExecutableUnitNode.manifest_ref` which points at arbitrary code).
3. **Sandbox import:** not practical — no standard Python pattern exists.

**Recommendation:** accept risk (option 2). Graph authors are already trusted to reference arbitrary executable units and tools; reducer functions are a strictly weaker capability. Document the trust assumption in the reducer_ref field docstring. A whitelist could be added later as a soft-landing feature.

**Error type:** create `MergeStrategyValidationError` in `src/zeroth/core/parallel/errors.py` (next to `FanOutValidationError`). Inherits from `ParallelExecutionError` for consistency.

**Surfacing in publish flow:** `GraphRepository.publish()` at `graph/repository.py:81-87` currently does **zero validation** beyond status transition. `GraphValidator.validate_or_raise()` exists (`graph/validation.py:110`) but is never invoked in production code. Phase 43 must wire it in, which adds a cross-cutting risk: wiring the existing validator could retroactively reject already-broken graphs currently slipping through publish. **Mitigation:** scope the new `validate_or_raise` call to only run the new merge-strategy checks initially, or accept that existing graphs are already well-formed (the 280+ test suite would flag violations). Recommendation: run full validation on publish — it's overdue anyway.

### Q7. Reducer call signature convention

**Recommendation: 2-arg `(acc, value)` only, no 3-arg variant.**

Rationale:
- Python's `functools.reduce` is the ecosystem standard: `reduce(fn, iter, initial)` where `fn(acc, item)` — 2-arg.
- The branch index is already encoded in list order (`outputs` is sorted by `branch_index` before reduction in `collect_fan_in`). If a reducer needs the index, it can build it from enumeration over the pre-sorted list — but there's no realistic use case cited for this need.
- 3-arg support doubles validation/introspection complexity (signature inspection via `inspect.signature`, handling wrong-arity callables).
- Codebase precedent: no existing fold/reduce helpers in the repo — this is the first. Set the simpler standard.

**Concrete contract:**
```python
ReducerFn = Callable[[dict[str, Any], dict[str, Any]], dict[str, Any]]
```
Reducer takes two dicts (each a branch output), returns a new dict. Sequential fold starts with branch 0's output as initial accumulator.

### Q8. Force-isolated thread semantics

**Current logic** (`src/zeroth/core/subgraph/executor.py:118-123`):
```python
if subgraph_data.thread_participation == "inherit":
    child_thread_id = parent_run.thread_id
else:
    child_thread_id = ""  # empty triggers auto-generation in Run validator
```

**Override mechanism for parallel branches:** if `branch_context is not None` (new kwarg from Q2), skip the conditional entirely and force `child_thread_id = ""`. The `SubgraphNodeData.thread_participation` setting is effectively ignored when invoked from a parallel branch.

**Surfacing the override to graph authors:** the decision is silent — a graph author who set `thread_participation="inherit"` and then puts the subgraph in a fan-out branch will get isolated threads. This is defensible under D-06 ("safety over flexibility") but deserves a docstring note on `SubgraphNodeData.thread_participation` and potentially a debug-level log line when the override fires.

### Q9. GlobalStepTracker sharing — nested inheritance bug

**Current implementation** (`parallel/models.py:90-113`): `GlobalStepTracker` is already `asyncio.Lock`-guarded and designed for cross-branch sharing — D-08 is satisfied trivially at the outer fan-out level.

**Bug-trap for D-12 (shared budget across nested fan-outs):** when fan-out lives inside a subgraph, the inner `_execute_parallel_fan_out` call in the child run's `_drive()` creates a *fresh* `GlobalStepTracker` at `runtime.py:523-526`:
```python
step_tracker = GlobalStepTracker(
    current_steps=len(run.execution_history),
    max_steps=graph.execution_settings.max_total_steps,
)
```
The child run has its own `execution_history` and its own `graph.execution_settings.max_total_steps`. This violates D-12 ("shared parent step budget").

**Fix:** thread a `GlobalStepTracker | None` through the orchestrator's `_drive` → `SubgraphExecutor.execute` → child `_drive` → child `_execute_parallel_fan_out` chain. When present, reuse it instead of constructing a new one. The tracker lives for the lifetime of the outer run.

**Implementation options:**
1. Add `step_tracker` as an attribute on `Run.metadata` — rejected, `GlobalStepTracker` is not JSON-serializable.
2. Add `step_tracker` as a keyword argument on `_drive()` and `SubgraphExecutor.execute()` — clean, testable.
3. Attach it to a contextvar (`contextvars.ContextVar`) — works with asyncio, but hidden state is harder to reason about.

**Recommendation: option 2** (explicit kwarg). Add `step_tracker: GlobalStepTracker | None = None` to `_drive()` signature; `_execute_parallel_fan_out()` either uses the passed-in tracker or creates a new one; `SubgraphExecutor.execute()` passes its received `step_tracker` to the recursive `_drive()` call. Non-parallel call paths pass nothing — zero regression.

### Q10. Branch cost rollup

**Current state:**
- `BranchResult.cost_usd: float = 0.0` (`parallel/models.py:73`) exists but is **never populated** — grep shows no writer.
- `FanInResult.total_cost_usd = sum(r.cost_usd for r in sorted_results)` (`parallel/executor.py:223`) aggregates, but since branches never set their cost, this is always 0.0.
- Per-node cost instrumentation happens in `_dispatch_node` via `InstrumentedProviderAdapter` (visible around `runtime.py:725-740`), which writes cost into audit records but not into the branch's `BranchResult`.

**D-09 requirement:** each branch's child-subgraph cost must roll up into that branch's `BranchResult.cost_usd`.

**Plumbing required:**
1. Child run's `final_output` or metadata gets a `total_cost_usd` field aggregated during `_drive()` — find where per-step costs are aggregated today. (Grep suggests `run.metadata` gets cost info via the audit records, not a first-class rollup — verify.)
2. In the `branch_coro_factory` in `_execute_parallel_fan_out`, after each downstream node dispatch, accumulate cost into `ctx.metadata["cost_usd"]` or equivalent.
3. After `execute_branches` returns, the loop at `runtime.py:612-615` currently copies `audit_refs` and `execution_history` onto `BranchResult` — extend to copy `cost_usd` from `ctx.metadata`.
4. For subgraph child runs inside branches: the child run's aggregated cost must be readable after `SubgraphExecutor.execute()` returns — add a helper `_sum_run_cost(run)` that walks `run.execution_history` entries' audit records, or attach a running total to `Run.metadata["total_cost_usd"]` during `_drive()`.

**Scope decision:** full cost rollup is a non-trivial refactor. Two-phase recommendation:
- **Phase 43 minimum:** ensure subgraph child-run cost is summable via existing audit trail, roll into branch `cost_usd` via a helper. Acceptable even if existing non-subgraph fan-out cost tracking is still broken.
- **Follow-up:** full per-step cost aggregation from audit into `Run.metadata` is a separate concern — can be deferred or handled in a separate plan.

**Verification needed in planning:** does any existing code write non-zero `BranchResult.cost_usd`? If no, D-09 cost rollup is net-new work and the plan should scope it explicitly rather than treating it as a minor tweak.

## Recommended Architecture

### Module/Function Changes

**`src/zeroth/core/parallel/models.py`**
- Expand `merge_strategy: Literal["collect", "reduce"] = "collect"` → `Literal["collect", "reduce", "merge", "custom"] = "collect"`
- Add `reducer_ref: str | None = None`
- Add `model_validator(mode="after")` that requires `reducer_ref` when `merge_strategy == "custom"` and forbids `reducer_ref` otherwise

**`src/zeroth/core/parallel/errors.py`**
- Add `MergeStrategyError(ParallelExecutionError)` (runtime failures)
- Add `MergeStrategyValidationError(ParallelExecutionError)` (publish-time failures)
- Add `BranchApprovalPauseSignal(BaseException)` — **subclass `BaseException` not `Exception`** so `asyncio.gather(return_exceptions=True)` doesn't swallow it, and so it cleanly cancels sibling branches via `asyncio.gather`

**`src/zeroth/core/parallel/reducers.py`** (new)
- Module-level functions: `_reduce_collect`, `_reduce_merge`, `_reduce_fold`
- `resolve_reducer_ref(reducer_ref: str) -> Callable` for runtime resolution
- `_STRATEGY_REGISTRY` dict mapping strategy name → handler

**`src/zeroth/core/parallel/executor.py`**
- `split_fan_out()`: remove the `SubgraphNode` reject block (lines 67-73). Keep `HumanApprovalNode` reject.
- `collect_fan_in()`: dispatch to strategy handlers via registry. For `collect`, keep current behavior. For `merge`/`reduce`/`custom`, call the reducer; write the reduced value at `split_path` in the merged output.
- `execute_branches()`: handle `BranchApprovalPauseSignal` — in both fail-fast and best-effort, catch it, cancel siblings, return a tagged result (new `FanInResult.pause_state` field or a raised wrapper exception).

**`src/zeroth/core/parallel/models.py` (FanInResult)**
- Add `pause_state: dict | None = None` for approval-pause propagation

**`src/zeroth/core/subgraph/resolver.py`**
- `namespace_subgraph()`: add optional keyword `branch_index: int | None = None`; branch-prefix when set

**`src/zeroth/core/subgraph/executor.py`**
- `SubgraphExecutor.execute()`: add kwarg `branch_context: BranchContext | None = None` and `step_tracker: GlobalStepTracker | None = None`. When `branch_context is not None`: force isolated thread, pass `branch_index` to `namespace_subgraph`. When `step_tracker is not None`: pass it through to the recursive `_drive()` call.

**`src/zeroth/core/graph/models.py`**
- `SubgraphNode._reject_parallel_config` model validator (lines 255-261): **delete entirely** (D-05/D-23 require allowing `parallel_config` on SubgraphNode)

**`src/zeroth/core/orchestrator/runtime.py`**
- `_drive()`: add `step_tracker: GlobalStepTracker | None = None` kwarg; pass through to `_execute_parallel_fan_out`
- `_execute_parallel_fan_out()`: if passed a tracker, reuse instead of constructing. Detect `BranchApprovalPauseSignal` from `FanInResult.pause_state`; stash new `pending_parallel_subgraph` run metadata; set parent `WAITING_APPROVAL`. On resume path, re-enter fan-out node, detect pending state, resume only the paused branch.
- `_execute_parallel_fan_out.branch_coro_factory()`: handle downstream node that is `SubgraphNode` — detect via `isinstance(ds_node, SubgraphNode)`, dispatch through `self.subgraph_executor.execute(..., branch_context=ctx, step_tracker=step_tracker)`, catch approval-pause propagation from child, convert to `BranchApprovalPauseSignal`.

**`src/zeroth/core/graph/validation.py`**
- Add `_validate_parallel_configs(graph, issues)` method to `GraphValidator`; called from `validate()`
- Per-node check: if `node.parallel_config is not None`, validate strategy literal, reducer_ref consistency, and (for `merge`) output contract dict-shape via injected `ContractRegistry`
- Reducer import check via `_resolve_reducer` helper in `parallel/reducers.py`
- Consider making the merge-contract check async (inject `ContractRegistry`), or degrade to "output_contract_ref must be non-None when merge_strategy=merge" and enforce the type check at runtime

**`src/zeroth/core/graph/repository.py`**
- `publish()` at line 81: call `GraphValidator.validate_or_raise(graph)` before `save(graph.publish())`. **This is net-new production wiring** — `GraphValidator` is currently only invoked from tests. Needs a `GraphValidator` instance — inject at repository construction or construct inline.

**`src/zeroth/core/service/bootstrap.py`**
- If `GraphValidator` needs `ContractRegistry` injection, wire it through — `ContractRegistry` is already constructed in bootstrap. Ensure `GraphRepository` is constructed with a validator.

### Data Flow: Parallel Subgraph + Merge Strategy

```
_drive(graph, run, step_tracker=None)
  │
  ├─ pop node from pending_node_ids → this is a regular AgentNode with parallel_config
  ├─ dispatch node → output_data = {items: [...]}
  ├─ parallel_config is set → _execute_parallel_fan_out(..., step_tracker)
  │     │
  │     ├─ split_fan_out() → [BranchContext 0, 1, 2]
  │     ├─ step_tracker = passed-in OR GlobalStepTracker(...)
  │     ├─ asyncio.gather(branch_coro_factory(ctx) for ctx in ...)
  │     │     │
  │     │     └─ branch_coro_factory(ctx):
  │     │           for ds_node in downstream:
  │     │              if isinstance(ds_node, SubgraphNode):
  │     │                 child_run = await subgraph_executor.execute(
  │     │                    ..., branch_context=ctx, step_tracker=step_tracker)
  │     │                 if child_run.status == WAITING_APPROVAL:
  │     │                    raise BranchApprovalPauseSignal(...)
  │     │                 branch_output = child_run.final_output
  │     │                 ctx.cost_usd += _sum_run_cost(child_run)
  │     │              else:
  │     │                 (existing dispatch logic)
  │     │              await step_tracker.increment()  # shared budget
  │     │
  │     ├─ collect_fan_in(branch_results, config, output_data)
  │     │     │
  │     │     └─ strategy = _STRATEGY_REGISTRY[config.merge_strategy]
  │     │        reduced = strategy(outputs, reducer=resolve_reducer_ref(config.reducer_ref) if custom)
  │     │        merged_output = dict(base_output); _set_path(merged_output, split_path, reduced)
  │     │
  │     └─ return FanInResult(..., pause_state=None)
  │
  └─ (if pause_state): stash pending_parallel_subgraph; set run WAITING_APPROVAL; return
```

### Cancellation Semantics for Approval Pause

1. Branch coroutine raises `BranchApprovalPauseSignal` (subclass of `BaseException`)
2. `asyncio.gather` propagates — in **fail-fast mode**, the existing `except Exception` clause at `parallel/executor.py:172` won't catch it (it's `BaseException`); add `except (Exception, BranchApprovalPauseSignal)` explicitly.
3. Cancel remaining tasks via the existing `for task in tasks: task.cancel()` pattern.
4. Return a `FanInResult` with `pause_state={"branch_index": i, "child_run_id": ..., "graph_ref": ..., "version": ..., "node_id": ...}`.
5. `_execute_parallel_fan_out` detects `pause_state`, stashes as `run.metadata["pending_parallel_subgraph"]`, returns the parent run in `WAITING_APPROVAL`.
6. Resume via `resume_graph(graph, run_id)` re-enters `_drive()`, which detects the pending state on the fan-out node and invokes a new code path that re-drives **only the paused branch's subgraph** via `subgraph_executor.resolver.resolve(...)` + `resume_graph` on the child — mirroring the existing Path B for non-parallel subgraphs (`runtime.py:286-347`). Other branches are **not** re-run (D-11); their previously-cancelled work is discarded (D-19).

**Unresolved:** the resume path means the paused branch runs *alone* after approval. If its output was supposed to be merged with sibling outputs via `merge_strategy=merge`, the siblings' outputs don't exist. **Two options:**
- **(a)** On resume, re-run all branches from scratch (simple, D-11 says "resume re-executes from the paused branch" — interpret as "from the paused branch of the original split, but recompute siblings"). Contradicts D-11 literal reading but semantically correct for merge.
- **(b)** Persist sibling branch outputs in pause metadata before cancelling, use them as-is on resume (contradicts D-19 "no partial output").

**Recommendation:** go with **(a)**, re-run all branches on resume. Document that approval-pause inside a parallel branch is expensive (double execution of siblings). This keeps D-19 clean (no partial output persisted) and produces correct merge results. D-11 "others are not re-driven" likely means "the non-parallel resume path doesn't fire for non-paused branches" — reconfirm interpretation with user during discuss-phase, but treat this as the default.

## Open Questions Resolved — Claude's Discretion

### Reducer Signature Convention
**Recommendation: 2-arg `(acc, value) -> acc` only.** See Q7. Matches `functools.reduce` ecosystem convention, simpler validation, no cited use case for 3-arg variant.

### Validation Code Location
**Recommendation: extend `graph/validation.py`.** Add a `_validate_parallel_configs(graph, issues, contract_registry=None)` method to the existing `GraphValidator` class. Reasons:
- `GraphValidator` already owns structural checks including per-node-type validation (`_validate_agent_node`, etc.) — merge-strategy checks are conceptually per-node.
- Avoids a new module that has to be imported and invoked separately.
- The new check needs the same `issues` list collection pattern and the same `_append_issue` helper.
- If the merge-contract check becomes async, use a new `async def validate_async(graph, contract_registry)` method rather than splitting files. One module, two entry points.

### Telemetry
**Recommendation: add minimal `logger.debug` statements, no metrics counters.** Debug logs at: branch-approval-pause signal received, merge-strategy dispatch chosen, reducer_ref resolved. No Prometheus/OTel metric counters — Zeroth's existing observability layer (`src/zeroth/core/observability/`) uses its own structured logging, not metrics counters, and adding counters for one phase's feature would be inconsistent. Users who need composition metrics can build them off the existing audit trail records.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `GraphValidator` not currently invoked at publish — wiring it may retroactively reject pre-existing broken graphs | MEDIUM | MEDIUM | Run full test suite on post-wiring change; if legacy graphs flagged, either fix them or add a feature flag `strict_publish_validation` for staged rollout |
| Audit ID collision: `branch:{i}:subgraph:{ref}:{depth}:` prefix could collide if same branch revisits same subgraph | LOW | HIGH (audit integrity) | Already mitigated by existing depth counter; add `branch_index` is strictly additive and each (branch, depth, ref) tuple is unique per run |
| `reducer_ref` importlib import executes arbitrary module top-level code | MEDIUM | MEDIUM (graph authors are already trusted) | Accept risk; document trust assumption; graph authors already have equivalent power via `ExecutableUnitNode.manifest_ref` |
| Approval-pause in parallel branch: siblings' cancelled work is re-run on resume (double cost) | HIGH | LOW (correctness preserved, just inefficient) | Document in user-facing cookbook; runs that pause inside parallel branches pay a re-execution cost on resume |
| `BranchApprovalPauseSignal` must subclass `BaseException` not `Exception` to survive `gather(return_exceptions=True)` | MEDIUM | HIGH (silent pause loss if wrong) | Test explicitly; add test `test_approval_pause_signal_is_base_exception` |
| `GlobalStepTracker` not threaded into nested `_drive()` calls today (D-12 gap) | HIGH | MEDIUM (nested fan-out budget escape) | Plan 43-01 must thread `step_tracker` kwarg through `_drive()` and `SubgraphExecutor.execute()`; existing Phase 39 tests should catch regressions |
| Cost rollup (D-09) is net-new: `BranchResult.cost_usd` is never populated today | HIGH | MEDIUM | Plan 43-01 must scope cost plumbing explicitly, not treat as free tweak. Consider a minimum subgraph-only rollup in 43-01, defer full per-step cost aggregation |
| Path B resume for non-parallel subgraph (`runtime.py:304`) re-namespaces without `branch_index` — if it was originally namespaced with `branch_index`, re-namespaced IDs won't match stored ones | MEDIUM | HIGH (resume broken) | When parent stashes `pending_parallel_subgraph`, also store `branch_index`. On resume, pass it back through `namespace_subgraph`. Test end-to-end. |
| Pydantic `model_config = ConfigDict(extra="forbid")` on `ParallelConfig` — adding `reducer_ref` field is safe, but old serialized graphs have no `reducer_ref` field (fine, default `None`); new graphs deserialized by old code would fail | LOW | LOW | Standard additive-field migration, no serialization break |

## Plan-Split Recommendation

**Keep the existing 2-plan split.** The current seam (43-01 composition + 43-02 merge strategies) is correct and the plans are mostly independent. Specifically:

### Plan 43-01: Parallel Subgraph Composition Unlock
**Scope:**
- Remove `SubgraphNode._reject_parallel_config` model validator (`graph/models.py:255-261`)
- Remove `split_fan_out` SubgraphNode reject (`parallel/executor.py:67-73`)
- Add `branch_context` kwarg to `SubgraphExecutor.execute()` with force-isolated-thread + branch-prefixed namespacing
- Add `branch_index` kwarg to `namespace_subgraph()`
- Thread `step_tracker` through `_drive()` + `SubgraphExecutor.execute()` for D-12 shared budget
- Detect `SubgraphNode` in `branch_coro_factory`; dispatch through subgraph executor
- `BranchApprovalPauseSignal` + run-wide approval pause plumbing (pending_parallel_subgraph metadata, resume path)
- Branch cost rollup plumbing (`BranchResult.cost_usd` population from child run)
- In-memory `SubgraphResolver` test stub helper
- Four required scenario tests (D-21)

**Requirements covered:** ORCH-01, ORCH-02

### Plan 43-02: Merge Strategies + Publish-Time Validation
**Scope:**
- `ParallelConfig.merge_strategy` literal expansion + `reducer_ref` field + consistency validator
- New `parallel/reducers.py` module: `_reduce_collect`, `_reduce_merge`, `_reduce_fold`, `resolve_reducer_ref`, `_STRATEGY_REGISTRY`
- `collect_fan_in` refactor to dispatch via registry
- `MergeStrategyError` + `MergeStrategyValidationError` in `parallel/errors.py`
- `GraphValidator._validate_parallel_configs` method + integration with `validate()`
- Wire `GraphValidator.validate_or_raise()` into `GraphRepository.publish()` (**net-new production wiring**)
- Bootstrap wiring to pass `ContractRegistry` into `GraphValidator` (if async contract check adopted)
- Unit tests for each reducer + publish-time validation rejection tests

**Requirements covered:** ORCH-03, ORCH-04

### Cross-Plan Dependency

43-02's new `ParallelConfig.reducer_ref` field is used by 43-01 tests (scenario 2: parallel subgraph with reduce strategy). **Resolution:** 43-02 goes first OR 43-01 uses only `collect` strategy in its tests and a follow-up test in 43-02 covers the composition×reduce case. Either works; 43-02-first is slightly cleaner.

**Alternative split considered and rejected:** splitting along "publish-time validation" as its own plan. Rejected because the validation is tightly coupled to the new `ParallelConfig` fields introduced in 43-02 — splitting them into different plans would just add coordination overhead.

## Code Context Snapshot

### Key existing code locations
- `src/zeroth/core/parallel/models.py:30` — `merge_strategy: Literal["collect", "reduce"]` (expand)
- `src/zeroth/core/parallel/executor.py:67-73` — SubgraphNode reject block (delete)
- `src/zeroth/core/parallel/executor.py:191-231` — `collect_fan_in` (refactor to dispatch)
- `src/zeroth/core/graph/models.py:255-261` — `SubgraphNode._reject_parallel_config` validator (delete)
- `src/zeroth/core/subgraph/resolver.py:65-85` — `namespace_subgraph` (add `branch_index` kwarg)
- `src/zeroth/core/subgraph/executor.py:49-57,113,118-123` — `execute()` signature/namespacing/thread_id
- `src/zeroth/core/orchestrator/runtime.py:273-407` — subgraph dispatch (Path A + Path B)
- `src/zeroth/core/orchestrator/runtime.py:473-618` — `_execute_parallel_fan_out` and `branch_coro_factory`
- `src/zeroth/core/orchestrator/runtime.py:522-526` — `GlobalStepTracker` construction (nesting bug)
- `src/zeroth/core/graph/validation.py:85-114` — `GraphValidator.validate` / `validate_or_raise`
- `src/zeroth/core/graph/repository.py:81-87` — `publish()` (not calling validator — wire it)
- `src/zeroth/core/parallel/errors.py` — error hierarchy (add MergeStrategy errors + BranchApprovalPauseSignal)
- `src/zeroth/core/contracts/registry.py:88` — `ContractVersion.json_schema` (source of truth for dict-like check)

### Testing assets to leverage
- `tests/subgraph/test_integration.py:60-180` — existing fixture patterns for mock `SubgraphResolver` via `_make_deployment_service` dict-backed stub; reuse for D-20
- `tests/parallel/test_executor.py` — existing ParallelExecutor test patterns
- `tests/subgraph/test_drive_subgraph.py:185` — existing `MagicMock(spec=SubgraphResolver)` pattern

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `GraphValidator` wiring at publish won't retroactively reject existing graphs (the 280+ test suite would already catch violations) | Risks §row1 | MEDIUM — if legacy graphs exist in persisted DB that fail new validation, publish breaks. Mitigation: run full test suite; add flag if needed |
| A2 | Graph authors are trusted to specify `reducer_ref` (equivalent trust to `ExecutableUnitNode.manifest_ref`) | Q6 §Security | MEDIUM — if Zeroth's trust boundary treats `reducer_ref` differently than exec unit manifests, a whitelist is needed |
| A3 | "Run-wide pause" (D-11) means re-running all branches on resume is acceptable | Q3 §Unresolved | LOW — user may prefer persisted sibling outputs; needs discuss-phase confirmation |
| A4 | No existing code populates `BranchResult.cost_usd` (cost rollup is net-new) | Q10, Risks §row8 | LOW-MEDIUM — needs grep verification before 43-01 estimation |
| A5 | `ContractVersion.json_schema` reliably has `type: "object"` at top for dict-shaped contracts (standard JSON schema convention) | Q5 | LOW — standard JSON schema |
| A6 | `asyncio.gather(return_exceptions=True)` treats `BaseException` subclasses differently from `Exception` | Risks §row5 | HIGH if wrong — but verified in Python docs |

## Sources

### Primary (HIGH confidence) — all direct repo Read/Grep
- `src/zeroth/core/parallel/models.py` (full) — ParallelConfig, BranchContext, BranchResult, FanInResult, GlobalStepTracker
- `src/zeroth/core/parallel/executor.py` (full) — split_fan_out, execute_branches, collect_fan_in
- `src/zeroth/core/parallel/errors.py` (full) — error hierarchy
- `src/zeroth/core/subgraph/executor.py` (full) — SubgraphExecutor.execute
- `src/zeroth/core/subgraph/resolver.py` (full) — namespace_subgraph, merge_governance
- `src/zeroth/core/subgraph/models.py` (full) — SubgraphNodeData
- `src/zeroth/core/graph/models.py:1-340` — NodeBase, SubgraphNode, Graph
- `src/zeroth/core/graph/validation.py` (full) — GraphValidator (never called in production)
- `src/zeroth/core/graph/repository.py` (full) — publish() has no validator wiring
- `src/zeroth/core/orchestrator/runtime.py` selected regions (lines 1-100, 140-270, 240-500, 490-740) — subgraph and parallel dispatch paths
- `src/zeroth/core/contracts/registry.py` (partial) — ContractVersion.json_schema
- `tests/subgraph/test_integration.py:1-180` — test stub patterns
- `.planning/phases/43-parallel-subgraph-fan-out-merge-strategies/43-CONTEXT.md` — 23 locked decisions

### Secondary
- `.planning/codebase/ARCHITECTURE.md` — graph domain lifecycle, orchestration layer
- `.planning/STATE.md` — project state confirmation

## Metadata

**Confidence breakdown:**
- Existing code structure: HIGH — all findings grounded in direct reads of current code
- Architecture recommendation: HIGH — follows established patterns, minimal-delta changes
- Open question resolutions (reducer signature, validator location, telemetry): HIGH — based on ecosystem convention + existing repo patterns
- Cost rollup plumbing (D-09): MEDIUM — partial grep verification; planner should spike before committing estimate
- Approval pause resume semantics: MEDIUM — D-11 wording has one plausible interpretation ("siblings re-run") that contradicts a literal reading; flagged in Assumptions Log A3

**Research date:** 2026-04-14
**Valid until:** 2026-05-14 (30 days; stable subsystem, no fast-moving upstream dependencies)
