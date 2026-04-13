---
phase: 39-subgraph-composition
plan: 01
one_liner: "Subgraph package foundation with models, error hierarchy, resolver, node ID namespacing, and Run parent-child linking"
completed: 2026-04-13
duration_minutes: 7
tasks_completed: 2
tasks_total: 2
test_count: 39
tests_passed: 39
tests_failed: 0
regression_tests: 24
regression_passed: 24
dependency_graph:
  requires: []
  provides:
    - zeroth.core.subgraph package (models, errors, resolver)
    - SubgraphNode in Node discriminated union
    - Run.parent_run_id field
  affects:
    - src/zeroth/core/graph/models.py (Node union extended)
    - src/zeroth/core/graph/__init__.py (new exports)
    - src/zeroth/core/runs/models.py (parent_run_id added)
tech_stack:
  added: []
  patterns:
    - "dataclass(slots=True) for SubgraphResolver"
    - "model_copy() for immutable graph transformations"
    - "Discriminated union extension for SubgraphNode"
key_files:
  created:
    - src/zeroth/core/subgraph/__init__.py
    - src/zeroth/core/subgraph/models.py
    - src/zeroth/core/subgraph/errors.py
    - src/zeroth/core/subgraph/resolver.py
    - tests/subgraph/__init__.py
    - tests/subgraph/test_models.py
    - tests/subgraph/test_resolver.py
  modified:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/graph/__init__.py
    - src/zeroth/core/runs/models.py
decisions:
  - "Resolver not re-exported from subgraph/__init__.py to avoid circular import (graph -> subgraph -> resolver -> deployments -> graph)"
  - "SubgraphNode parallel_config rejection relies on NodeBase extra=forbid rather than explicit validator since parallel_config is not a field in this codebase version"
---

# Phase 39 Plan 01: Subgraph Package Foundation Summary

Subgraph package foundation with models, error hierarchy, resolver, node ID namespacing, and Run parent-child linking.

## Task Results

### Task 1: Create subgraph models, errors, and extend Run/Node types (TDD)

| Step | Result |
|------|--------|
| RED | 25 tests written, all fail (module not found) |
| GREEN | All 25 tests pass |
| Regressions | 24/24 existing graph+runs tests pass |

**Created:**
- `SubgraphNodeData` model with `graph_ref`, `version`, `thread_participation`, `max_depth` (bounded 1-10)
- `SubgraphNode(NodeBase)` with `node_type="subgraph"`, `to_governed_step_spec()` returning `kind="subgraph_ref"`
- Error hierarchy: `SubgraphError(RuntimeError)` base, with `SubgraphDepthLimitError`, `SubgraphResolutionError`, `SubgraphExecutionError`, `SubgraphCycleError`
- `Node` discriminated union updated to include `SubgraphNode`
- `Run.parent_run_id: str | None = None` for parent-child linking

**Commits:** `64e94da` (RED), `f52cb23` (GREEN)

### Task 2: Create SubgraphResolver with graph resolution and node ID namespacing (TDD)

| Step | Result |
|------|--------|
| RED | 14 tests written, all fail (module not found) |
| GREEN | All 14 tests pass |
| Regressions | 24/24 existing graph+runs tests pass |

**Created:**
- `SubgraphResolver` dataclass with `resolve(graph_ref, version)` via `DeploymentService.get()`
- `namespace_subgraph()` prefixes all node_ids, edge fields, entry_step with `subgraph:{ref}:{depth}:`
- `merge_governance()` prepends parent policy_bindings for parent-ceiling semantics
- All functions return copies (original graphs never modified)

**Commits:** `4448be6` (RED), `3b2d81e` (GREEN)

## Verification

| Check | Result |
|-------|--------|
| `uv run pytest tests/subgraph/ -v` | 39 passed |
| `uv run pytest tests/graph/ tests/runs/ -v` | 24 passed (zero regressions) |
| `uv run ruff check src/zeroth/core/subgraph/ src/zeroth/core/graph/ src/zeroth/core/runs/` | All checks passed |
| `uv run ruff format --check src/zeroth/core/subgraph/ src/zeroth/core/graph/ src/zeroth/core/runs/` | 17 files already formatted |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Circular import in subgraph/__init__.py**
- **Found during:** Task 2
- **Issue:** Importing SubgraphResolver in `subgraph/__init__.py` created a circular import chain: `graph/__init__.py` -> `graph/models.py` -> `subgraph/models.py` -> (via `__init__.py`) -> `subgraph/resolver.py` -> `deployments/service.py` -> `graph/__init__.py`
- **Fix:** Removed SubgraphResolver, namespace_subgraph, merge_governance from `subgraph/__init__.py` re-exports. Users import directly from `zeroth.core.subgraph.resolver`.
- **Files modified:** `src/zeroth/core/subgraph/__init__.py`
- **Commit:** `3b2d81e`

**2. [Rule 3 - Blocking] SubgraphNode parallel_config validator approach**
- **Found during:** Task 1
- **Issue:** Plan specified adding a `@model_validator` to reject parallel_config on SubgraphNode (matching HumanApprovalNode pattern). However, `NodeBase` in this worktree does not have a `parallel_config` field (it was added in Phase 38 which is in a different branch). No validator needed because `extra="forbid"` on NodeBase already rejects unknown fields.
- **Fix:** Test verifies that parallel_config is rejected via extra="forbid" rather than explicit validator.
- **Files modified:** `tests/subgraph/test_models.py`
- **Commit:** `f52cb23`

## Decisions Made

1. **Resolver import path:** SubgraphResolver is imported from `zeroth.core.subgraph.resolver` rather than `zeroth.core.subgraph` to avoid circular imports. This is consistent with how the deployments service works (also has internal cross-references to graph).

2. **parallel_config rejection:** Relies on NodeBase `extra="forbid"` since the parallel_config field doesn't exist on NodeBase in this codebase version. When Phase 38 is merged and parallel_config is added to NodeBase, a model_validator should be added to SubgraphNode to explicitly reject it.

## Self-Check: PASSED

All 8 created files verified on disk. All 4 commit hashes verified in git log.
