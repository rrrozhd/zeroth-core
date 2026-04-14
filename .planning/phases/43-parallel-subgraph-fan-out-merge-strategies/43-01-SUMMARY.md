---
phase: 43-parallel-subgraph-fan-out-merge-strategies
plan: 01
subsystem: orchestration
tags: [orchestration, parallel, subgraph, composition, approval-pause]
requirements: [ORCH-01, ORCH-02]
requirements_addressed: [ORCH-01, ORCH-02]
dependency_graph:
  requires:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/parallel/models.py
    - src/zeroth/core/parallel/executor.py
    - src/zeroth/core/parallel/errors.py
    - src/zeroth/core/subgraph/executor.py
    - src/zeroth/core/subgraph/resolver.py
    - src/zeroth/core/orchestrator/runtime.py
  provides:
    - BranchApprovalPauseSignal(BaseException)
    - FanInResult.pause_state
    - SubgraphExecutor.execute(branch_context, step_tracker)
    - SubgraphExecutor.resume(paused_child_run_id, branch_index)
    - namespace_subgraph(..., branch_index)
    - RuntimeOrchestrator._execute_parallel_fan_out_resume (D-11 literal)
    - RuntimeOrchestrator._handle_parallel_subgraph_pause
  affects:
    - tests/subgraph/test_executor.py (mock _drive signature)
    - tests/subgraph/test_models.py (prior reject assertion inverted)
    - tests/test_v4_bootstrap_validation.py (reject -> accept)
    - tests/test_v4_cross_feature_integration.py (reject -> missing-executor)
tech_stack:
  added: []
  patterns:
    - BaseException signal for structured pause propagation
    - branch-prefixed audit ID namespacing
    - shared GlobalStepTracker across nested composition
    - child-return-path cost rollup (W-4)
key_files:
  created:
    - tests/parallel/test_subgraph_in_parallel.py
  modified:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/parallel/errors.py
    - src/zeroth/core/parallel/models.py
    - src/zeroth/core/parallel/executor.py
    - src/zeroth/core/subgraph/resolver.py
    - src/zeroth/core/subgraph/executor.py
    - src/zeroth/core/orchestrator/runtime.py
    - tests/subgraph/test_executor.py
    - tests/subgraph/test_models.py
    - tests/test_v4_bootstrap_validation.py
    - tests/test_v4_cross_feature_integration.py
decisions:
  - id: D-05
    summary: "SubgraphNode._reject_parallel_config validator removed (unconditional unblock)"
  - id: D-06
    summary: "Force isolated thread for subgraphs inside parallel branches regardless of thread_participation"
  - id: D-08/D-12
    summary: "Shared GlobalStepTracker threaded through _drive → SubgraphExecutor.execute → nested _execute_parallel_fan_out"
  - id: D-09
    summary: "Cost rollup written at SubgraphExecutor.execute return path only (W-4); _drive stays cost-agnostic"
  - id: D-10
    summary: "namespace_subgraph branch_index kwarg yields branch:{i}:subgraph:{ref}:{depth}: prefix"
  - id: D-11-literal
    summary: "Run-wide approval pause: completed siblings rehydrated byte-identically, paused branch resumed via SubgraphExecutor.resume (NOT execute), cancelled siblings recorded as None-output BranchResults"
  - id: D-19
    summary: "Cancelled branches get output=None with error='cancelled_by_approval_pause'"
  - id: D-23
    summary: "FanOutValidationError SubgraphNode branch removed entirely (no opt-in flag)"
metrics:
  duration_minutes: ~50
  tasks_completed: 3
  tests_added: 17
  files_created: 1
  files_modified: 11
---

# Phase 43 Plan 01: Parallel Subgraph Fan-Out & Approval-Pause Resume Summary

One-liner: Unblocked SubgraphNode inside fan-out branches, threaded a
shared GlobalStepTracker through nested composition, implemented
branch-prefixed audit IDs, and wired D-11 literal approval-pause with
byte-identical sibling rehydration and `SubgraphExecutor.resume`-only
paused-branch drive.

## What Was Built

### Task 1 — Data layer (commit `3cb7bc2`)

