# Phase 33: Computed Data Mappings - Context

**Gathered:** 2026-04-12
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a `transform` mapping operation to the existing edge mapping system. Transform mappings evaluate an expression (using the existing `_SafeEvaluator`) against the source payload and write the computed result to a target path on the next node's input. The four existing operations (passthrough, rename, constant, default) must continue working unchanged.

</domain>

<decisions>
## Implementation Decisions

### Expression Engine
- **D-01:** Transform expressions use the existing `_SafeEvaluator` from `zeroth.core.conditions.evaluator` — no new parser, no pipe syntax. The `| length` example in requirements is illustrative; actual expressions use Python syntax (e.g., `payload.score * 100`, `payload.status == 'done'`).
- **D-02:** Transform expressions access the same namespace as condition expressions: `payload`, `state`, `variables`, `node_visit_counts`, `edge_visit_counts`, `path`, `metadata` — consistent with XFRM-02 requirement ("same syntax as condition expressions").
- **D-03:** If safe built-in functions (e.g., `len`, `str`, `int`, `float`, `bool`, `abs`, `min`, `max`, `round`, `sorted`) are needed for transform use cases, they may be added to the `_SafeEvaluator` as a safe builtins allowlist. This is Claude's discretion based on what's needed during implementation — the key constraint is no side effects.

### Error Handling
- **D-04:** When a transform expression fails at runtime (syntax error, type error, division by zero, missing attribute), a `MappingExecutionError` is raised — fail loudly. Silent failures would hide bugs in graph definitions, conflicting with the platform's governance-first philosophy.
- **D-05:** Expression validation at graph validation time should catch statically-detectable errors (syntax errors, unsupported AST nodes) before runtime.

### Model & Integration
- **D-06:** The new `TransformMappingOperation` model follows the existing discriminated union pattern — `operation: Literal["transform"]` with an `expression: str` field and the inherited `target_path` from `MappingOperationBase`.
- **D-07:** The `MappingOperation` union, `MappingExecutor`, and `MappingValidator` are extended to handle the new operation type. No new files needed — this fits cleanly into the existing module structure.
- **D-08:** The orchestrator's mapping execution path (`runtime.py:448`) needs no changes — it already calls `mapping_executor.execute()` which will handle the new operation type after the executor is updated.

### Contract Validation
- **D-09:** Per XFRM-03, the output of a transform expression is validated against the target node's input contract. This happens at the same point existing mapping outputs are validated — no special handling needed beyond ensuring the transform result flows through the existing validation pipeline.

### Side-Effect Safety
- **D-10:** Per XFRM-04, transform expressions are guaranteed side-effect-free by the `_SafeEvaluator`'s existing restrictions: no function calls (unless safe builtins added per D-03), no imports, no dunder attribute traversal, no network/filesystem access. The AST allowlist enforces this structurally.

### Claude's Discretion
- Whether to add safe builtins to `_SafeEvaluator` (D-03) — decide based on what transform use cases actually need
- Whether to add a `source_path` field to `TransformMappingOperation` (for convenience like `expression` operating on a pre-extracted value) vs requiring full path in expression
- Test structure and organization within existing test patterns

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Mapping System
- `src/zeroth/core/mappings/models.py` — Defines `MappingOperationBase`, all 4 existing operations, `MappingOperation` discriminated union, `EdgeMapping`
- `src/zeroth/core/mappings/executor.py` — `MappingExecutor` with match/case dispatch and `_get_path`/`_set_path` helpers
- `src/zeroth/core/mappings/validator.py` — `MappingValidator` with path validation and duplicate target check
- `src/zeroth/core/mappings/errors.py` — `MappingValidationError`
- `src/zeroth/core/mappings/__init__.py` — Public API re-exports

### Expression Engine
- `src/zeroth/core/conditions/evaluator.py` — `_SafeEvaluator` class (AST-based safe expression evaluation) and `ConditionEvaluator`
- `src/zeroth/core/conditions/models.py` — `ConditionContext` with `namespace()` method defining available variables

### Graph & Orchestrator Integration
- `src/zeroth/core/graph/models.py` — `Edge` model (line 244) with `mapping: EdgeMapping | None`
- `src/zeroth/core/orchestrator/runtime.py` — Mapping execution at line 448, `MappingExecutor` injected at line 96
- `src/zeroth/core/graph/validation.py` — Graph validation (may need update for static expression validation)

### Tests
- `tests/mappings/test_executor.py` — Existing executor tests (patterns to follow)
- `tests/mappings/test_validator.py` — Existing validator tests

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `_SafeEvaluator` — Full AST-based expression engine already built and tested. Supports: constants, names, attributes, subscripts, lists, tuples, dicts, sets, boolean ops, unary ops, binary ops (add/sub/mult/div/mod), comparisons (all), ternary (if/else). This is the core engine for transforms.
- `ConditionContext.namespace()` — Ready-made namespace builder that exposes `payload`, `state`, `variables` and more.
- `MappingOperationBase` — Base class with `target_path` field that the new operation inherits from.
- `_get_path` / `_set_path` — Dot-path utilities in executor, reusable if transform needs source path extraction.

### Established Patterns
- **Discriminated union** — `MappingOperation` uses `Field(discriminator="operation")` with `Literal` type per operation. New operation follows this exactly.
- **Match/case dispatch** — Both `MappingExecutor._apply_operation` and `MappingValidator._validate_operation` use structural pattern matching. Add a new case for `TransformMappingOperation`.
- **Validation before execution** — `MappingExecutor.execute()` calls `validator.validate()` before applying operations.
- **Pydantic ConfigDict(extra="forbid")** — All models use strict validation.

### Integration Points
- `MappingOperation` union in `models.py:74-80` — Add `TransformMappingOperation` to the union
- `MappingExecutor._apply_operation` in `executor.py:78-110` — Add transform case
- `MappingValidator._validate_operation` in `validator.py:56-81` — Add transform validation case
- `__init__.py` exports — Add new operation to public API
- `graph/validation.py` — Possibly add static expression syntax validation for transform expressions

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. The implementation is well-constrained by XFRM-01 through XFRM-04 and the existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 33-computed-data-mappings*
*Context gathered: 2026-04-12*
