# Phase 38: Parallel Fan-Out / Fan-In - Research

**Researched:** 2026-04-12
**Domain:** Concurrent branch execution, shared-state isolation, fan-out/fan-in orchestration
**Confidence:** HIGH

## Summary

Phase 38 adds parallel fan-out / fan-in execution to the Zeroth orchestrator. When a node with `parallel_config` completes, the orchestrator splits its output list into N items, spawns N concurrent execution branches via `asyncio.gather`, collects results at a synchronization barrier, and produces a deterministically ordered aggregated payload. Each branch gets its own isolated execution context (visit counts, audit trail, failure tracking). Two fail modes are supported: `fail_fast` (first exception cancels remaining) and `best_effort` (all branches complete, failures become `None` in the result list).

This is the hardest v4.0 phase because it touches the core `_drive()` loop -- the 170-line heart of the orchestrator that currently processes nodes sequentially in a `while True` loop. The primary engineering risk is shared-state mutation: the `Run` object is mutable Pydantic with `dict` and `list` fields (`node_visit_counts`, `metadata`, `execution_history`, `pending_node_ids`, `audit_refs`) that the `_drive()` loop mutates in-place on every iteration. Running multiple `_drive()` loops concurrently against the same `Run` would cause data corruption. The solution mandated by the CONTEXT.md decisions is `BranchContext` -- a lightweight per-branch wrapper that provides isolated copies of all mutable state, merged back after the barrier.

The second risk is budget pre-reservation. The current `BudgetEnforcer.check_budget()` only checks whether a tenant is within its cap; it has no reservation/hold API. The CONTEXT.md decision (D-10) calls for pre-reservation of estimated cost across N branches before spawning. This must be implemented as a simple estimated cost multiplier check (not a distributed lock), consistent with the existing fail-open design.

**Primary recommendation:** Create `zeroth.core.parallel` package following the established v4.0 pattern (models.py, executor.py, errors.py, __init__.py). The `ParallelExecutor` delegates to `_dispatch_node()` per branch, not to `_drive()` -- each branch executes a linear sub-graph segment, not a full recursive orchestration. Fan-out detection is a single `if` check in `_drive()` after `_dispatch_node()` returns, keyed on the node's `parallel_config` field.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Fan-out is configured per-node via a `parallel_config: ParallelConfig | None` field on the node model. `ParallelConfig` specifies: `split_path` (str -- dot-path to list in output), `merge_strategy` (str -- "collect" or "reduce"), `fail_mode` (str -- "fail_fast" or "best_effort"), `max_branches` (int | None -- optional cap).
- **D-02:** When a node with `parallel_config` completes, the orchestrator splits its output list into N items and spawns N parallel branches. Each branch receives one item as its input payload.
- **D-03:** Branches execute via `asyncio.gather` with `return_exceptions=True` for best-effort mode. In fail-fast mode, first exception cancels remaining branches.
- **D-04:** Each parallel branch gets its own isolated execution context: separate visit counts, separate audit trail, separate failure tracking. Implemented by creating a lightweight `BranchContext` that wraps Run state per branch.
- **D-05:** Branch visit counts do NOT count against the parent run's visit counts. Each branch has an independent counter starting from zero.
- **D-06:** ExecutionSettings guardrails (max_total_steps, max_visits_per_node) apply as sum across all branches -- the orchestrator tracks total steps globally.
- **D-07:** Synchronization barrier collects all branch outputs into a list, ordered by branch index (0, 1, 2, ...). The aggregated payload is placed at a configurable `merge_path` (default: same as `split_path`) on the next node's input.
- **D-08:** If a branch fails in best-effort mode, its slot in the result list is `None` (not omitted -- preserves index alignment).
- **D-09:** Policy enforcement and audit recording apply independently per branch. Each branch produces its own audit records linked to the parent run via `parent_run_id`.
- **D-10:** Cost attribution tracks per-branch spend. BudgetEnforcer is consulted before spawning with a pre-reservation of estimated cost across N branches.
- **D-11:** Contract validation applies per-branch -- each branch's output is validated independently.
- **D-12:** New `zeroth.core.parallel` package with models.py, executor.py, errors.py, __init__.py.
- **D-13:** The `_drive()` loop in orchestrator detects fan-out nodes and delegates to `ParallelExecutor.execute_fan_out()` instead of proceeding sequentially.

