---
phase: 44-declarative-template-memory-sourcing
plan: 01
subsystem: orchestration
tags: [orchestration, memory, templates, agent, injection]
requirements: [TMB-01, TMB-02, TMB-03, TMB-04, TMB-05, TMB-06]
requirements_addressed: [TMB-01, TMB-02, TMB-03, TMB-04, TMB-05, TMB-06]
dependency_graph:
  requires:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/graph/__init__.py
    - src/zeroth/core/orchestrator/runtime.py
    - src/zeroth/core/memory/registry.py
    - src/zeroth/core/templates/ (Phase 36)
  provides:
    - TemplateMemoryBinding Pydantic model
    - AgentNodeData.template_memory_bindings field
    - MemoryBindingResolutionError(OrchestratorError)
    - _substitute_tmb_key() module helper
    - RuntimeOrchestrator._resolve_template_memory()
    - updated _dispatch_node with pre-template thread_id + memory resolution
  affects:
    - All callers of AgentNodeData (backward-compatible; field defaults to [])
tech_stack:
  added: []
  patterns:
    - declarative memory namespace injection before template rendering
    - {input.field}/{state.field}/{run.run_id} key placeholder substitution
    - scan mode via empty-query search + client-side prefix filter
    - per-binding audit record in execution_metadata
key_files:
  created:
    - tests/orchestrator/test_template_memory_bindings.py
  modified:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/graph/__init__.py
    - src/zeroth/core/orchestrator/runtime.py
decisions:
  - id: D-05
    summary: "thread_id resolved before template block so THREAD-scoped bindings resolve correctly"
  - id: D-07
    summary: "MemoryBindingResolutionError subclasses OrchestratorError (not NodeDispatcherError)"
  - id: D-09
    summary: "MemoryScope imported inside _resolve_template_memory (lazy, matches Phase 36 pattern)"
metrics:
  tasks_completed: 3
  tests_added: 13
  files_created: 1
  files_modified: 3
---

# Phase 44 Plan 01: Declarative Template Memory Sourcing — Summary

One-liner: Added `TemplateMemoryBinding` model and `_resolve_template_memory` orchestrator method so agent nodes can declaratively pull memory connector values into the `{{ memory.* }}` template namespace at dispatch time, without extra graph nodes.

## What Was Built

### Task 1 — Data model

**`TemplateMemoryBinding`** added to `src/zeroth/core/graph/models.py` before `AgentNodeData`:

- `as_name: str` — key in the `memory` namespace
- `connector_instance_id: str` — must match a name in `AgentNodeData.memory_refs`
- `access_mode: Literal["get", "scan"]` — single-key or prefix-scan
- `key: str | None` — required in get mode; supports `{input.field}` placeholders
- `key_prefix: str | None` — optional prefix filter for scan mode
- `default: Any` — returned when get key is absent or scan returns nothing
- `max_items: int | None` — truncates scan results
- `scope: Literal["run", "thread", "shared"]` — which scope the ScopedMemoryConnector resolves to
- `@model_validator` enforces `key` is required for `access_mode="get"`

`AgentNodeData.template_memory_bindings: list[TemplateMemoryBinding]` added (default `[]`, backward-compatible).

`TemplateMemoryBinding` exported from `zeroth.core.graph.__init__`.

### Task 2 — Orchestrator wiring

**`src/zeroth/core/orchestrator/runtime.py`** changes:

- `import re` added at module level.
- `MemoryBindingResolutionError(OrchestratorError)` added — raised when a connector is unregistered or a read/search call fails.
- `_KEY_PLACEHOLDER_RE = re.compile(r"\{(input|state|run)\.([^}]+)\}")` — module constant.
- `_substitute_tmb_key(key, *, input_payload, state, run_id) -> str` — module helper that replaces `{namespace.field}` placeholders; unknown placeholders left unchanged.
- **`_dispatch_node` restructured**:
  - `thread_id = await self._resolve_thread(node, run)` moved from after the context-window block to **before** the template `if` block (D-05).
  - `tmb_audit_records: list[dict[str, Any]] = []` initialised outside the template block.
  - Inside the template block: `_memory_ns, _tmb_records = await self._resolve_template_memory(node, run, thread_id, input_payload)` called before `render_vars` is built.
  - `"memory": {}` replaced with `"memory": _memory_ns`.
  - After the `finally` block: `tmb_audit_records` appended to `audit_record["execution_metadata"]["template_memory_bindings"]` when non-empty.
- **`_resolve_template_memory(node, run, thread_id, input_payload)`** method:
  - Early-returns `({}, [])` when `bindings` is empty or `self.memory_resolver is None`.
  - Deduplicates connector refs and resolves them via `self.memory_resolver.resolve(refs_needed, ...)`.
  - For `"get"`: calls `connector.read(resolved_key, scope)` after key substitution; returns `entry.value` or `binding.default`.
  - For `"scan"`: calls `connector.search({}, scope)` (empty query = all entries), filters by prefix, strips prefix from keys, applies `max_items` truncation.
  - Wraps unknown-connector `KeyError` and connector exceptions in `MemoryBindingResolutionError`.
  - Returns `(memory_ns: dict[str, Any], audit_records: list[dict[str, Any]])`.

### Task 3 — Tests

`tests/orchestrator/test_template_memory_bindings.py` — 13 tests, all green:

| Test | Coverage |
|------|----------|
| `test_get_mode_requires_key` | model validator rejects get + no key |
| `test_scan_mode_does_not_require_key` | scan without key/key_prefix is valid |
| `test_get_mode_returns_value_from_connector` | value injected; audit found=True |
| `test_get_mode_uses_default_when_key_missing` | default returned; found=False |
| `test_scan_mode_returns_prefix_filtered_dict` | prefix filter + suffix keys + item_count |
| `test_scan_max_items_truncates_results` | ≤ max_items entries |
| `test_key_substitution_with_input_placeholder` | {input.user_id} expanded before read |
| `test_empty_bindings_returns_empty_namespace` | ({}, []) |
| `test_no_memory_resolver_returns_empty_namespace` | ({}, []) even with bindings |
| `test_unknown_connector_raises_memory_binding_resolution_error` | MemoryBindingResolutionError |
| `test_multiple_bindings_all_resolved` | two as_name keys in namespace |
| `test_memory_flows_into_rendered_instruction` | rendered instruction contains memory value (integration) |
| `test_tmb_audit_record_present_in_dispatch_audit` | audit_record has template_memory_bindings (integration) |

## Deviations from Plan

None. Implemented exactly as planned. All 13 tests pass; 1298 total tests pass (zero regressions).

## Verification Results

| Check | Result |
|-------|--------|
| `uv run pytest tests/orchestrator/test_template_memory_bindings.py -v` | **13 passed** |
| `uv run pytest -x -q` | **1298 passed, 0 failed** |
| `uv run ruff check src/zeroth/core/graph/models.py src/zeroth/core/graph/__init__.py src/zeroth/core/orchestrator/runtime.py` | **All checks passed** |

## Open Follow-ups

1. **THREAD-scoped binding integration test** — requires `RepositoryThreadResolver` wiring in tests; deferred.
2. **Scan-mode prefix substitution test** — `key_prefix` supports `{state.field}` placeholders but no dedicated test.
3. **`max_items` ordering guarantee** — current truncation is dict-insertion-order (backend-defined); a future `order_by` field would make ordering explicit.
