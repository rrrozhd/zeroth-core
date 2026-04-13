---
phase: 38-parallel-fan-out-fan-in
plan: 02
title: "Orchestrator Fan-Out Integration & Governance"
subsystem: orchestrator, parallel
tags: [fan-out, fan-in, governance, audit, policy, budget, step-limit, tdd]
dependency_graph:
  requires: [38-01]
  provides: [parallel-orchestrator-integration, per-branch-governance]
  affects: [orchestrator-runtime, parallel-executor]
tech_stack:
  added: []
  patterns: [branch-coro-factory, global-step-tracker, fan-out-detection-in-drive-loop]
key_files:
  created:
    - tests/parallel/test_drive_integration.py
    - tests/parallel/test_governance_integration.py
  modified:
    - src/zeroth/core/orchestrator/runtime.py
decisions:
  - "Governance logic in runtime.py _execute_parallel_fan_out rather than executor.py -- single method owns budget check, branch dispatch, audit, policy, step tracking"
  - "Post-fan-out downstream planning skips already-executed branch nodes, plans from downstream perspective instead"
  - "executor.py unchanged from Plan 01 -- ParallelExecutor is a pure split/execute/collect engine, governance wraps around it"
metrics:
  duration_seconds: 654
  completed: "2026-04-13T01:35:30Z"
  tasks_completed: 2
  tasks_total: 2
  test_count: 18
  files_changed: 3
  insertions: 1183
---

# Phase 38 Plan 02: Orchestrator Fan-Out Integration & Governance Summary

Wired ParallelExecutor into RuntimeOrchestrator._drive() loop with full per-branch governance: audit records with branch_id, per-branch policy enforcement, budget pre-reservation, global step tracking, and branch visit count isolation.

## What Changed

### RuntimeOrchestrator (runtime.py)

**New field:** `parallel_executor: ParallelExecutor` -- default-constructed, available on all orchestrator instances.

**Fan-out detection in _drive():** After `_dispatch_node()` returns, checks `getattr(node, 'parallel_config', None)`. When non-None, delegates to `_execute_parallel_fan_out()` instead of normal history/planning flow. Uses `continue` to re-enter the loop without queuing downstream nodes that were already executed in branches.

**_execute_parallel_fan_out():** Core integration method that:
1. Splits node output via `parallel_executor.split_fan_out()`
2. Checks budget via `budget_enforcer.check_budget()` before spawning (fail-open when None)
3. Creates `GlobalStepTracker` from current history length and `max_total_steps`
4. Builds `branch_coro_factory` that for each branch: enforces policy, dispatches downstream node, increments step tracker, writes audit record with branch_id/branch_index in execution_metadata, appends to branch-isolated history
5. Executes branches via `parallel_executor.execute_branches()`
6. Collects results via `parallel_executor.collect_fan_in()`

**_enforce_policy_for_branch():** Lightweight policy check that returns denial reason string (or None), used inside branch coroutines where we cannot return a Run directly.

**_merge_fan_in_state():** Appends all branch execution_history entries and audit_refs into the parent Run for full traceability.

**Post-fan-out planning:** After fan-in, plans next nodes from the downstream node perspective (not the source node), preventing already-executed downstream nodes from being re-queued.

### Tests

**test_drive_integration.py (8 tests):**
- Sequential execution unchanged (backward compat)
- Basic 2-item fan-out with merged output
- 5-item ordering preservation by branch_index
- Best-effort: failed branch produces None, run completes
- Fail-fast: failed branch causes run failure
- parallel_config=None has no effect
- Branch history entries merged into parent run
- Branch audit refs merged into parent run

**test_governance_integration.py (10 tests):**
- Per-branch audit records carry branch_id and branch_index
- All branch audits linked to parent run_id
- Policy guard called once per branch
- Policy denial in one branch fails that branch only (best-effort)
- Budget denied before spawn prevents fan-out
- Budget allowed proceeds normally
- Budget enforcer None skips check (fail-open)
- Global step limit enforced across branches
- Branch visit counts isolated from parent
- Per-branch contract validation via independent _dispatch_node calls

## Deviations from Plan

### Architectural Simplification

**1. [Rule 2 - Simplification] executor.py unchanged**
- **Plan expected:** Modifications to `src/zeroth/core/parallel/executor.py` for governance integration
- **Actual:** All governance logic resides in `runtime.py`'s `_execute_parallel_fan_out` method. The ParallelExecutor from Plan 01 is a pure split/execute/collect engine. Governance (audit, policy, budget, step tracking) wraps around it in the branch_coro_factory closure, which has access to the orchestrator's services. This is cleaner separation of concerns.

### TDD Adjustment

**2. [Rule 3 - Blocking] Task 2 governance tests passed immediately**
- **Plan expected:** RED-GREEN-REFACTOR for Task 2
- **Actual:** Governance features were implemented as part of Task 1's `_execute_parallel_fan_out` method because they form a single cohesive unit with the fan-out logic. Task 2 tests serve as verification that governance behavior is correct. The 10 governance tests all passed on first run.

## Verification Results

| Suite | Tests | Status |
|-------|-------|--------|
| tests/parallel/ | 55 | PASSED |
| tests/orchestrator/ | 11 | PASSED |
| ruff check (parallel + orchestrator + graph) | - | CLEAN |

## Threat Mitigations Applied

| Threat ID | Mitigation |
|-----------|------------|
| T-38-07 (Run state tampering) | BranchContext has isolated node_visit_counts, execution_history, audit_refs; branches never mutate parent Run state; merge happens sequentially after barrier |
| T-38-08 (Budget DoS) | check_budget() called before spawning; fail-open when budget_enforcer is None |
| T-38-09 (Repudiation) | Each branch produces NodeAuditRecord with branch_id and branch_index in execution_metadata, linked to parent run_id |
| T-38-10 (Policy bypass) | _enforce_policy_for_branch() called per branch before each downstream dispatch; denial in one branch does not affect others |
| T-38-11 (Orphaned tasks) | Handled by Plan 01 executor's fail-fast task cancellation |

## Self-Check: PASSED

- All 3 created/modified files exist on disk
- All 3 commits (ad6b33f, c4ac499, 162e4f9) exist in git log
- All 15 acceptance criteria grep checks pass
- 55 parallel tests pass, 11 orchestrator tests pass, ruff lint clean