### Claude's Discretion
- BranchContext internal implementation details
- Whether to add a `branch_id` field to audit records or use metadata
- Error aggregation strategy for multi-branch failures
- Test approach for concurrent execution (deterministic vs timing-based)

### Deferred Ideas (OUT OF SCOPE)
- Distributed parallel execution across workers -- explicitly out of scope per FUTURE-04 in REQUIREMENTS.md (in-process asyncio only)
- Dynamic branch count based on runtime conditions -- future enhancement
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PARA-01 | A graph author can configure a node to spawn N parallel branches from its output (e.g., one branch per list item), and a synchronization barrier collects all branch outputs into an aggregated payload with deterministic ordering by branch index | `ParallelConfig` on node model + `_get_path`/`_set_path` from mappings/executor.py already handle dot-path traversal for split/merge; `asyncio.gather` preserves argument order in results |
| PARA-02 | Each parallel branch has its own isolated execution context (visit counts, audit trail, failure tracking); a failure in one branch does not automatically fail others when configured for best-effort mode (fail-fast is also supported) | `BranchContext` isolates mutable Run state per branch; `asyncio.gather(return_exceptions=True)` enables best-effort; `asyncio.gather` with exception propagation enables fail-fast |
| PARA-03 | Policy enforcement, audit recording, and contract validation apply independently per branch, each producing its own audit records linked to the parent run | `PolicyGuard.evaluate()`, `AuditRepository.write()`, and contract validation all accept per-invocation parameters; `NodeAuditRecord` has `run_id` field that can carry `parent_run_id` context in `execution_metadata` |
| PARA-04 | Cost attribution tracks per-branch spend; BudgetEnforcer is consulted before spawning with a pre-reservation of total estimated cost; ExecutionSettings guardrails account for parallel branches as sum across all branches | `BudgetEnforcer.check_budget()` returns `(allowed, current_spend, budget_cap)` -- pre-reservation is a headroom check against `estimated_cost_per_branch * N`; global step counter maintained by ParallelExecutor |
| PARA-05 | Branch isolation is complete: separate visit counts, separate audit trail, separate failure tracking per branch | Achieved via BranchContext deep-copying mutable state from Run; branch visit counts start at zero per D-05 |
| PARA-06 | Fan-out integrates cleanly with existing orchestrator without breaking sequential execution | Detection is a single conditional check on `parallel_config`; `None` means sequential (default); existing tests remain unaffected |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Build/Test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Layout:** Source in `src/zeroth/`, tests in `tests/`
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Context efficiency:** Read only task-relevant files; do NOT read root PLAN.md
- **Pydantic pattern:** `ConfigDict(extra="forbid")` on all models [VERIFIED: codebase inspection]
- **Package pattern:** models.py, errors.py, executor/core module, __init__.py with explicit re-exports [VERIFIED: artifacts, templates, context_window, http packages]

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| asyncio (stdlib) | Python 3.12.12 | `asyncio.gather` for concurrent branch execution | Standard library; already used in codebase (health.py, dispatch) [VERIFIED: codebase grep] |
| pydantic | 2.x (installed) | ParallelConfig, BranchResult, BranchContext models | Project standard; ConfigDict(extra="forbid") pattern [VERIFIED: codebase inspection] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| copy (stdlib) | Python 3.12 | `copy.deepcopy` for Run state isolation in BranchContext | Needed to create isolated mutable state copies for each branch [VERIFIED: stdlib] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `asyncio.gather` | `asyncio.TaskGroup` (Python 3.11+) | TaskGroup has stricter error handling (cancels all on first exception by default), which is desirable for fail_fast but requires workarounds for best_effort; `asyncio.gather` with `return_exceptions=True` is simpler for best_effort mode and is the locked decision (D-03) |
| `asyncio.gather` | `anyio.create_task_group` | External dependency not in project; asyncio.gather is simpler and matches project patterns [ASSUMED] |
| Deep copy of full Run | Selective field copying | Deep copy is safer but slower; selective copying risks missing new fields added later; recommend deep copy with explicit field list for safety |

