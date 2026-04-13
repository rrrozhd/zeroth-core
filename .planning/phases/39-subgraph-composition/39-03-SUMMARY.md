---
phase: 39-subgraph-composition
plan: "03"
subsystem: subgraph-approval-propagation
tags: [subgraph, approval, orchestrator, resume, composition]
dependency_graph:
  requires: [39-02]
  provides: [approval-propagation, resume-chain, pending_subgraph-metadata]
  affects: [orchestrator-runtime]
tech_stack:
  added: []
  patterns: [two-phase-resume, pending-metadata-pattern, recursive-resume]
key_files:
  created:
    - tests/subgraph/test_approval_propagation.py
    - tests/subgraph/test_integration.py
  modified:
    - src/zeroth/core/orchestrator/runtime.py
decisions:
  - "Resume path uses self.resume_graph() for child (not _drive() directly) to get status validation and standard resume flow"
  - "pending_subgraph metadata stores only graph_ref and version (not full Graph object) to keep run metadata lightweight; subgraph is re-resolved on resume"
  - "Parent explicitly sets status=WAITING_APPROVAL on nested resume (child still waiting) to handle case where parent status was set to RUNNING by caller"
metrics:
  duration: "11m 11s"
  completed: "2026-04-13"
  tasks: 2
  tests_added: 26
  files_created: 2
  files_modified: 1
---

# Phase 39 Plan 03: Approval Propagation and Integration Tests Summary

Two-phase resume pattern for approval propagation across subgraph boundaries: child WAITING_APPROVAL cascades to parent, resume_graph cascades back down, with re-resolution and governance merge on each resume cycle.

## What Was Built

### Task 1: Approval propagation and resume chain in _drive() SubgraphNode handler

Modified `src/zeroth/core/orchestrator/runtime.py` to add two new code paths in the SubgraphNode handler:

**Path A (first encounter):** After `SubgraphExecutor.execute()` returns a child run with `WAITING_APPROVAL` status:
- Parent transitions to `WAITING_APPROVAL`
- Stores `pending_subgraph` metadata: `{child_run_id, node_id, graph_ref, version}`
- Re-queues the SubgraphNode at front of `pending_node_ids`
- Checkpoints and returns

**Path B (resume after approval):** When `_drive()` pops a SubgraphNode and finds matching `pending_subgraph` metadata:
- Re-resolves the subgraph via `resolver.resolve(graph_ref, version)` (Graph objects too large for metadata)
- Re-namespaces and re-merges governance (parent-ceiling)
- Calls `self.resume_graph(subgraph, child_run_id)` to continue the child
- If child still `WAITING_APPROVAL`: parent stays paused (nested approval)
- If child `COMPLETED`: clears `pending_subgraph`, records audit with `subgraph_resumed=True`, continues

Added import for `namespace_subgraph` and `merge_governance` from `zeroth.core.subgraph.resolver`.

12 tests in `tests/subgraph/test_approval_propagation.py`:
- 4 first-encounter tests (parent pauses, stores metadata, re-queues, checkpoints)
- 7 resume tests (child completes, clears metadata, child still waiting, re-resolves, governance, resolution failure, audit)
- 1 two-level nesting test (full pause-resume round trip)

**Commits:** `652d068` (RED), `ae13c50` (GREEN)

### Task 2: Comprehensive integration tests for full subgraph composition lifecycle

Created `tests/subgraph/test_integration.py` with 14 integration tests covering all 10 aspects:

1. **Happy path** (2 tests): parent completes with child output; output propagates to successor node
2. **Thread inheritance** (2 tests): inherit mode passes parent thread_id; isolated mode configured
3. **Governance ceiling** (1 test): parent policy_bindings propagate to child executor
4. **Depth limit** (2 tests): chain exceeding max_depth fails; depth tracked in metadata
5. **Cycle detection** (1 test): circular reference produces SubgraphCycleError
6. **Multi-reference** (1 test): same subgraph referenced twice produces two distinct child runs
7. **Node ID namespacing** (1 test): child history has `subgraph:` prefixed IDs
8. **Audit trail** (1 test): parent history contains subgraph_run_id
9. **Error paths** (1 test): non-existent graph_ref fails with resolution error
10. **Full approval flow** (2 tests): pause at child approval, resume completes; pending_subgraph preserved

