# Phase 33: Computed Data Mappings - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-12
**Phase:** 33-computed-data-mappings
**Mode:** --auto (all decisions auto-selected with recommended defaults)
**Areas discussed:** Expression namespace, Error handling, Pipe/filter syntax

---

## Expression Namespace

| Option | Description | Selected |
|--------|-------------|----------|
| Same as conditions | payload, state, variables, visit counts, path, metadata — consistent with XFRM-02 | ✓ |
| Restricted subset | Only payload and variables — simpler but limits expressiveness | |
| Extended namespace | Add node metadata, graph info — more powerful but scope creep | |

**User's choice:** [auto] Same as conditions (recommended default)
**Notes:** XFRM-02 explicitly requires "same syntax as condition expressions" — using the same namespace is the natural interpretation.

---

## Error Handling

| Option | Description | Selected |
|--------|-------------|----------|
| Raise MappingExecutionError | Fail loudly on expression errors — consistent with governance philosophy | ✓ |
| Return None and skip | Silent fallback — hides bugs in graph definitions | |
| Configurable per-operation | Add an `on_error` field — flexibility but complexity | |

**User's choice:** [auto] Raise MappingExecutionError (recommended default)
**Notes:** The platform's governance-first philosophy favors explicit failures over silent data corruption. Graph authors should catch errors during development, not discover them in production audit trails.

---

## Pipe/Filter Syntax

| Option | Description | Selected |
|--------|-------------|----------|
| No pipes, Python syntax only | Use existing _SafeEvaluator as-is — XFRM-02 says "same engine" | ✓ |
| Add Jinja2-style pipe filters | New parser for `| length`, `| upper` etc — significant complexity | |
| Add safe builtins only | Allow len(), str(), int() etc as function calls in expressions | |

**User's choice:** [auto] No pipes, Python syntax only (recommended default)
**Notes:** The `| length` example in the XFRM-01 requirement description is illustrative, not a literal syntax requirement. XFRM-02 locks the engine to the existing condition evaluator. Safe builtins (len, str, etc.) may be added at Claude's discretion if transform use cases need them (see D-03).

---

## Claude's Discretion

- Whether to add safe builtins to `_SafeEvaluator`
- Whether `TransformMappingOperation` needs a `source_path` convenience field
- Test structure and organization

## Deferred Ideas

None.