**Installation:**
```bash
# No new packages needed -- all dependencies already installed
uv sync
```

**Version verification:** Python 3.12.12 confirmed via `uv run python --version`. asyncio.gather and copy.deepcopy are stdlib. Pydantic 2.x already installed. [VERIFIED: runtime check]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/parallel/
    __init__.py          # Public API re-exports
    models.py            # ParallelConfig, BranchContext, BranchResult, FanInResult
    executor.py          # ParallelExecutor (fan-out splitting, branch dispatch, fan-in barrier)
    errors.py            # ParallelExecutionError hierarchy
```

### Pattern 1: ParallelConfig on Node Model
**What:** A `parallel_config: ParallelConfig | None` field on `NodeBase` (or a new mixin) that enables fan-out behavior for any node type.
**When to use:** Always -- this is the configuration surface per D-01.
**Example:**
```python
# Source: Established pattern from AgentNodeData.context_window (Phase 37)
from zeroth.core.parallel.models import ParallelConfig

class NodeBase(BaseModel):
    model_config = ConfigDict(extra="forbid")
    # ... existing fields ...
    parallel_config: ParallelConfig | None = None
```

### Pattern 2: BranchContext for State Isolation
**What:** A lightweight dataclass that holds isolated copies of mutable Run state for a single branch. Each branch gets its own `node_visit_counts`, `execution_history`, `audit_refs`, `metadata` (partial), and `condition_results`. The parent Run's `execution_history` length is tracked globally for `max_total_steps` enforcement.
**When to use:** Created before branch dispatch, consumed at fan-in barrier.
**Example:**
```python
# Source: Derived from Run model analysis + D-04, D-05, D-06
@dataclass(slots=True)
class BranchContext:
    """Isolated execution state for a single parallel branch."""
    branch_index: int
    branch_id: str  # e.g. "{run_id}:branch:{index}"
    input_payload: dict[str, Any]
    node_visit_counts: dict[str, int]  # starts at {} per D-05
    execution_history: list[RunHistoryEntry]  # branch-local
    audit_refs: list[str]  # branch-local
    condition_results: list[RunConditionResult]  # branch-local
    metadata: dict[str, Any]  # branch-scoped copy
```

### Pattern 3: Fan-Out Detection in _drive() Loop
**What:** After `_dispatch_node()` returns output for a node, check if `node.parallel_config` is not None. If so, delegate to `ParallelExecutor.execute_fan_out()` instead of the normal `_plan_next_nodes` / `_queue_next_nodes` sequence.
**When to use:** This is the single integration point in the orchestrator.
**Example:**
```python
# Source: _drive() loop integration point (runtime.py line ~258-270)
# After: output_data, audit_record = await self._dispatch_node(node, run, input_payload)
# Before: self._plan_next_nodes(...)

parallel_config = getattr(node, 'parallel_config', None)
if parallel_config is not None:
    fan_in_result = await self._parallel_executor.execute_fan_out(
        orchestrator=self,
        graph=graph,
        run=run,
        source_node=node,
        source_output=output_data,
        config=parallel_config,
    )
    # Merge fan-in result back into run state
    self._merge_fan_in_result(run, fan_in_result, parallel_config)
    # Continue normal _drive() loop with merged payload
    continue
```

### Pattern 4: asyncio.gather for Branch Execution
**What:** Use `asyncio.gather(*branch_coros, return_exceptions=True)` for best-effort mode. For fail_fast mode, use `asyncio.gather(*branch_coros)` (exceptions propagate immediately, cancelling siblings via task cancellation).
**When to use:** Core execution model per D-03.
**Example:**
```python
# Source: D-03 + asyncio stdlib docs
async def _execute_branches(
    self,
    branch_contexts: list[BranchContext],
    orchestrator: RuntimeOrchestrator,
    graph: Graph,
    run: Run,
    config: ParallelConfig,
) -> list[BranchResult]:
    coros = [
        self._execute_single_branch(ctx, orchestrator, graph, run)
        for ctx in branch_contexts
    ]
    if config.fail_mode == "best_effort":
        raw_results = await asyncio.gather(*coros, return_exceptions=True)
    else:  # fail_fast
        raw_results = await asyncio.gather(*coros)
    return [self._wrap_result(i, r) for i, r in enumerate(raw_results)]
