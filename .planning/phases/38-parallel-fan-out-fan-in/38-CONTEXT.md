# Phase 38: Parallel Fan-Out / Fan-In - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Add parallel fan-out / fan-in execution to the orchestrator. A node can spawn N parallel execution branches that run concurrently (asyncio.gather), each with isolated execution context. A synchronization barrier collects all branch outputs into a deterministically ordered aggregated payload. Per-branch governance, audit, and cost tracking are maintained.

</domain>

<decisions>
## Implementation Decisions

### Fan-Out Model
- **D-01:** Fan-out is configured per-node via a `parallel_config: ParallelConfig | None` field on the node model. `ParallelConfig` specifies: `split_path` (str — dot-path to list in output), `merge_strategy` (str — "collect" or "reduce"), `fail_mode` (str — "fail_fast" or "best_effort"), `max_branches` (int | None — optional cap).
- **D-02:** When a node with `parallel_config` completes, the orchestrator splits its output list into N items and spawns N parallel branches. Each branch receives one item as its input payload.
- **D-03:** Branches execute via `asyncio.gather` with `return_exceptions=True` for best-effort mode. In fail-fast mode, first exception cancels remaining branches.

### Branch Isolation
- **D-04:** Each parallel branch gets its own isolated execution context: separate visit counts, separate audit trail, separate failure tracking. Implemented by creating a lightweight `BranchContext` that wraps Run state per branch.
- **D-05:** Branch visit counts do NOT count against the parent run's visit counts. Each branch has an independent counter starting from zero.
- **D-06:** ExecutionSettings guardrails (max_total_steps, max_visits_per_node) apply as sum across all branches — the orchestrator tracks total steps globally.

### Fan-In / Synchronization
- **D-07:** Synchronization barrier collects all branch outputs into a list, ordered by branch index (0, 1, 2, ...). The aggregated payload is placed at a configurable `merge_path` (default: same as `split_path`) on the next node's input.
- **D-08:** If a branch fails in best-effort mode, its slot in the result list is `None` (not omitted — preserves index alignment).

### Governance & Cost
- **D-09:** Policy enforcement and audit recording apply independently per branch. Each branch produces its own audit records linked to the parent run via `parent_run_id`.
- **D-10:** Cost attribution tracks per-branch spend. BudgetEnforcer is consulted before spawning with a pre-reservation of estimated cost across N branches.
- **D-11:** Contract validation applies per-branch — each branch's output is validated independently.

### Implementation Structure
- **D-12:** New `zeroth.core.parallel` package with models.py, executor.py, errors.py, __init__.py.
- **D-13:** The `_drive()` loop in orchestrator detects fan-out nodes and delegates to `ParallelExecutor.execute_fan_out()` instead of proceeding sequentially.

### Claude's Discretion
- BranchContext internal implementation details
- Whether to add a `branch_id` field to audit records or use metadata
- Error aggregation strategy for multi-branch failures
- Test approach for concurrent execution (deterministic vs timing-based)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Orchestrator Core
- `src/zeroth/core/orchestrator/runtime.py` — `_drive()` loop, `_dispatch_node()`, `_step()` — the core execution loop where fan-out integrates
- `src/zeroth/core/graph/models.py` — Node models, ExecutionSettings

### Cost & Budget
- `src/zeroth/core/econ/budget.py` — BudgetEnforcer — pre-reservation before spawning
- `src/zeroth/core/econ/cost.py` — Cost tracking per invocation

### Audit
- `src/zeroth/core/audit/models.py` — NodeAuditRecord — per-branch audit

### Policy
- `src/zeroth/core/policy/guard.py` — PolicyGuard — per-branch governance

### Conditions & Mappings
- `src/zeroth/core/conditions/branch.py` — Branch resolution (fan-out interacts with this)
- `src/zeroth/core/mappings/executor.py` — Mapping execution (fan-in aggregation may use transform mappings from Phase 33)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `asyncio.gather` — Core concurrency primitive for parallel branches
- `_dispatch_node()` — Existing node dispatch can be reused per-branch
- `BudgetEnforcer` — Pre-reservation API exists
- `NodeAuditRecord` — Per-branch audit records
- `Run.node_visit_counts` — Visit tracking (needs branch isolation)

### Established Patterns
- Pydantic ConfigDict(extra="forbid")
- Package structure: models.py, errors.py, __init__.py
- `_drive()` loop with step tracking and guardrails

### Integration Points
- `orchestrator/runtime.py` — Fan-out detection in `_drive()` loop
- `graph/models.py` — ParallelConfig on node model
- `econ/budget.py` — Pre-reservation before spawn
- `audit/models.py` — Branch-linked audit records

</code_context>

<specifics>
## Specific Ideas

STATE.md notes: "_drive() loop shared-state mutation hazard requires careful branch isolation design" and "Budget pre-reservation mechanics need specification during planning." These are the two highest-risk items.

</specifics>

<deferred>
## Deferred Ideas

- Distributed parallel execution across workers — explicitly out of scope per FUTURE-04 in REQUIREMENTS.md (in-process asyncio only)
- Dynamic branch count based on runtime conditions — future enhancement

</deferred>

---

*Phase: 38-parallel-fan-out-fan-in*
*Context gathered: 2026-04-13*
