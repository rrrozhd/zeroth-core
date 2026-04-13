---
phase: 38-parallel-fan-out-fan-in
plan: 01
subsystem: parallel
tags: [parallel, fan-out, fan-in, asyncio, concurrency, branch-isolation]
dependency_graph:
  requires: [zeroth.core.mappings.executor, zeroth.core.graph.models]
  provides: [zeroth.core.parallel]
  affects: [zeroth.core.graph.models.NodeBase]
tech_stack:
  added: []
  patterns: [asyncio.gather, asyncio.Lock, dataclass-slots, pydantic-literal-validation]
key_files:
  created:
    - src/zeroth/core/parallel/__init__.py
    - src/zeroth/core/parallel/models.py
    - src/zeroth/core/parallel/errors.py
    - src/zeroth/core/parallel/executor.py
    - tests/parallel/__init__.py
    - tests/parallel/test_models.py
    - tests/parallel/test_executor.py
  modified:
    - src/zeroth/core/graph/models.py
decisions:
  - "GlobalStepTracker checks count >= max before increment (not after) -- allows exactly max_steps increments before raising"
  - "BranchContext uses @dataclass(slots=True) for performance and memory isolation"
  - "Non-dict items in split list wrapped as {'_item': value} for uniform BranchContext.input_payload typing"
metrics:
  duration: 8m 40s
  completed: "2026-04-13T01:13:43Z"
  tests_added: 37
  tests_passed: 37
  files_created: 7
  files_modified: 1
---

# Phase 38 Plan 01: Parallel Package Models, Errors, Executor & NodeBase Field Summary

Parallel fan-out/fan-in engine with asyncio.gather concurrency, BranchContext isolation, GlobalStepTracker async-safe limits, and ParallelExecutor split/execute/collect pipeline reusing mappings dot-path utilities.

## What Was Built

### New Package: `zeroth.core.parallel`

**models.py** -- Core data models:
- `ParallelConfig` (Pydantic, extra=forbid): split_path, merge_strategy (Literal["collect","reduce"]), fail_mode (Literal["fail_fast","best_effort"]), max_branches (int|None, ge=1)
- `BranchContext` (dataclass, slots=True): branch_index, branch_id, input_payload, plus isolated mutable state (empty node_visit_counts, execution_history, audit_refs, condition_results, metadata per D-05)
- `BranchResult` (dataclass, slots=True): branch_index, output (dict|None), error (str|None), audit_refs, execution_history, cost_usd
- `FanInResult` (dataclass, slots=True): results list, merged_output dict, total_cost_usd, total_steps
- `GlobalStepTracker`: asyncio.Lock-protected counter enforcing max_total_steps across all branches (D-06)

**errors.py** -- Error hierarchy:
- `ParallelExecutionError(RuntimeError)` -- base
- `BranchError` -- single branch failure with branch_index and original exception
- `FanOutValidationError` -- bad split_path, empty list, exceeded max_branches, unsupported node type
- `ParallelStepLimitError` -- global step limit exceeded

**executor.py** -- `ParallelExecutor` class:
- `split_fan_out()`: extracts list at split_path via `_get_path`, validates shape/type/size, creates N BranchContexts with "{run_id}:branch:{index}" IDs, wraps non-dict items as {"_item": value}
- `execute_branches()`: best-effort mode uses `asyncio.gather(return_exceptions=True)`, fail-fast mode uses `asyncio.create_task` with explicit cancellation on first exception
- `collect_fan_in()`: sorts by branch_index, builds output list (None preserved for failed branches per D-08), merges at split_path via `_set_path`, aggregates cost and step counts

**__init__.py** -- Re-exports all public names from models, errors, and executor.

### Modified: `src/zeroth/core/graph/models.py`

Added `parallel_config: ParallelConfig | None = None` field to `NodeBase`, making parallel fan-out configurable on any node type.

### Tests: 37 tests across 2 files

**test_models.py** (20 tests): ParallelConfig validation (defaults, Literal types, max_branches ge=1, extra=forbid), BranchContext isolation (empty defaults, no mutable sharing), BranchResult/FanInResult construction, GlobalStepTracker (within limit, at limit raises, above limit raises, 10 concurrent increments with max=5 yields exactly 5 successes), NodeBase with/without parallel_config.

**test_executor.py** (17 tests): split_fan_out (valid split, not found, not a list, empty, exceeds max_branches, nested path, non-dict wrapping, HumanApprovalNode rejection), execute_branches (best-effort mixed, fail-fast cancels, all succeed), collect_fan_in (all successful, failed=None, merge_path defaults, cost/steps aggregation, ordering preserved), real HumanApprovalNode node type validation.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed GlobalStepTracker test initial value**
- **Found during:** GREEN phase, test_increment_raises_at_limit
- **Issue:** Plan specified `current_steps=4, max_steps=5` but the correct semantics (check >= max before increment, allowing exactly max_steps increments) requires `current_steps=5` for immediate raise
- **Fix:** Changed test to `current_steps=5, max_steps=5` -- consistent with concurrent test (0, 5 yields exactly 5 successes) and above_limit test (0, 3 yields 3 successes then raise)
- **Files modified:** tests/parallel/test_models.py
- **Commit:** 5e474e1

## Threat Mitigations Verified

| Threat ID | Mitigation | Status |
|-----------|-----------|--------|
| T-38-01 | ParallelConfig extra="forbid" + Literal types | Verified via test_extra_fields_forbidden, test_invalid_merge_strategy_rejected, test_invalid_fail_mode_rejected |
| T-38-02 | max_branches cap + empty list rejection | Verified via test_exceeds_max_branches_raises, test_empty_list_raises |
| T-38-03 | asyncio.Lock-protected GlobalStepTracker | Verified via test_concurrent_increments_respect_limit (10 concurrent, exactly 5 succeed) |
| T-38-04 | BranchContext isolated copies | Verified via test_isolated_defaults, test_mutable_defaults_not_shared |
| T-38-06 | Explicit task cancellation in fail-fast | Verified via test_fail_fast_cancels_remaining |

## Self-Check: PASSED

- [x] src/zeroth/core/parallel/__init__.py exists
- [x] src/zeroth/core/parallel/models.py exists
- [x] src/zeroth/core/parallel/errors.py exists
- [x] src/zeroth/core/parallel/executor.py exists
- [x] tests/parallel/__init__.py exists
- [x] tests/parallel/test_models.py exists
- [x] tests/parallel/test_executor.py exists
- [x] Commit 8269d83 (RED) exists
- [x] Commit 5e474e1 (GREEN) exists
- [x] 37/37 tests pass
- [x] Lint clean
- [x] Graph tests (14) still pass