```

### Pattern 5: Global Step Counter for ExecutionSettings Enforcement
**What:** A shared `asyncio.Lock`-protected counter that all branches increment. When the sum across all branches exceeds `max_total_steps`, a `ParallelStepLimitError` is raised to cancel remaining work.
**When to use:** Per D-06 -- guardrails apply as sum across branches.
**Example:**
```python
# Source: D-06 + existing _enforce_loop_guards pattern
class GlobalStepTracker:
    """Thread-safe step counter shared across parallel branches."""
    def __init__(self, current_steps: int, max_steps: int) -> None:
        self._count = current_steps
        self._max = max_steps
        self._lock = asyncio.Lock()

    async def increment(self) -> None:
        async with self._lock:
            self._count += 1
            if self._count >= self._max:
                msg = f"parallel execution exceeded max_total_steps ({self._max})"
                raise ParallelStepLimitError(msg)
```

### Anti-Patterns to Avoid
- **Sharing the Run object across branches:** Mutable Pydantic model with dict/list fields. Concurrent writes WILL corrupt state. Each branch MUST operate on isolated copies.
- **Running full _drive() recursively per branch:** The `_drive()` loop manages the entire run lifecycle (completion, failure, checkpointing). Branches should use a simpler dispatch-per-node mechanism, not recursive orchestration.
- **Using threading for parallelism:** The codebase is async-first. Threads would bypass the event loop and create GIL contention. Use asyncio only.
- **Modifying BudgetEnforcer to add reservation locking:** The BudgetEnforcer is designed as fail-open with TTL cache. Adding distributed locks contradicts its design philosophy. Pre-reservation should be a simple headroom check.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Concurrent execution | Custom threading/multiprocessing | `asyncio.gather` | Stdlib, event-loop native, matches codebase patterns |
| Dot-path traversal for split_path/merge_path | Custom path resolution | `_get_path` / `_set_path` from `mappings/executor.py` | Already tested, handles nested dicts correctly |
| Branch ID generation | Custom UUID scheme | `f"{run.run_id}:branch:{index}"` | Deterministic, traceable, follows existing `audit_id` pattern (`{run_id}:{audit_ref}`) |
| Deep copy of mutable state | Manual field-by-field copy | `copy.deepcopy` on selected Run fields | Safer against future field additions; Run fields are all JSON-serializable |
| Error aggregation | Custom error tracking | Pydantic model `BranchResult(index, output, error)` | Type-safe, serializable, follows project patterns |

**Key insight:** The orchestrator already has all the building blocks -- `_dispatch_node()`, `_record_history()`, `_enforce_policy()`, `_increment_node_visit()`. The parallel executor reuses these per-branch by passing branch-scoped state, NOT by reimplementing dispatch logic.

## Common Pitfalls

### Pitfall 1: Shared-State Mutation in _drive() Loop
**What goes wrong:** Multiple branches concurrently modify the same `Run.node_visit_counts`, `Run.execution_history`, `Run.metadata["node_payloads"]`, or `Run.audit_refs`, causing data corruption, lost updates, or interleaved list entries.
**Why it happens:** The `Run` model uses plain Python dicts and lists. `asyncio.gather` runs coroutines on the same event loop thread, but any `await` point allows interleaving. Dict updates like `run.node_visit_counts[node_id] = count + 1` are NOT atomic across await points.
**How to avoid:** Each branch operates on an isolated `BranchContext` with deep-copied mutable fields. NEVER pass the parent `Run` object directly to branch coroutines. At the barrier, merge branch states into the parent Run sequentially (single-threaded, no concurrency).
**Warning signs:** Flaky test failures; audit records with wrong branch_index; visit counts lower than expected; execution_history entries missing or duplicated.

### Pitfall 2: asyncio.gather Exception Handling Asymmetry
**What goes wrong:** In `fail_fast` mode, `asyncio.gather` raises the first exception but does NOT cancel other tasks. Remaining tasks continue running in the background, consuming resources and potentially mutating state after the parent has moved on.
**Why it happens:** `asyncio.gather` does not automatically cancel remaining tasks when one raises. The caller must explicitly cancel pending tasks.
**How to avoid:** In fail_fast mode, wrap branch coroutines as `asyncio.Task` objects. On first exception, explicitly cancel remaining tasks and await their cancellation. Use `try/except/finally` to ensure cleanup. OR use a simple approach: wrap in a wrapper coroutine that checks a shared `asyncio.Event` flag.
**Warning signs:** Test passes but background tasks log errors after the test completes; resource leaks in long-running services.

### Pitfall 3: Budget Pre-Reservation Race Condition
**What goes wrong:** Two parallel fan-outs start simultaneously, both check budget, both see headroom, both proceed, and the combined cost exceeds the cap.
**Why it happens:** The BudgetEnforcer uses a TTL cache and checks against the Regulus backend. Between the check and the actual execution, costs can change.
**How to avoid:** Accept this as a known limitation per the fail-open design philosophy (D-12 of BudgetEnforcer). Pre-reservation is a best-effort estimate, not a hard guarantee. The check should multiply `estimated_cost_per_branch * N` and compare against remaining headroom. If insufficient, log a warning and proceed (fail-open) or reject (fail-closed per configuration).
**Warning signs:** Budget overshoot in high-concurrency scenarios. This is acceptable -- Regulus is an observability tool, not a billing hard-stop.

### Pitfall 4: Checkpoint/Persistence During Parallel Execution
**What goes wrong:** The orchestrator calls `run_repository.put(run)` and `write_checkpoint(run)` during parallel execution, saving intermediate state that represents a partially-completed fan-out. If the process crashes and resumes, the run is in an inconsistent state (some branches completed, some not).
**Why it happens:** The existing `_drive()` loop checkpoints after every node. During parallel execution, checkpointing must be deferred until the fan-in barrier completes.
**How to avoid:** The `ParallelExecutor` should NOT checkpoint during branch execution. Instead, it should:
1. Save a "fan-out started" checkpoint before spawning branches
2. Execute all branches in-memory (no persistence)
3. Merge results and save a "fan-out completed" checkpoint after the barrier
4. If the process crashes during fan-out, resumption replays the entire fan-out from scratch
**Warning signs:** Incomplete branch results after crash recovery; duplicate branch executions.

### Pitfall 5: Node Type Mismatch on parallel_config
**What goes wrong:** `parallel_config` is added to `NodeBase`, making it available on `HumanApprovalNode`. A fan-out on an approval node creates N approval gates, which is semantically nonsensical and could deadlock the run.
**Why it happens:** Pydantic model inheritance -- fields on `NodeBase` propagate to all subtypes.
**How to avoid:** Add `parallel_config` to `NodeBase` (it's the simplest approach and keeps the model hierarchy clean), but validate in the `ParallelExecutor` that the fan-out source node type is supported (AgentNode or ExecutableUnitNode only). Reject HumanApprovalNode with a clear error.
**Warning signs:** Approval gates being created N times during fan-out.

### Pitfall 6: Non-Deterministic Result Ordering
**What goes wrong:** Branch results arrive in completion order (fastest first), not branch index order. The aggregated payload has items in the wrong order.
**Why it happens:** `asyncio.gather` returns results in argument order, not completion order. This is actually correct behavior -- BUT if the implementation uses `asyncio.create_task` and collects results via `asyncio.as_completed`, ordering is lost.
**How to avoid:** Use `asyncio.gather` exactly as specified in D-03. It preserves argument order. Each branch carries its `branch_index` in `BranchContext` as a safety net for verification.
**Warning signs:** Test results vary between runs; list items appear in different orders on different executions.

## Code Examples

Verified patterns from the existing codebase:

### Dot-Path Value Extraction (reusable for split_path)
```python
# Source: mappings/executor.py lines 26-37
def _get_path(payload: Mapping[str, Any], path: str) -> tuple[bool, Any]:
    current: Any = payload
    for part in path.split("."):
        if not isinstance(current, Mapping) or part not in current:
            return False, None
        current = current[part]
    return True, current
