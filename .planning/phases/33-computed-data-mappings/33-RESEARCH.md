# Phase 33: Computed Data Mappings - Research

**Researched:** 2026-04-12
**Domain:** Edge data transformation via safe expression evaluation (Python AST)
**Confidence:** HIGH

## Summary

Phase 33 adds a fifth mapping operation type (`transform`) to the existing edge mapping system. The implementation is tightly constrained: the expression engine (`_SafeEvaluator`), the mapping model hierarchy (Pydantic discriminated union), the executor (match/case dispatch), and the validator all already exist and follow clear extension patterns. The new operation evaluates a Python expression against a namespace (payload, state, variables, etc.) and writes the computed result to a target path.

The primary complexity is not in the transform operation itself -- it is in **threading the expression namespace through the mapping executor**. Today, `MappingExecutor.execute()` only receives `payload` (the raw output data from the previous node). Transform expressions need access to `state.*`, `variables.*`, `node_visit_counts`, etc. (per XFRM-02 / D-02). The executor's signature must be extended with an optional `context` parameter without breaking existing callers. The orchestrator's `_queue_next_nodes` method (the sole call site) has access to the `Run` object and can construct the context.

**Primary recommendation:** Extend `MappingExecutor.execute()` with an optional `context: Mapping[str, Any] | None = None` parameter. Add `TransformMappingOperation` to the model union. Add a `case TransformMappingOperation()` branch to both executor and validator. Add `MappingExecutionError` to the errors module. No new files, no new dependencies.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Transform expressions use the existing `_SafeEvaluator` from `zeroth.core.conditions.evaluator` -- no new parser, no pipe syntax. The `| length` example in requirements is illustrative; actual expressions use Python syntax (e.g., `payload.score * 100`, `payload.status == 'done'`).
- **D-02:** Transform expressions access the same namespace as condition expressions: `payload`, `state`, `variables`, `node_visit_counts`, `edge_visit_counts`, `path`, `metadata` -- consistent with XFRM-02 requirement ("same syntax as condition expressions").
- **D-03:** If safe built-in functions (e.g., `len`, `str`, `int`, `float`, `bool`, `abs`, `min`, `max`, `round`, `sorted`) are needed for transform use cases, they may be added to the `_SafeEvaluator` as a safe builtins allowlist. This is Claude's discretion based on what's needed during implementation -- the key constraint is no side effects.
- **D-04:** When a transform expression fails at runtime (syntax error, type error, division by zero, missing attribute), a `MappingExecutionError` is raised -- fail loudly. Silent failures would hide bugs in graph definitions, conflicting with the platform's governance-first philosophy.
- **D-05:** Expression validation at graph validation time should catch statically-detectable errors (syntax errors, unsupported AST nodes) before runtime.
- **D-06:** The new `TransformMappingOperation` model follows the existing discriminated union pattern -- `operation: Literal["transform"]` with an `expression: str` field and the inherited `target_path` from `MappingOperationBase`.
- **D-07:** The `MappingOperation` union, `MappingExecutor`, and `MappingValidator` are extended to handle the new operation type. No new files needed -- this fits cleanly into the existing module structure.
- **D-08:** The orchestrator's mapping execution path (`runtime.py:448`) needs no changes -- it already calls `mapping_executor.execute()` which will handle the new operation type after the executor is updated.
- **D-09:** Per XFRM-03, the output of a transform expression is validated against the target node's input contract. This happens at the same point existing mapping outputs are validated -- no special handling needed beyond ensuring the transform result flows through the existing validation pipeline.
- **D-10:** Per XFRM-04, transform expressions are guaranteed side-effect-free by the `_SafeEvaluator`'s existing restrictions: no function calls (unless safe builtins added per D-03), no imports, no dunder attribute traversal, no network/filesystem access. The AST allowlist enforces this structurally.