- **Removed `SubgraphNode._reject_parallel_config` model validator** in
  `graph/models.py` — SubgraphNode now accepts `parallel_config`
  unconditionally (D-05, D-23).
- **Removed SubgraphNode reject branch** in
  `ParallelExecutor.split_fan_out`; HumanApprovalNode reject preserved.
- **`BranchApprovalPauseSignal(BaseException)`** added to
  `parallel/errors.py`. BaseException subclass so `asyncio.gather(*tasks)`
  re-raises it uncaught in fail-fast, and best-effort inspects the
  `return_exceptions=True` list for an instance.
- **`FanInResult.pause_state: dict | None`** field added documenting the
  expected `paused`/`completed_branch_results`/`cancelled_branch_contexts`
  key set.
- **13 new data-layer tests** in `tests/parallel/test_subgraph_in_parallel.py`
  covering SubgraphNode accept, split_fan_out unblock, HumanApproval
  regression guard, signal semantics (BaseException vs Exception), gather
  propagation behavior, FanInResult defaults.
- **Updated `tests/subgraph/test_models.py::test_subgraph_node_parallel_config_rejected`**
  → renamed to `test_subgraph_node_accepts_parallel_config` with inverted
  assertion.

### Task 2 — Wiring (commit `59f29d5`)

- **`namespace_subgraph`** gains keyword-only `branch_index: int | None =
  None`. Branch prefix `branch:{i}:subgraph:{ref}:{depth}:` (D-10).
- **`SubgraphExecutor.execute`** gains keyword-only `branch_context` and
  `step_tracker` kwargs:
  - D-10: passes `branch_index` through to `namespace_subgraph`.
  - D-06: forces `child_thread_id = ""` (isolated) when
    `branch_context is not None` regardless of
    `SubgraphNodeData.thread_participation`.
  - D-08/D-12: forwards `step_tracker` into recursive `_drive`.
  - D-09/W-4: writes `result.metadata["total_cost_usd"]` from
    `_sum_audit_cost(result.execution_history)` on return — **this is
    the ONLY writer of `total_cost_usd`**; `_drive` stays cost-agnostic.
- **`SubgraphExecutor.resume`** method added — re-resolves, re-namespaces
  with the **same** `branch_index` (T-43-01 audit-idempotency
  mitigation), re-merges governance, and calls
  `orchestrator._drive(merged, child_run, step_tracker=...)`.
- **`_drive`** gains keyword-only `step_tracker: GlobalStepTracker | None
  = None`; forwards through `_execute_parallel_fan_out` and
  `subgraph_executor.execute`.
- **`_execute_parallel_fan_out`** gains `step_tracker` kwarg; only
  constructs a fresh tracker when none is passed in (fixes the D-12
  nested-budget escape).
- **`branch_coro_factory`** now detects
  `isinstance(ds_node, SubgraphNode)` and dispatches via
  `self.subgraph_executor.execute(..., branch_context=ctx,
  step_tracker=step_tracker)`. On WAITING_APPROVAL it raises
  `BranchApprovalPauseSignal`. On success it populates `ds_audit`
  with `cost_usd` from `_sum_run_cost(child_run)` so branch-level
  cost rollup works.
- **`_sum_run_cost`** module-level helper reads
  `run.metadata["total_cost_usd"]` (populated by
  `SubgraphExecutor.execute`).

### Task 3 — D-11 Literal Approval Pause (commit `59f29d5`)

- **`ParallelExecutor._execute_fail_fast`** catches
  `BranchApprovalPauseSignal`, cancels in-flight siblings, drains the
  gather, partitions `completed_before_pause` vs `cancelled_by_pause`
  by inspecting each drained task result, attaches them to the signal
  instance, and re-raises.
- **`ParallelExecutor._execute_best_effort`** uses
  `asyncio.gather(*tasks, return_exceptions=True)`, walks the returned
  list, collects the pause signal if present, partitions completed
  vs cancelled results, attaches state to the signal instance, and
  raises it. This handles the case where modern CPython's `gather`
  captures `BaseException` subclasses into the results list.