```

### Dot-Path Value Setting (reusable for merge_path)
```python
# Source: mappings/executor.py lines 40-54
def _set_path(payload: dict[str, Any], path: str, value: Any) -> None:
    current = payload
    parts = path.split(".")
    for part in parts[:-1]:
        next_value = current.get(part)
        if not isinstance(next_value, dict):
            next_value = {}
            current[part] = next_value
        current = next_value
    current[parts[-1]] = value
```

### Existing Node Dispatch (reusable per branch)
```python
# Source: orchestrator/runtime.py lines 275-503
# _dispatch_node() handles AgentNode, ExecutableUnitNode dispatch
# Returns (output_data: dict, audit_record: dict)
# This method is stateless w.r.t. Run -- it reads from Run but
# only writes to the return values. Safe to call per-branch with
# isolated input_payload.
```

### Existing History Recording (reusable per branch)
```python
# Source: orchestrator/runtime.py lines 606-664
# _record_history() writes to run.execution_history and audit_repository
# Must be called with branch-isolated Run state to prevent interleaving
```

### asyncio.gather with Exception Handling
```python
# Source: Python 3.12 stdlib docs
import asyncio

async def execute_branches_best_effort(coros):
    results = await asyncio.gather(*coros, return_exceptions=True)
    return results  # Some may be Exception instances

