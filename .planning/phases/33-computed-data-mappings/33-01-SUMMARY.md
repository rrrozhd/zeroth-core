---
phase: 33-computed-data-mappings
plan: 01
subsystem: mappings
tags: [ast, pydantic, safe-evaluator, transform, expression-engine]

# Dependency graph
requires:
  - phase: conditions
    provides: _SafeEvaluator for AST-based expression evaluation, ConditionEvaluationError
provides:
  - TransformMappingOperation model in MappingOperation discriminated union
  - MappingExecutionError for runtime transform failures
  - Static expression validation (syntax + AST allowlist) in MappingValidator
  - Transform execution via _SafeEvaluator in MappingExecutor
  - Optional context parameter on MappingExecutor.execute()
affects: [33-02-plan, orchestrator-context-wiring, edge-mapping-consumers]

# Tech tracking
tech-stack:
  added: []
  patterns: [lazy-import-for-circular-deps, ast-allowlist-validation, exception-wrapping]

key-files:
  created:
    - tests/mappings/test_transform_model.py
  modified:
    - src/zeroth/core/mappings/models.py
    - src/zeroth/core/mappings/errors.py
    - src/zeroth/core/mappings/executor.py
    - src/zeroth/core/mappings/validator.py
    - src/zeroth/core/mappings/__init__.py
    - tests/mappings/test_executor.py
    - tests/mappings/test_validator.py

key-decisions:
  - "Lazy import of _SafeEvaluator in executor to break circular dependency (mappings -> conditions -> graph -> mappings)"
  - "Catch broad Exception from evaluator (not just ConditionEvaluationError) since _SafeEvaluator can raise raw Python exceptions like ZeroDivisionError"
  - "AST allowlist includes structural nodes (Load, Index, Expression) needed by Python 3.12 AST walker"

patterns-established:
  - "Lazy import pattern: defer cross-package imports inside method body when circular dependency exists"
  - "Exception wrapping: catch Exception from reused evaluator, wrap as domain-specific MappingExecutionError"
  - "AST allowlist validation: static check at graph definition time mirrors runtime evaluator node support"

requirements-completed: [XFRM-01, XFRM-02, XFRM-03, XFRM-04]

# Metrics
duration: 8min
completed: 2026-04-12
---

# Phase 33 Plan 01: Core Transform Mapping Summary

**TransformMappingOperation model with AST-validated expressions, _SafeEvaluator-based execution, and optional context parameter on MappingExecutor**

## Performance

- **Duration:** 8 min
- **Started:** 2026-04-12T18:42:30Z
- **Completed:** 2026-04-12T18:50:11Z
- **Tasks:** 3
- **Files modified:** 8

## Accomplishments
- TransformMappingOperation added to the MappingOperation discriminated union with operation="transform" and required expression field
- MappingValidator statically validates transform expressions at graph definition time (syntax checking via ast.parse, AST node allowlist matching _SafeEvaluator support)
- MappingExecutor evaluates transform expressions at runtime via _SafeEvaluator, with optional context parameter for full namespace access (payload, state, variables, etc.)
- MappingExecutionError wraps all runtime evaluation failures (division by zero, type errors, unsupported nodes)
- 43 mapping tests pass (38 new + 5 existing), zero regressions across conditions and graph subsystems

## Task Commits

Each task was committed atomically using TDD (RED then GREEN):

1. **Task 1: TransformMappingOperation model + MappingExecutionError**
   - `26851de` (test: failing tests for model and error)
   - `2345cfc` (feat: model, error, and exports)
2. **Task 2: Transform validation with static expression checking**
   - `b59f046` (test: failing tests for validation)
   - `49de66a` (feat: validator with AST allowlist)
3. **Task 3: Transform execution with optional context**
   - `43b895a` (test: failing tests for execution)
   - `c072cc2` (feat: executor with _SafeEvaluator integration)

## Files Created/Modified
- `src/zeroth/core/mappings/models.py` - Added TransformMappingOperation class, updated MappingOperation union
- `src/zeroth/core/mappings/errors.py` - Added MappingExecutionError (ValueError subclass)
- `src/zeroth/core/mappings/validator.py` - Added transform case with _validate_expression and _check_ast_safety
- `src/zeroth/core/mappings/executor.py` - Added transform case with _SafeEvaluator, optional context param, _build_namespace
- `src/zeroth/core/mappings/__init__.py` - Added TransformMappingOperation and MappingExecutionError to public API
- `tests/mappings/test_transform_model.py` - 13 tests for model, union, error, exports
- `tests/mappings/test_validator.py` - Added 13 transform validation tests
- `tests/mappings/test_executor.py` - Added 12 transform execution tests

## Decisions Made
- **Lazy import for circular dependency:** `from zeroth.core.conditions.evaluator import _SafeEvaluator` placed inside the transform case method body rather than at module level, because the import chain mappings -> conditions.__init__ -> binding -> models -> graph.models -> mappings.models creates a circular import. This is the standard Python pattern for breaking import cycles.
- **Broad exception catch:** The executor catches `Exception` (not just `ConditionEvaluationError`) because `_SafeEvaluator._visit` can raise raw Python exceptions (ZeroDivisionError, TypeError) from arithmetic/comparison operations that are not wrapped by the evaluator itself.
- **AST structural nodes in allowlist:** Added `ast.Load`, `ast.Index`, `ast.Expression` to the validator's allowed node set because `ast.walk()` visits these structural nodes that appear as children in valid expression trees.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Circular import when importing from conditions package**
- **Found during:** Task 3 (Transform execution)
- **Issue:** Top-level `from zeroth.core.conditions.errors import ConditionEvaluationError` triggered `zeroth.core.conditions.__init__` which imports binding -> models -> graph.models -> mappings.models, creating circular import
- **Fix:** Moved conditions imports to lazy (inside method body), only triggered when transform operations are actually executed
- **Files modified:** src/zeroth/core/mappings/executor.py
- **Verification:** All 43 mapping tests pass, all 78 core subsystem tests pass
- **Committed in:** c072cc2

**2. [Rule 1 - Bug] _SafeEvaluator raises raw Python exceptions, not ConditionEvaluationError**
- **Found during:** Task 3 (Transform execution)
- **Issue:** Division by zero in expression raised ZeroDivisionError directly, not ConditionEvaluationError as the plan assumed
- **Fix:** Changed except clause from `ConditionEvaluationError` to broad `Exception` catch, all wrapped as MappingExecutionError
- **Files modified:** src/zeroth/core/mappings/executor.py
- **Verification:** test_transform_division_by_zero_raises_mapping_execution_error passes
- **Committed in:** c072cc2

---

**Total deviations:** 2 auto-fixed (1 blocking circular import, 1 bug in exception handling)
**Impact on plan:** Both auto-fixes necessary for correctness. No scope creep.

## Issues Encountered
None beyond the auto-fixed deviations above.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Plan 02 (safe builtins, orchestrator context wiring, integration tests) can proceed immediately
- Plan 02 will add ast.Call to the validator's allowlist when safe builtins are introduced
- The optional `context` parameter on `execute()` is ready for orchestrator wiring in Plan 02

---
*Phase: 33-computed-data-mappings*
*Completed: 2026-04-12*