### Claude's Discretion
- Whether to add safe builtins to `_SafeEvaluator` (D-03) -- decide based on what transform use cases actually need
- Whether to add a `source_path` field to `TransformMappingOperation` (for convenience like `expression` operating on a pre-extracted value) vs requiring full path in expression
- Test structure and organization within existing test patterns

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| XFRM-01 | A new transform mapping operation evaluates an expression against the source payload and writes the result to the target path | `TransformMappingOperation` model + `_apply_operation` case in executor; `_SafeEvaluator` provides evaluation; `_set_path` writes result |
| XFRM-02 | Transform expressions use the same expression evaluation engine as conditions and can access payload.*, state.*, variables.* | `_SafeEvaluator` from `zeroth.core.conditions.evaluator` + `ConditionContext.namespace()` provide the identical namespace; executor needs optional context parameter |
| XFRM-03 | The output of a transform expression is validated against the target node's input contract | Existing contract validation pipeline at orchestrator level handles this; transform results flow through `_set_path` into the output dict which is validated downstream |
| XFRM-04 | Transform expressions are guaranteed side-effect-free | `_SafeEvaluator` AST allowlist structurally prevents function calls (ast.Call not in match cases), imports, dunder access, network/filesystem; adding safe builtins requires extending the `_visit` method with an `ast.Call` handler restricted to an allowlist |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Python `ast` | stdlib | AST parsing for safe expression evaluation | Already used by `_SafeEvaluator`; zero dependencies [VERIFIED: codebase] |
| Pydantic | (existing) | Model definitions, discriminated unions, validation | Already used for all mapping models [VERIFIED: codebase] |

### Supporting
No new libraries required. All implementation uses existing codebase modules. [VERIFIED: codebase grep]

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `_SafeEvaluator` (AST-based) | `simpleeval` library | External dependency; our evaluator is already built, tested, and tailored to our security model |
| Python AST | Jinja2 SandboxedEnvironment | Jinja is for template rendering not expression evaluation; would add complexity for no gain |
| Custom pipe syntax (`payload.items \| length`) | Python syntax (`len(payload.items)`) | D-01 explicitly locks to Python syntax via existing evaluator |

**Installation:**
```bash
# No new packages needed
uv sync  # existing deps sufficient
```

## Architecture Patterns

### Relevant Project Structure
```
src/zeroth/core/
├── mappings/
│   ├── __init__.py       # Public API re-exports (add TransformMappingOperation, MappingExecutionError)
│   ├── models.py         # Add TransformMappingOperation to discriminated union
│   ├── executor.py       # Add transform case + optional context parameter
│   ├── validator.py      # Add transform validation case (static expression check)
│   └── errors.py         # Add MappingExecutionError
├── conditions/
│   ├── evaluator.py      # _SafeEvaluator (import, do not modify unless adding builtins)
│   └── models.py         # ConditionContext.namespace() (reuse pattern)
├── graph/
│   ├── models.py         # Edge.mapping already typed as EdgeMapping | None
│   └── validation.py     # _validate_mapping already delegates to MappingValidator
└── orchestrator/
    └── runtime.py        # _queue_next_nodes calls mapping_executor.execute() (line 448)
```

### Pattern 1: Discriminated Union Extension
**What:** Add `TransformMappingOperation` to the existing `MappingOperation` union using `Literal["transform"]` discriminator. [VERIFIED: codebase]
**When to use:** This is the exact pattern used by all four existing operations.
**Example:**
```python
# Source: src/zeroth/core/mappings/models.py (existing pattern)
class TransformMappingOperation(MappingOperationBase):
    """Evaluate an expression and write the result to target_path."""
    operation: Literal["transform"] = "transform"
    expression: str

MappingOperation = Annotated[
    PassthroughMappingOperation
    | RenameMappingOperation
    | ConstantMappingOperation
    | DefaultMappingOperation
    | TransformMappingOperation,   # <-- add here
    Field(discriminator="operation"),
]
```

### Pattern 2: Match/Case Dispatch Extension
**What:** Add a `case TransformMappingOperation()` branch to both `MappingExecutor._apply_operation` and `MappingValidator._validate_operation`. [VERIFIED: codebase]
**When to use:** Both executor and validator use structural pattern matching for operation dispatch.
**Example:**
```python
# Source: src/zeroth/core/mappings/executor.py (existing pattern)
case TransformMappingOperation():
    evaluator = _SafeEvaluator(context_namespace)
    try:
        result = evaluator.evaluate(operation.expression)
    except ConditionEvaluationError as exc:
        raise MappingExecutionError(
            f"transform expression failed: {exc}"
        ) from exc
    _set_path(output, operation.target_path, result)
```