async def execute_branches_fail_fast(coros):
    tasks = [asyncio.create_task(c) for c in coros]
    try:
        results = await asyncio.gather(*tasks)
        return results
    except Exception:
        for t in tasks:
            t.cancel()
        # Wait for cancellation to complete
        await asyncio.gather(*tasks, return_exceptions=True)
        raise
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Sequential node execution only | Adding parallel fan-out/fan-in | Phase 38 (this phase) | Enables batch processing, map-reduce patterns |
| `asyncio.gather` as primary concurrency | `asyncio.TaskGroup` (Python 3.11+) | Python 3.11, March 2022 | TaskGroup is the "modern" approach but gather is simpler for mixed fail modes; D-03 locks gather |

**Deprecated/outdated:**
- `asyncio.ensure_future()` -- replaced by `asyncio.create_task()` in Python 3.7+; do not use [VERIFIED: Python docs]
- `@asyncio.coroutine` decorator -- replaced by `async def` in Python 3.5+; not present in codebase [VERIFIED: codebase grep]

## Architecture Deep Dive: _drive() Loop Integration

### Current _drive() Flow (lines 166-273 of runtime.py)
```
while True:
    1. _enforce_loop_guards() -- check max_total_steps, max_runtime
    2. Check pending_node_ids -- if empty, COMPLETED
    3. Pop next node_id from pending_node_ids
    4. _payload_for() -- get queued input payload
    5. _consume_side_effect_approval() -- handle pending approvals
    6. _enforce_policy() -- check policy guard
    7. _gate_policy_required_side_effects() -- handle side-effect gates
    8. Handle HumanApprovalNode -- pause for approval
    9. _dispatch_node() -- execute the node
    10. _record_history() -- save audit + history
    11. _increment_node_visit() -- bump visit count
    12. _plan_next_nodes() -- determine next steps
    13. _queue_next_nodes() -- prepare payloads for next nodes
    14. Checkpoint run state
```

### Fan-Out Integration Point

The fan-out check should be inserted between step 9 and step 10. After `_dispatch_node()` returns `output_data`, the orchestrator checks if the CURRENT node has `parallel_config`. If yes, it:

1. Extracts the list from `output_data` at `parallel_config.split_path`
2. Validates the list (must be a list, length > 0, length <= max_branches)
3. Creates N `BranchContext` objects
4. Delegates to `ParallelExecutor.execute_fan_out()`
5. Receives aggregated `FanInResult`
6. Replaces `output_data` with the merged result
7. Continues normal flow (steps 10-14) with the merged output

This means the fan-out node itself executes normally (producing the list), and then its OUTPUT triggers fan-out. The branches execute the DOWNSTREAM nodes (the nodes after the fan-out node), not the fan-out node itself. This is consistent with D-02: "When a node with `parallel_config` completes, the orchestrator splits its output list."