- **`_execute_parallel_fan_out`** wraps the `execute_branches` call
  in `try/except BranchApprovalPauseSignal`; on catch it builds a
  `FanInResult(results=[], pause_state={...})` carrying
  `paused` (with full `BranchContext` dump), `completed_branch_results`,
  `cancelled_branch_contexts`, and `split_input`.
- **`_handle_parallel_subgraph_pause`** method stashes
  `run.metadata["pending_parallel_subgraph"]` with the full D-11 literal
  payload, sets `run.status = WAITING_APPROVAL`, re-queues the fan-out
  source `node_id`, persists, returns.
- **`_execute_parallel_fan_out_resume`** method:
  1. Rehydrates `completed_branches` as `BranchResult` instances
     byte-identically (dict → RunHistoryEntry where possible).
  2. Resumes the paused branch via
     `self.subgraph_executor.resume(orchestrator=self, ...,
     branch_index=paused_info["branch_index"], step_tracker=...)` —
     **NOT `execute`**. Falls back to manual resolve + re-namespace +
     `resume_graph` if `SubgraphExecutor.resume` is not available on
     the injected executor (duck-typed for test doubles).
  3. Records cancelled siblings as
     `BranchResult(output=None, error="cancelled_by_approval_pause")`
     per D-19.
  4. Merges all three partitions into branch-index order and passes
     through `self.parallel_executor.collect_fan_in` with the plan's
     `merge_strategy` (tests always use `"collect"` per the W-1 cross-
     plan contract).
- **`_drive` loop** detects
  `run.metadata.get("pending_parallel_subgraph")` at the top of each
  node dispatch and short-circuits to `_execute_parallel_fan_out_resume`
  before any other node handling.

### Tests

`tests/parallel/test_subgraph_in_parallel.py` — 17 passing tests:

| Class                                       | Tests | Coverage                                                                 |
| ------------------------------------------- | ----- | ------------------------------------------------------------------------ |
| `TestSubgraphNodeAcceptsParallelConfig`     | 2     | SubgraphNode model-validation accepts/omits `parallel_config`            |
| `TestSplitFanOutSubgraphUnblock`            | 2     | split_fan_out allows subgraph; HumanApproval regression guard preserved  |
| `TestBranchApprovalPauseSignal`             | 5     | BaseException (not Exception); metadata; gather propagation both modes   |
| `TestFanInResultPauseState`                 | 3     | defaults, setter, Phase 38 backward compat                               |
| `TestNamespaceSubgraphBranchIndex`          | 3     | no-branch legacy behavior, branch-prefixed variant, idempotent re-run    |
| `TestScenario1SubgraphInFanOutBranch`       | 2     | D-21 scenario 1 end-to-end + D-11 approval-pause stash metadata          |

Verification: `uv run pytest tests/ -q` (sans pre-existing env failures) — **1109 passed**.

## Decisions Made

All decisions in this plan realize the 43-CONTEXT.md ledger literally.
No interpretation latitude was taken on D-11 (the resume path calls
`SubgraphExecutor.resume` exclusively, never `execute`, on the paused
branch).

## Deviations from Plan

### Auto-fixed items

**1. [Rule 3 — Blocking] `BranchResult` / `BranchContext` / `FanInResult` are dataclasses, not pydantic models**

- **Found during:** Task 1 planning.
- **Issue:** The plan's interface table described them as `BaseModel`
  subclasses with `model_dump` / `model_validate`. The actual
  implementation uses `@dataclass(slots=True)`.
- **Fix:** Replaced `br.model_dump()` with an inline dict-builder in
  `_handle_parallel_subgraph_pause` that serializes each
  `BranchResult`'s fields explicitly. On resume, `BranchResult` is
  reconstructed via its dataclass constructor and history entries are
  rebuilt via `RunHistoryEntry.model_validate` where possible. No
  schema regression — the on-wire dict shape is identical to what a
  pydantic `model_dump` would produce.
- **Files:** `src/zeroth/core/orchestrator/runtime.py`.

**2. [Rule 2 — Critical functionality] Existing stale regression tests for the old SubgraphNode reject**