**Commit:** `3be80b8`

## Verification Results

| Check | Result |
|-------|--------|
| `uv run pytest tests/subgraph/ -v` | 90 passed |
| `uv run pytest tests/orchestrator/ -v` | 11 passed (zero regressions) |
| `uv run ruff check src/zeroth/core/orchestrator/ src/zeroth/core/subgraph/ tests/subgraph/` | All checks passed |
| `uv run ruff format --check` (modified files) | All formatted |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Missing explicit WAITING_APPROVAL on nested resume**
- **Found during:** Task 1 GREEN phase
- **Issue:** When child still WAITING_APPROVAL on resume, the code returned without explicitly setting `run.status = RunStatus.WAITING_APPROVAL`. Parent status could be RUNNING if caller set it before driving.
- **Fix:** Added `run.status = RunStatus.WAITING_APPROVAL` before re-queuing and returning in the nested approval path.
- **Files modified:** `src/zeroth/core/orchestrator/runtime.py`
- **Commit:** ae13c50

**2. [Rule 1 - Bug] Lint warning for single-element tuple in except clause**
- **Found during:** Task 1 GREEN phase
- **Issue:** `except (SubgraphResolutionError,)` flagged by ruff B013 as redundant tuple.
- **Fix:** Changed to `except SubgraphResolutionError`.
- **Files modified:** `src/zeroth/core/orchestrator/runtime.py`
- **Commit:** ae13c50

**3. [Rule 3 - Blocking] PolicyBindingEntry import doesn't exist**
- **Found during:** Task 1 GREEN phase
- **Issue:** Test tried to import `PolicyBindingEntry` from `zeroth.core.graph.models` but `policy_bindings` is `list[str]`, not a list of objects.
- **Fix:** Changed test to use `["deny-dangerous"]` string list instead of PolicyBindingEntry objects.
- **Files modified:** `tests/subgraph/test_approval_propagation.py`
- **Commit:** ae13c50

**4. [Rule 1 - Bug] Resume tests had child_run with COMPLETED status**
- **Found during:** Task 1 GREEN phase
- **Issue:** `resume_graph()` validates run status and rejects COMPLETED runs. Test mocked child_run as COMPLETED, but `resume_graph` only accepts RUNNING/PENDING/WAITING_APPROVAL.
- **Fix:** Changed test child runs to WAITING_APPROVAL with empty pending_node_ids (simulating approval already resolved, no more work). `_drive()` immediately completes them.
- **Files modified:** `tests/subgraph/test_approval_propagation.py`
- **Commit:** ae13c50

## Decisions Made

1. **Resume uses public `resume_graph()` not direct `_drive()`:** The resume path calls `self.resume_graph(subgraph, child_run_id)` rather than fetching the child and calling `_drive()` directly. This ensures the standard status validation gate is applied, preventing accidentally resuming a completed or failed child run.

2. **Graph re-resolution on every resume:** The subgraph Graph object is NOT stored in `pending_subgraph` metadata (too large). Only `graph_ref` and `version` are stored. This means resolution happens twice: once on first encounter, once on resume. Version pin (if set) prevents drift; if no version pin, latest deployment is used (by design per D-07).

3. **Explicit status set on nested resume:** When child is still WAITING_APPROVAL, the parent explicitly sets `run.status = RunStatus.WAITING_APPROVAL` rather than relying on the status being preserved. This handles edge cases where the caller (e.g., `resume_graph`) may have set the parent to a different status.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 (RED) | 652d068 | test(39-03): add failing tests for approval propagation |
| 1 (GREEN) | ae13c50 | feat(39-03): implement approval propagation and resume chain |
| 2 | 3be80b8 | test(39-03): comprehensive integration tests for subgraph lifecycle |
| format | e68d515 | chore(39-03): format modified files |

## Self-Check: PASSED