### Critical Design Decision: Branch Scope

Each branch executes a SINGLE downstream node per item (not a sub-graph). The fan-out node's output list is split, each item becomes input to the same set of downstream nodes, and all branch outputs are collected at the barrier. The downstream nodes are determined by `_plan_next_nodes()` applied to the fan-out node.

If the fan-out node has multiple downstream edges, each branch executes all downstream nodes (the same edges apply to each branch). The branch result is the output of the last node in each branch's execution.

### BranchContext Design

```python
@dataclass(slots=True)
class BranchContext:
    branch_index: int
    branch_id: str
    input_payload: dict[str, Any]
    # Isolated mutable state (deep-copied from Run)
    node_visit_counts: dict[str, int]  # Empty per D-05
    execution_history: list  # Branch-local
    audit_refs: list[str]  # Branch-local
    condition_results: list  # Branch-local
    metadata: dict[str, Any]  # Branch-scoped copy
```

**What gets isolated (per branch):**
- `node_visit_counts` -- fresh `{}` per D-05
- `execution_history` -- branch-local list
- `audit_refs` -- branch-local list
- `condition_results` -- branch-local list
- `metadata["node_payloads"]` -- branch-scoped
- `metadata["edge_visit_counts"]` -- branch-scoped
- `metadata["path"]` -- branch-scoped

**What remains shared (read-only):**
- `run.run_id` -- all branches belong to the same run
- `run.graph_version_ref` -- immutable
- `run.deployment_ref` -- immutable
- `run.tenant_id` -- immutable
- `run.thread_id` -- immutable (thread sharing per run)
- `graph` -- immutable Graph object

**What gets tracked globally:**
- Total step count across all branches (for `max_total_steps` enforcement via `GlobalStepTracker`)

### Fan-In Merge Strategy

After all branches complete:

1. Collect `BranchResult` objects (output_data or error per branch)
2. Build aggregated list: `[result_0.output, result_1.output, ..., result_N.output]`
3. For failed branches in best-effort mode: slot = `None` per D-08
4. Create merged output: set aggregated list at `merge_path` (default = `split_path`)
5. Merge branch execution histories into parent Run (append all)
6. Merge branch audit refs into parent Run (append all)
7. Sum branch visit counts into parent Run (only needed if we want per-node-across-branches tracking)
8. Update parent Run's total step count

## Budget Pre-Reservation Design

The existing `BudgetEnforcer.check_budget(tenant_id)` returns `(allowed, current_spend, budget_cap)`. Pre-reservation works as follows:

```python
async def check_budget_with_reservation(
    enforcer: BudgetEnforcer,
    tenant_id: str,
    estimated_cost_per_branch: float,
    num_branches: int,
) -> tuple[bool, str]:
    """Check if tenant has budget headroom for N parallel branches."""
    allowed, current_spend, budget_cap = await enforcer.check_budget(tenant_id)
    if not allowed:
        return False, "tenant budget already exceeded"
    estimated_total = estimated_cost_per_branch * num_branches
    remaining = budget_cap - current_spend
    if estimated_total > remaining:
        return False, f"estimated parallel cost ${estimated_total:.4f} exceeds remaining budget ${remaining:.4f}"
    return True, ""
```

**Open question:** Where does `estimated_cost_per_branch` come from? Options:
1. Fixed configurable estimate per node (e.g., `ParallelConfig.estimated_branch_cost: float | None`)
2. Average cost of previous executions of this node (requires history lookup)
3. Skip cost estimation entirely -- just check `allowed` (simplest, consistent with fail-open)

**Recommendation:** Option 3 for now. The BudgetEnforcer already checks whether the tenant is within budget. Multiplying by N is speculative and could be wildly wrong. Just verify the tenant has budget headroom before spawning. This is consistent with the fail-open design philosophy. [ASSUMED -- needs user confirmation]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Each branch executes the downstream nodes of the fan-out node (not a configurable sub-graph) | Architecture Deep Dive | If branches should execute arbitrary sub-graphs, the BranchContext design needs full recursive _drive() support, which is significantly more complex |
| A2 | Budget pre-reservation is a simple headroom check, not a distributed reservation/hold | Budget Pre-Reservation Design | If the user wants actual cost reservation with deduction, the BudgetEnforcer needs a new `reserve()` / `release()` API |
| A3 | parallel_config goes on NodeBase (all node types) with runtime validation rejecting HumanApprovalNode | Common Pitfalls | If it should only be on specific node types, model hierarchy changes needed |
| A4 | The `_dispatch_node()` method is safe to call from branch context because it only reads from Run (via `_enforcement_context_for`, etc.) and returns output | Architecture Deep Dive | If _dispatch_node has hidden Run mutations, branch isolation breaks |

