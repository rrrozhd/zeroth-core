---
phase: 39-subgraph-composition
plan: "02"
subsystem: subgraph-executor
tags: [subgraph, orchestrator, recursion, composition, bootstrap]
dependency_graph:
  requires: [39-01]
  provides: [SubgraphExecutor, _drive-SubgraphNode-handling, bootstrap-subgraph-wiring]
  affects: [orchestrator-runtime, service-bootstrap]
tech_stack:
  added: []
  patterns: [recursive-drive, child-run-creation, depth-tracking, cycle-detection, lazy-import]
key_files:
  created:
    - src/zeroth/core/subgraph/executor.py
    - tests/subgraph/test_executor.py
    - tests/subgraph/test_drive_subgraph.py
  modified:
    - src/zeroth/core/subgraph/__init__.py
    - src/zeroth/core/orchestrator/runtime.py
    - src/zeroth/core/service/bootstrap.py
decisions:
  - "Used Any type hint for subgraph_executor field on RuntimeOrchestrator to avoid circular import (SubgraphExecutor imports RuntimeOrchestrator via TYPE_CHECKING)"
  - "Used lazy __getattr__ in subgraph/__init__.py to defer SubgraphExecutor import, preventing circular import chain through graph.models"
  - "SubgraphNode handling placed after HumanApprovalNode check and before _dispatch_node in _drive() loop, with continue to skip normal dispatch flow"
metrics:
  duration: "9m 25s"
  completed: "2026-04-13"
  tasks: 2
  tests_added: 25
  files_created: 3
  files_modified: 3
---

# Phase 39 Plan 02: SubgraphExecutor and _drive() Integration Summary

SubgraphExecutor creates child Runs with parent linkage and thread participation control, drives them recursively via orchestrator._drive(), with depth tracking and cycle detection preventing infinite recursion. Bootstrap wires it automatically.

## What Was Built

### Task 1: SubgraphExecutor with child Run creation and recursive _drive()

Created `src/zeroth/core/subgraph/executor.py` containing the `SubgraphExecutor` dataclass. The `execute()` method performs these steps in order:

1. **Depth check** -- reads `subgraph_depth` from parent run metadata, raises `SubgraphDepthLimitError` if new depth exceeds `max_depth` (capped at 10 by model validator)
2. **Cycle detection** -- reads `visited_subgraph_refs` from parent run metadata, raises `SubgraphCycleError` if the current `graph_ref` is already in the visited set
3. **Resolution** -- calls `SubgraphResolver.resolve()` to look up the published graph
4. **Namespacing** -- calls `namespace_subgraph()` to prefix all node/edge IDs
5. **Governance merge** -- calls `merge_governance()` to prepend parent policy bindings
6. **Child Run creation** -- creates a Run with `parent_run_id`, correct `thread_id` based on thread participation mode, and metadata containing depth/lineage tracking
7. **Recursive execution** -- calls `orchestrator._drive()` on the merged subgraph with the child Run

Updated `src/zeroth/core/subgraph/__init__.py` with lazy `__getattr__` for `SubgraphExecutor` to avoid circular import through `graph.models`.

13 tests in `tests/subgraph/test_executor.py` covering happy path, depth tracking, cycle detection, error wrapping, and None orchestrator.

### Task 2: Integrate SubgraphNode into _drive() loop and wire bootstrap

Modified `src/zeroth/core/orchestrator/runtime.py`:
- Added `subgraph_executor: Any | None = None` field to `RuntimeOrchestrator`
- Added `SubgraphNode` to graph imports, added subgraph error imports
- Inserted SubgraphNode detection block in `_drive()` after HumanApprovalNode check and before `_dispatch_node()` -- delegates to `subgraph_executor.execute()`, propagates child run's `final_output` as node output, records history with subgraph audit metadata, continues loop

Modified `src/zeroth/core/service/bootstrap.py`:
- Added `subgraph_executor: object | None = None` field to `ServiceBootstrap`
- Added wiring in `bootstrap_service()`: creates `SubgraphResolver(deployment_service)` and `SubgraphExecutor(resolver)`, sets `orchestrator.subgraph_executor`, passes to `ServiceBootstrap` return

12 tests in `tests/subgraph/test_drive_subgraph.py` covering _drive() delegation, error handling (all 4 subgraph error types), no-executor failure, output propagation, history recording, and bootstrap field/wiring.

## Verification Results

- `uv run pytest tests/subgraph/ -v` -- 64 passed
- `uv run pytest tests/orchestrator/ -v` -- 11 passed (zero regressions)
- `uv run ruff check` -- all checks passed
- `uv run ruff format --check` -- all files formatted

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Circular import via __init__.py**
- **Found during:** Task 1
- **Issue:** Importing `SubgraphExecutor` at module level in `subgraph/__init__.py` caused a circular import chain: `graph.models` -> `subgraph.models` -> `subgraph/__init__` -> `subgraph.executor` -> `graph.models`
- **Fix:** Used `__getattr__` lazy import pattern in `__init__.py` to defer `SubgraphExecutor` import until first access
- **Files modified:** `src/zeroth/core/subgraph/__init__.py`
- **Commit:** 4f6316b

**2. [Rule 1 - Bug] Missing deployment_id in test helper**
- **Found during:** Task 1
- **Issue:** `_make_child_deployment()` test helper was missing required `deployment_id` field
- **Fix:** Added `deployment_id="dep-child-1"` to the Deployment constructor
- **Files modified:** `tests/subgraph/test_executor.py`
- **Commit:** 4f6316b

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 4f6316b | feat(39-02): SubgraphExecutor with child Run creation and recursive _drive() |
| 2 | 13b4fef | feat(39-02): integrate SubgraphNode into _drive() loop and wire bootstrap |

## Self-Check: PASSED

All 3 created files exist. Both commit hashes verified. All 11 acceptance criteria met.