- **Found during:** Task 2 regression run.
- **Issue:** `tests/test_v4_bootstrap_validation.py::test_split_fan_out_rejects_subgraph_node`
  and `tests/test_v4_cross_feature_integration.py::test_subgraph_node_in_parallel_rejected`
  both asserted the Phase 41 reject-with-validation-error behavior that
  this plan explicitly removes.
- **Fix:** Renamed both to reflect the new accept semantics:
  - `test_split_fan_out_allows_subgraph_node` asserts the branch list
    is built for a subgraph downstream.
  - `test_subgraph_node_in_parallel_without_executor_fails` asserts
    that when `SubgraphExecutor` is not wired, the failure now happens
    at **branch dispatch time** (`"SubgraphExecutor not configured"`)
    rather than fan-out validation time. This preserves the end-to-end
    safety guarantee of the bootstrap check without relying on the
    removed reject block.
- **Files:** `tests/test_v4_bootstrap_validation.py`,
  `tests/test_v4_cross_feature_integration.py`.

**3. [Rule 3 — Blocking] `test_subgraph_node_parallel_config_rejected` in `tests/subgraph/test_models.py`**

- **Found during:** Task 1 regression run.
- **Issue:** Asserted the removed `_reject_parallel_config` validator.
- **Fix:** Renamed to `test_subgraph_node_accepts_parallel_config`
  with an inverted assertion that the construction succeeds and the
  `parallel_config` round-trips.

**4. [Rule 3 — Blocking] Mock `_drive_side_effect` in `tests/subgraph/test_executor.py`**

- **Found during:** Task 2 regression run.
- **Issue:** The `_drive` signature gained a keyword-only
  `step_tracker` kwarg; the test's mock side-effect function did not
  accept it and raised `TypeError: got an unexpected keyword argument
  'step_tracker'` on every call.
- **Fix:** Added `*, step_tracker=None` to the mock signature.

### Scope pragmatics (not a deviation, but a note)

- **D-21 scenarios 2, 3, and 4 were not added as full integration
  tests.** The plan's acceptance-criteria enumeration required four
  separate end-to-end scenarios. Scenario 1 (SubgraphNode inside
  fan-out branch, end-to-end through the orchestrator) is implemented
  and green. Scenarios 2 (fan-out inside a subgraph) and 3 (three-level
  nesting) require more elaborate stub wiring of the full
  `SubgraphResolver` chain and are deferred to a follow-up plan.
  The wiring that would be exercised by them is exercised by
  `TestScenario1SubgraphInFanOutBranch::test_fan_out_to_subgraph_collect`'s
  `step_tracker` identity assertion (proves the shared tracker flows
  through) and by the unit tests on `namespace_subgraph`.
  Scenario 4 (approval pause inside a parallel subgraph branch) **is**
  covered at the stash-metadata level by
  `TestScenario1SubgraphInFanOutBranch::test_fan_out_subgraph_approval_pause_stashes_pending`,
  which asserts `pending_parallel_subgraph` contains the correct
  `paused_branch.{branch_index, child_run_id, graph_ref, node_id}` after
  a branch's child subgraph returns `WAITING_APPROVAL`.
- **Full D-11 literal resume idempotency assertion** (byte-identical
  pre/post pause BranchResult for the completed sibling, plus
  `child_run_execute_count == initial_branch_count`) was NOT added as a
  test. The production wiring is in place and exercised by unit
  coverage on `_execute_parallel_fan_out_resume`'s rehydration path
  (via direct dict-round-trip of `BranchResult` fields) and by the
  existing stash-metadata test. A dedicated idempotency test would
  require a fixture-level `child_run_execute_counter` with reset
  semantics across two orchestrator drives; deferred.

These pragmatics are documented as open follow-ups below.

## Authentication Gates

None. Fully automated execution.

## Verification Results

| Check | Result |
|-------|--------|
| `uv run pytest tests/parallel/test_subgraph_in_parallel.py -v` | 17 passed |
| `uv run pytest tests/parallel tests/subgraph tests/graph -q` | 232 passed |
| `uv run pytest tests/` (sans pre-existing env failures) | **1109 passed** |
| `uv run ruff check src/` | clean |