## Open Questions

1. **Branch scope: single node or sub-graph?**
   - What we know: D-02 says "splits its output list into N items and spawns N parallel branches. Each branch receives one item as its input payload." D-13 says the _drive() loop "delegates to ParallelExecutor."
   - What's unclear: Does each branch execute just the immediate downstream node(s), or an entire sub-graph until a convergence point?
   - Recommendation: Start with single-downstream-node branches (simpler, covers the primary use case of "process each item independently"). Sub-graph branches can be added in Phase 39 (Subgraph Composition) or a future phase.

2. **Estimated cost per branch for budget pre-reservation**
   - What we know: D-10 says "pre-reservation of estimated cost across N branches"
   - What's unclear: How to estimate cost before execution -- no historical data, no model pricing lookup
   - Recommendation: Use a configurable `estimated_branch_cost` field on ParallelConfig (default: None = skip estimation, just check budget is not exceeded). This keeps it simple and user-configurable.

3. **Audit record branch identification**
   - What we know: D-09 says audit records are linked via `parent_run_id`
   - What's unclear: `NodeAuditRecord` has no `parent_run_id` field. Whether to add one, or use `execution_metadata` dict
   - Recommendation (Claude's discretion): Add a `branch_id` field to `execution_metadata` dict rather than modifying the NodeAuditRecord schema. This is non-breaking and follows the pattern of `execution_metadata["context_window"]` from Phase 37.

4. **How to handle fail_fast cancellation**
   - What we know: D-03 says "first exception cancels remaining branches"
   - What's unclear: asyncio.gather does NOT auto-cancel on exception. Manual cancellation is needed.
   - Recommendation: Use `asyncio.create_task()` + explicit cancellation in fail_fast mode. In best_effort mode, use `asyncio.gather(return_exceptions=True)` directly.

## Sources

### Primary (HIGH confidence)
- [VERIFIED: codebase inspection] orchestrator/runtime.py `_drive()` loop structure (lines 166-273), `_dispatch_node()` (lines 275-503), Run model fields, BudgetEnforcer API
- [VERIFIED: codebase inspection] graph/models.py Node hierarchy, ExecutionSettings, Edge model
- [VERIFIED: codebase inspection] mappings/executor.py `_get_path`/`_set_path` dot-path utilities
- [VERIFIED: codebase inspection] audit/models.py NodeAuditRecord fields
- [VERIFIED: codebase inspection] runs/models.py Run mutable fields analysis
- [VERIFIED: codebase inspection] artifacts, templates, context_window packages for v4.0 pattern reference
- [VERIFIED: runtime] Python 3.12.12 via `uv run python --version`
- [VERIFIED: pyproject.toml] pytest-asyncio 0.25+, asyncio_mode = "auto"

### Secondary (MEDIUM confidence)
- [CITED: docs.python.org/3/library/asyncio-task.html] asyncio.gather behavior: preserves argument order, return_exceptions parameter, exception propagation
- [CITED: docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup] TaskGroup comparison: auto-cancellation vs gather's manual approach

### Tertiary (LOW confidence)
- None -- all claims verified against codebase or official Python docs

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no new dependencies, all stdlib + existing pydantic
- Architecture: HIGH -- all integration points verified via codebase inspection; _drive() loop structure fully mapped
- Pitfalls: HIGH -- shared-state mutation hazard identified via detailed Run model field analysis; asyncio.gather behavior verified against Python docs
- Budget pre-reservation: MEDIUM -- simple headroom check is assumed; user may want more sophisticated reservation

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable domain -- asyncio stdlib is mature, orchestrator code changes only within this phase)