### Pattern 3: Optional Context Parameter (Backward Compatible)
**What:** Extend `MappingExecutor.execute()` with an optional `context` parameter so transform operations can access the full namespace while existing callers continue to work without changes. [VERIFIED: codebase analysis]
**When to use:** Required for XFRM-02 compliance.
**Example:**
```python
# Extended signature
def execute(
    self,
    payload: Mapping[str, Any],
    mapping: EdgeMapping,
    *,
    context: Mapping[str, Any] | None = None,
) -> dict[str, Any]:
```

### Pattern 4: Static Expression Validation
**What:** At graph validation time, attempt `ast.parse(expression, mode="eval")` and check that all AST nodes are in the `_SafeEvaluator` allowlist. [VERIFIED: codebase -- evaluator already does this at runtime]
**When to use:** Catches syntax errors and unsupported constructs before a graph is published, per D-05.
**Example:**
```python
# In MappingValidator._validate_operation, transform case:
import ast
try:
    tree = ast.parse(operation.expression, mode="eval")
    _check_ast_safety(tree.body)  # walk tree, verify all nodes are supported
except SyntaxError:
    raise MappingValidationError(f"invalid transform expression syntax: {operation.expression!r}")
```

### Anti-Patterns to Avoid
- **Modifying `_SafeEvaluator` internals for transform-specific behavior:** The evaluator should remain a general-purpose safe expression engine. Transform-specific logic (error wrapping, namespace construction) belongs in the mapping executor. [VERIFIED: codebase separation of concerns]
- **Passing raw `Run` object to executor:** The executor is a pure data transformation layer. It should receive a pre-built namespace dict, not a domain-specific `Run` model. This maintains the clean boundary between orchestrator and mapping modules. [VERIFIED: codebase architecture]
- **Making context mandatory on execute():** This would break all existing callers and tests. Use `context: ... | None = None` with keyword-only syntax. [VERIFIED: codebase -- 2 existing tests + runtime call site]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Expression parsing | Custom tokenizer/parser | `_SafeEvaluator` + Python `ast` | Already built, tested, and security-hardened [VERIFIED: codebase] |
| Namespace construction | Custom dict builder | `ConditionContext.namespace()` pattern | Already defines the canonical namespace shape [VERIFIED: codebase] |
| Dot-path get/set | Custom path walker | `_get_path` / `_set_path` in executor | Already handles nested dicts with auto-creation [VERIFIED: codebase] |
| AST safety validation | Runtime-only checks | Static `ast.parse` + node-type walk | Catches errors at validation time, not runtime [VERIFIED: D-05 decision] |

**Key insight:** This phase is almost entirely about wiring existing pieces together. The expression engine, the mapping model hierarchy, the executor dispatch pattern, and the validator structure are all in place. The new code is a thin integration layer.

## Common Pitfalls

### Pitfall 1: Breaking Existing Executor Callers
**What goes wrong:** Making `context` a required parameter on `execute()` breaks the runtime call site and all tests.
**Why it happens:** Easy to overlook that the parameter must be backward-compatible.
**How to avoid:** Use `*, context: Mapping[str, Any] | None = None` (keyword-only, optional). Build a default namespace from just `payload` when context is None.
**Warning signs:** Existing tests in `tests/mappings/test_executor.py` start failing.

### Pitfall 2: Evaluator Error Types Leaking
**What goes wrong:** `ConditionEvaluationError` propagates from transform evaluation, confusing error handlers that expect mapping errors.
**Why it happens:** `_SafeEvaluator` raises `ConditionEvaluationError` -- this is the wrong domain for mapping operations.
**How to avoid:** Catch `ConditionEvaluationError` in the executor's transform case and re-raise as `MappingExecutionError` with the original message. Chain exceptions with `from exc`.
**Warning signs:** Error handling tests catching `MappingExecutionError` fail because the wrong exception type is raised.

