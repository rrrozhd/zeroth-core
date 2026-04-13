---
phase: 40-integration-service-wiring
plan: 01
subsystem: integration-service-wiring
tags: [bootstrap-validation, cross-feature-integration, parallel-guard, subgraph, template, context-window, artifact-store]
dependency_graph:
  requires: [phase-34, phase-35, phase-36, phase-37, phase-38, phase-39]
  provides: [v4-bootstrap-validation, cross-feature-integration-tests, subgraph-parallel-guard]
  affects: [src/zeroth/core/parallel/executor.py]
tech_stack:
  added: []
  patterns: [cross-feature-integration-testing, node-type-guard-pattern]
key_files:
  created:
    - tests/test_v4_bootstrap_validation.py
    - tests/test_v4_cross_feature_integration.py
  modified:
    - src/zeroth/core/parallel/executor.py
decisions:
  - "Used string-based node_type check (not isinstance) for SubgraphNode guard, matching existing HumanApprovalNode guard pattern"
  - "Used BranchItemInput/BranchValueInput models for branch-specific input validation in cross-feature tests"
  - "Mocked SubgraphExecutor for template-in-subgraph test to avoid full subgraph resolution stack"
metrics:
  duration: 365s
  completed: "2026-04-13T03:41:42Z"
  tasks_completed: 2
  tasks_total: 2
  test_count: 12
  files_changed: 3
---

# Phase 40 Plan 01: Bootstrap Validation and Cross-Feature Integration Summary

SubgraphNode-in-parallel guard plus 12 integration tests proving all v4.0 subsystems are wired and interact correctly across parallel, template, context window, artifact, and subgraph features.

## What Was Done

### Task 1: Bootstrap validation test + SubgraphNode-in-parallel guard

Added 6 tests in `tests/test_v4_bootstrap_validation.py`:
- 5 bootstrap validation tests confirming all v4.0 subsystem fields are non-None after `bootstrap_service()`: `artifact_store`, `http_client`, `template_registry`, `subgraph_executor`, and `orchestrator.context_window_enabled`
- 1 test for SubgraphNode guard: `split_fan_out` rejects nodes with `node_type == "subgraph"` by raising `FanOutValidationError`

Implementation: Added a SubgraphNode type check in `src/zeroth/core/parallel/executor.py` `split_fan_out()` method, immediately after the existing HumanApprovalNode guard. Uses string-based `node_type` check to match the existing pattern and avoid importing SubgraphNode into the parallel module.

### Task 2: Cross-feature integration tests for D-05 scenarios

Added 6 tests in `tests/test_v4_cross_feature_integration.py`:
1. `test_parallel_branches_with_artifact_store` - Parallel fan-out with `FilesystemArtifactStore` completes without error
2. `test_parallel_branches_respect_context_window` - Parallel fan-out with `context_window_enabled=True` and per-node `ContextWindowSettings` completes
3. `test_subgraph_node_in_parallel_rejected` - Graph with parallel fan-out targeting SubgraphNode fails with clear error containing "SubgraphNode"
4. `test_template_resolution_in_parallel_branches` - Template resolution via `TemplateRegistry` + `TemplateRenderer` works inside branch_coro_factory dispatch
5. `test_template_resolution_in_subgraph` - SubgraphNode with template_ref on child graph completes (mocked SubgraphExecutor)
6. `test_concurrent_branch_runner_isolation` - 4 concurrent branches targeting the same downstream agent produce correct, non-corrupted output

## Commits

| Commit | Type | Description |
|--------|------|-------------|
| 0311dbe | feat | Bootstrap validation tests and SubgraphNode-in-parallel guard |
| c6806f5 | feat | Cross-feature integration tests for D-05 scenarios |

## Verification

```
67 passed in 2.20s (6 bootstrap + 6 cross-feature + 55 existing parallel tests)
ruff check: All checks passed!
```

## Deviations from Plan

None - plan executed exactly as written.

## Threat Mitigations

| Threat ID | Status | Implementation |
|-----------|--------|----------------|
| T-40-01 | Mitigated | SubgraphNode type check guard in split_fan_out rejects with FanOutValidationError |
| T-40-02 | Tested | test_concurrent_branch_runner_isolation validates runner state isolation across 4 concurrent branches |
| T-40-03 | Mitigated | Guard prevents SubgraphNode in parallel branches, blocking recursive _drive() loops |

## Self-Check: PASSED
