---
phase: 33-computed-data-mappings
plan: 02
subsystem: mappings, conditions, orchestrator
tags: [transform, safe-builtins, ast-call, expression-engine, context-wiring, integration-tests]
dependency_graph:
  requires: [33-01]
  provides: [safe-builtins-evaluator, orchestrator-context-wiring, xfrm-integration-tests]
  affects: [conditions/evaluator, mappings/validator, orchestrator/runtime]
tech_stack:
  added: []
  patterns: [frozen-allowlist, ast-call-handler, runtime-vs-static-validation-split]
key_files:
  created:
    - tests/mappings/test_transform_integration.py
  modified:
    - src/zeroth/core/conditions/evaluator.py
    - src/zeroth/core/orchestrator/runtime.py
    - src/zeroth/core/mappings/validator.py
    - tests/conditions/test_evaluator.py
    - tests/mappings/test_validator.py
decisions:
  - "Safe builtins resolved in ast.Name BEFORE namespace lookup to prevent namespace poisoning"
  - "ast.Call allowed at static validation level; runtime evaluator enforces callable allowlist"
  - "Validator test updated: __import__ now rejected at runtime (ast.Call handler) not static validation"
metrics:
  duration: 9m20s
  completed: 2026-04-12
  tasks: 3
  commits: 4
  files_changed: 6
  tests_added: 35
requirements:
  - XFRM-01
  - XFRM-02
  - XFRM-03
  - XFRM-04
---

# Phase 33 Plan 02: Safe Builtins, Context Wiring & Integration Tests Summary

Safe builtins (len, str, int, float, bool, abs, min, max, round, sorted) added to _SafeEvaluator via frozen allowlist with ast.Call handler; orchestrator wires full context namespace (payload, state, variables, visit counts, path, metadata) to mapping executor; 20 integration tests prove all XFRM requirements end-to-end.

## Task Results

| Task | Name | Commit | Key Changes |
|------|------|--------|-------------|
| 1 (TDD) | Safe builtins in _SafeEvaluator | a2192bb, 2664e55 | _SAFE_BUILTINS frozenset, _SAFE_BUILTIN_MAP, ast.Call handler, ast.Name builtin-first resolution |
| 2 | Context wiring + validator update | 4c6e3e9 | context_ns construction in _queue_next_nodes, ast.Call/ast.keyword in validator allowed set |
| 3 (TDD) | Integration tests | 8bf0789 | 20 tests covering XFRM-01 through XFRM-04 plus backward compatibility |

## What Changed

### _SafeEvaluator (evaluator.py)

- Added `_SAFE_BUILTINS` frozenset (10 entries) and `_SAFE_BUILTIN_MAP` dict mapping names to actual Python builtins
- `ast.Name` case resolves safe builtin names BEFORE namespace lookup, preventing namespace poisoning (T-33-06 mitigation)
- New `ast.Call` case: resolves function via `_visit`, checks against `_SAFE_BUILTIN_MAP.values()`, rejects non-allowlisted with clear error, evaluates args/kwargs, wraps exceptions in `ConditionEvaluationError`

### Orchestrator (runtime.py)

- `_queue_next_nodes` now builds `context_ns` dict with payload, state, variables, node_visit_counts, edge_visit_counts, path, and metadata (run_id only)
- Passes `context=context_ns` to `mapping_executor.execute()`

### Validator (validator.py)

- Added `ast.Call` and `ast.keyword` to `_allowed_expression_nodes` set
- Static validation now permits function call syntax; runtime evaluator enforces which callables are safe

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Updated existing __import__ test error message**
- **Found during:** Task 1
- **Issue:** Existing test `test_condition_evaluator_rejects_unsupported_calls` expected "unsupported expression node" but `__import__` now hits the `ast.Call` handler first with "not an allowed safe builtin"
- **Fix:** Updated match pattern to "not an allowed safe builtin" -- the security behavior is identical (rejection), only the error path changed
- **Files modified:** tests/conditions/test_evaluator.py
- **Commit:** a2192bb

**2. [Rule 1 - Bug] Updated validator test for ast.Call allowance**
- **Found during:** Task 2
- **Issue:** Existing test `test_rejects_unsupported_ast_node_call` expected ast.Call to be rejected at static validation, but we intentionally allow it now
- **Fix:** Replaced with `test_allows_call_nodes_in_static_validation` that verifies `len(payload.items)` passes validation
- **Files modified:** tests/mappings/test_validator.py
- **Commit:** 4c6e3e9

**3. [Rule 3 - Blocking] Soft reset from worktree branch correction included unintended file deletions**
- **Found during:** Initial setup
- **Issue:** The worktree had staged deletions; `git reset --soft` to correct base commit bundled them into the first commit, reverting Plan 01 work
- **Fix:** Hard reset to base commit and restarted cleanly
- **Files affected:** All Plan 01 files (restored to correct state)

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| Safe builtins resolved before namespace in ast.Name | Prevents namespace poisoning where a user could shadow `len` with a malicious value in payload |
| ast.Call allowed at static validation, enforced at runtime | Clean separation: validator catches syntax/structure errors, evaluator enforces security policy |
| Updated existing tests instead of adding workarounds | Error paths changed legitimately; tests should reflect actual behavior |

## Verification

- 122 relevant tests pass (conditions, mappings, orchestrator, graph, runs)
- 20 new integration tests covering all 4 XFRM requirements
- 15 new safe-builtins unit tests (10 builtins + 4 rejections + 1 import)
- Ruff lint: all checks passed
- Ruff format: all 13 files formatted

## Self-Check: PASSED

All 7 key files verified present. All 4 commit hashes found in git log.