### Pitfall 3: Namespace Construction When Context Is None
**What goes wrong:** Transform expressions fail with `None` lookups when no context is provided (e.g., in unit tests that don't pass context).
**Why it happens:** If context is None, the namespace must still include `payload` so expressions like `payload.score * 100` work.
**How to avoid:** When `context is None`, build a minimal namespace: `{"payload": dict(payload), "state": {}, "variables": {}, ...}`. This mirrors the `ConditionContext.namespace()` shape with empty defaults.
**Warning signs:** Transform tests pass when context is provided but fail in isolation.

### Pitfall 4: Silent None Returns from Missing Paths
**What goes wrong:** `_SafeEvaluator` returns `None` for missing attributes (by design -- see `ast.Attribute` case), so `payload.nonexistent.field` silently produces `None` instead of erroring.
**Why it happens:** The evaluator was designed for conditions where `None` is a valid falsy result. For transforms, writing `None` to a target path may violate the target node's input contract.
**How to avoid:** This is actually correct behavior per the existing design -- contract validation (XFRM-03) catches the `None` downstream. Document this as expected. Do NOT add null-safety checks to the evaluator itself.
**Warning signs:** Users expect transform expressions to raise on missing paths, but they produce `None` instead.

### Pitfall 5: Static Validation Over-Restricting Expressions
**What goes wrong:** The static AST validator rejects expressions that are actually safe at runtime (e.g., ternary expressions, complex comparisons).
**Why it happens:** Implementing a separate AST walk that doesn't exactly match `_SafeEvaluator`'s supported nodes.
**How to avoid:** Extract the set of supported AST node types directly from `_SafeEvaluator._visit`'s match cases. Use the same set in both runtime and static validation. Consider a shared constant or utility function.
**Warning signs:** Valid expressions pass at runtime but fail static validation (or vice versa).

### Pitfall 6: Adding Safe Builtins Without ast.Call Support
**What goes wrong:** Adding `len`, `str`, etc. to the namespace without also handling `ast.Call` nodes in `_SafeEvaluator._visit` means expressions like `len(payload.items)` raise "unsupported expression node: Call".
**Why it happens:** The evaluator currently has no `case ast.Call()` branch -- function calls are structurally blocked. Adding builtins to the namespace alone is insufficient.
**How to avoid:** If safe builtins are added (D-03), also add an `ast.Call` case that: (1) evaluates the function reference, (2) checks it's in the safe builtins allowlist, (3) evaluates arguments, (4) calls the function. Reject all other calls.
**Warning signs:** Tests defining `len(payload.items)` fail with "unsupported expression node".

## Code Examples

### Example 1: TransformMappingOperation Model
```python
# Source: follows pattern in src/zeroth/core/mappings/models.py
class TransformMappingOperation(MappingOperationBase):
    """Evaluate an expression against the source namespace and write the result.

    The expression is evaluated using the safe AST-based evaluator. It can
    reference ``payload.*``, ``state.*``, ``variables.*``, and other namespace
    entries available to condition expressions.
    """

    operation: Literal["transform"] = "transform"
    expression: str
```

### Example 2: Executor Transform Case
```python
# Source: follows pattern in src/zeroth/core/mappings/executor.py
from zeroth.core.conditions.evaluator import _SafeEvaluator
from zeroth.core.conditions.errors import ConditionEvaluationError
from zeroth.core.mappings.errors import MappingExecutionError

# In _apply_operation:
case TransformMappingOperation():
    namespace = self._build_namespace(payload, context)
    evaluator = _SafeEvaluator(namespace)
    try:
        result = evaluator.evaluate(operation.expression)
    except ConditionEvaluationError as exc:
        raise MappingExecutionError(
            f"transform expression failed on '{operation.expression}': {exc}"
        ) from exc
    _set_path(output, operation.target_path, result)
```

### Example 3: MappingExecutionError
```python
# Source: follows pattern in src/zeroth/core/mappings/errors.py
class MappingExecutionError(ValueError):
    """Raised when a mapping operation fails during execution.

    For example, this is raised when a transform expression encounters a
    division by zero, type error, or references an unsupported AST construct.
    """
```

### Example 4: Static Expression Validation in Validator
```python
# Source: follows pattern in src/zeroth/core/mappings/validator.py
import ast

# In _validate_operation:
case TransformMappingOperation():
    _validate_path(operation.target_path, label="target_path")
    self._check_target_path(operation.target_path, target_paths)
    self._validate_expression(operation.expression)

def _validate_expression(self, expression: str) -> None:
    """Check that an expression is syntactically valid and uses only safe AST nodes."""
    if not expression or not expression.strip():
        raise MappingValidationError("transform expression must not be empty")
    try:
        tree = ast.parse(expression, mode="eval")
    except SyntaxError as exc:
        raise MappingValidationError(
            f"invalid transform expression syntax: {expression!r}"
        ) from exc
    self._check_ast_nodes(tree.body, expression)
```

### Example 5: Namespace Builder in Executor
```python
# Source: follows ConditionContext.namespace() pattern from conditions/models.py
def _build_namespace(
    self,
    payload: Mapping[str, Any],
    context: Mapping[str, Any] | None,
) -> dict[str, Any]:
    """Build the expression namespace from payload and optional context."""
    if context is not None:
        return dict(context)
    # Minimal namespace when no context provided (backward compat / unit tests)
    return {
        "payload": dict(payload),
        "state": {},
        "variables": {},
        "node_visit_counts": {},
        "edge_visit_counts": {},
        "path": [],
        "metadata": {},
    }
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Raw payload passthrough only | 4 mapping operations (passthrough, rename, constant, default) | Phase 7 (v1.0) | Basic data reshaping between nodes |
| No computed values in mappings | Transform operation (this phase) | Phase 33 (v4.0) | Derived values without intermediate nodes |

**Not deprecated/outdated:** All four existing operations remain unchanged and fully supported. The transform operation is additive. [VERIFIED: XFRM success criteria #4]

## Key Architectural Insight: Context Threading

The most important technical detail for planning this phase:

**Current state:** `MappingExecutor.execute(payload, mapping)` at `runtime.py:448` passes only `output_data` as the payload. [VERIFIED: codebase]

**Required state:** Transform expressions need `state.*`, `variables.*`, visit counts, etc. per D-02. [VERIFIED: CONTEXT.md]

**Resolution:** D-08 says "the orchestrator's mapping execution path needs no changes." This is technically accurate if we interpret it as "the call site structure stays the same" -- but the executor's `execute()` method signature DOES need to accept an optional context parameter, and the call site at `runtime.py:448` needs to pass context when available. This is a minimal, backward-compatible change:

```python
# runtime.py:448 (before)
payload = self.mapping_executor.execute(output_data, edge.mapping)

# runtime.py:448 (after) -- only changes if transform operations are present
context_ns = {
    "payload": dict(output_data),
    "state": dict(run.metadata.get("state", {})),
    "variables": dict(run.metadata.get("variables", {})),
    "node_visit_counts": dict(run.node_visit_counts),
    "edge_visit_counts": dict(run.metadata.get("edge_visit_counts", {})),
    "path": list(run.metadata.get("path", [])),
    "metadata": {"run_id": run.run_id},
}
payload = self.mapping_executor.execute(output_data, edge.mapping, context=context_ns)
```

**Note:** The current runtime does not populate `state` or `variables` in the `ConditionContext` either (they default to `{}`). This is consistent -- both conditions and transforms will access whatever `state`/`variables` are available. The mechanism exists; it's just not wired to external state yet. This is outside the scope of Phase 33. [VERIFIED: codebase -- runtime.py:408-414]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `run.metadata` can carry `state` and `variables` for the context namespace | Key Architectural Insight | If Run has no way to carry these, context construction at the call site would need a different data source. LOW risk -- metadata is a `dict[str, Any]` that already carries arbitrary data. |
| A2 | Contract validation of transform output happens downstream of mapping execution (at the orchestrator level), not within the mapping executor | Phase Requirements (XFRM-03) | If contract validation must happen inside the executor, the executor would need access to contract definitions. LOW risk -- D-09 explicitly states "no special handling needed." |

**If this table is empty:** N/A -- two assumptions flagged above.

## Open Questions

1. **Should the runtime call site always pass context, or only when transform operations exist?**
   - What we know: Passing context is cheap (dict construction). Transform detection requires inspecting all operations.
   - What's unclear: Whether always passing context is preferable for simplicity vs. lazy construction.
   - Recommendation: Always pass context. The cost is negligible and it simplifies the code. The planner should decide.

2. **Should safe builtins (len, str, int, etc.) be added in this phase?**
   - What we know: D-03 makes this Claude's discretion. `_SafeEvaluator` currently has NO `ast.Call` handler, so adding builtins requires also adding Call support.
   - What's unclear: Whether transform use cases actually need builtins. Without `len()`, users cannot compute list lengths. Without `str()`/`int()`, they cannot cast types.
   - Recommendation: Add a minimal set (`len`, `str`, `int`, `float`, `bool`, `abs`, `min`, `max`, `round`, `sorted`) because transform expressions without these are severely limited. The `ast.Call` handler should be restricted to a frozen set of allowed callables. This is the only part of the phase that touches `_SafeEvaluator` internals.

## Project Constraints (from CLAUDE.md)

- **Build/test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** Source in `src/zeroth/`, tests in `tests/`
- **Progress logging:** Every implementation session must use the `progress-logger` skill
- **Context efficiency:** Read only relevant phase files, not all phases

## Sources

### Primary (HIGH confidence)
- `src/zeroth/core/mappings/models.py` -- All 4 existing operation models, discriminated union, EdgeMapping [VERIFIED: codebase]
- `src/zeroth/core/mappings/executor.py` -- MappingExecutor, _get_path, _set_path, match/case dispatch [VERIFIED: codebase]
- `src/zeroth/core/mappings/validator.py` -- MappingValidator, _validate_path, _validate_operation, match/case dispatch [VERIFIED: codebase]
- `src/zeroth/core/mappings/errors.py` -- MappingValidationError (MappingExecutionError does not exist yet) [VERIFIED: codebase]
- `src/zeroth/core/conditions/evaluator.py` -- _SafeEvaluator (full AST-based evaluator), ConditionEvaluator [VERIFIED: codebase]
- `src/zeroth/core/conditions/models.py` -- ConditionContext.namespace() defines canonical namespace shape [VERIFIED: codebase]
- `src/zeroth/core/conditions/errors.py` -- ConditionEvaluationError [VERIFIED: codebase]
- `src/zeroth/core/graph/models.py` -- Edge model with mapping field (line 244-259) [VERIFIED: codebase]
- `src/zeroth/core/graph/validation.py` -- GraphValidator._validate_mapping delegates to MappingValidator [VERIFIED: codebase]
- `src/zeroth/core/graph/validation_errors.py` -- ValidationCode.INVALID_MAPPING exists [VERIFIED: codebase]
- `src/zeroth/core/orchestrator/runtime.py` -- mapping_executor.execute() call at line 448, ConditionContext construction at line 408 [VERIFIED: codebase]
- `tests/mappings/test_executor.py` -- 2 existing executor tests [VERIFIED: codebase]
- `tests/mappings/test_validator.py` -- 3 existing validator tests [VERIFIED: codebase]
- `tests/conditions/test_evaluator.py` -- 3 existing evaluator tests including security rejection [VERIFIED: codebase]
- `governai.RunState` -- Base class for Run, has metadata dict[str, Any] [VERIFIED: runtime inspection]

### Secondary (MEDIUM confidence)
- None

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- zero new dependencies, all existing modules verified via codebase read
- Architecture: HIGH -- all integration points verified, all patterns confirmed from source
- Pitfalls: HIGH -- derived from actual codebase analysis (e.g., missing ast.Call handler, context threading)

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- no external dependencies or fast-moving ecosystem concerns)