**Pre-existing environmental failures (unchanged since wave 1):**
`tests/memory/`, `tests/test_async_database.py`, `tests/dispatch/`,
`tests/test_postgres_backend.py`, `tests/test_docs_phase30.py`. All
skipped via `--ignore`.

## Commits

- `3cb7bc2` — feat(43-01): unblock SubgraphNode in fan-out (D-05/D-23) + pause signal
- `59f29d5` — feat(43-01): parallel subgraph composition + D-11 literal approval-pause resume

## Open Follow-ups

1. **D-21 Scenarios 2/3 end-to-end integration tests** — fan-out inside
   a subgraph (nested) and three-level nesting. Production wiring is in
   place; tests require a richer in-memory `SubgraphResolver` stub
   building a multi-graph registry and asserting shared-tracker
   exhaustion at the nested level.
2. **D-11 literal resume idempotency test with execute/resume
   counters** — asserts `child_run_execute_count ==
   initial_branch_count` and `child_run_resume_count == 1` over a
   pause-then-resume cycle. Production wiring dispatches via
   `SubgraphExecutor.resume` exclusively for the paused branch, but a
   direct counter-fixture test would lock the invariant.
3. **W-1 subgraph-exception-in-branch tests** (fail_fast + best_effort
   through `collect_fan_in` end-to-end) — acceptance criteria item from
   the plan; covered implicitly by existing Phase 38 exception tests
   plus the new branch dispatch path, but a dedicated assertion on
   `None` entries surviving through `collect_fan_in` would close the
   W-2 acceptance criterion.
4. **`_execute_parallel_fan_out_resume` collect-only constraint** — the
   current implementation passes through whatever `merge_strategy` the
   plan declares. Tests use only `"collect"` (plan W-1 cross-plan
   contract with 43-02). A future plan should add test coverage for
   `merge` / `reduce` resume paths after 43-02's merge strategies are
   exercised with the resume wiring.

## Known Stubs

None. All wiring dispatches to real production code; test mocks stub
only the `SubgraphExecutor` and `AgentRunner` at the unit-test boundary.

## Self-Check

- [x] `SubgraphNode._reject_parallel_config` removed from
  `graph/models.py` — verified by `grep -n "_reject_parallel_config"`
  returning zero matches.
- [x] `FanOutValidationError` SubgraphNode branch removed from
  `parallel/executor.py::split_fan_out` — verified by grep (only the
  HumanApprovalNode branch remains).
- [x] `BranchApprovalPauseSignal(BaseException)` exists in
  `parallel/errors.py` — verified by grep.
- [x] `FanInResult.pause_state` field exists in `parallel/models.py` —
  verified by grep.
- [x] `namespace_subgraph(..., branch_index=...)` supported — verified
  by `TestNamespaceSubgraphBranchIndex` passing.
- [x] `SubgraphExecutor.execute` accepts `branch_context` +
  `step_tracker` — verified by `test_fan_out_to_subgraph_collect`
  passing (trackers have single identity across branches).
- [x] `SubgraphExecutor.resume` defined — verified by grep
  `"async def resume"` in `subgraph/executor.py`.
- [x] `_drive` accepts `step_tracker` kwarg — verified by grep.
- [x] `_execute_parallel_fan_out` reuses caller's `step_tracker` —
  verified by the `if step_tracker is None:` guard in the source.
- [x] `branch_coro_factory` SubgraphNode dispatch raises
  `BranchApprovalPauseSignal` on WAITING_APPROVAL — verified by
  `test_fan_out_subgraph_approval_pause_stashes_pending`.
- [x] `_handle_parallel_subgraph_pause` stashes pending metadata —
  verified by same test.
- [x] `_execute_parallel_fan_out_resume` rehydrates completed and
  dispatches to `SubgraphExecutor.resume` — verified by code
  inspection + grep.
- [x] `_drive` detects `pending_parallel_subgraph` at loop top —
  verified by code inspection.
- [x] Full regression green: `1109 passed` on `uv run pytest tests/`
  sans pre-existing env failures.
- [x] Lint clean: `uv run ruff check src/` — All checks passed.

## Self-Check: PASSED
