---
phase: 44-declarative-template-memory-sourcing
type: verification
status: passed
verified_at: 2026-04-15
---

# Phase 44 Verification

## Goal

Allow agent nodes to declare memory connector values that are automatically fetched and injected into the `{{ memory.* }}` Jinja2 template namespace at dispatch time.

## Verification Checklist

### Model

- [x] `TemplateMemoryBinding` class exists in `src/zeroth/core/graph/models.py`
- [x] `TemplateMemoryBinding` exported from `zeroth.core.graph` (`__init__.py`)
- [x] `AgentNodeData.template_memory_bindings: list[TemplateMemoryBinding]` field present, defaults to `[]`
- [x] `@model_validator` rejects `access_mode="get"` with `key=None`
- [x] `access_mode="scan"` with no key or key_prefix is valid

### Orchestrator

- [x] `MemoryBindingResolutionError(OrchestratorError)` defined in `runtime.py`
- [x] `import re` at module level
- [x] `_KEY_PLACEHOLDER_RE` defined at module level
- [x] `_substitute_tmb_key` defined at module level
- [x] `thread_id = await self._resolve_thread(node, run)` appears **before** the template `if` block in `_dispatch_node`
- [x] `tmb_audit_records` initialised outside the template block
- [x] `"memory": {}` replaced with `"memory": _memory_ns` inside template block
- [x] `_resolve_template_memory` method present on `RuntimeOrchestrator`
- [x] `audit_record["execution_metadata"]["template_memory_bindings"]` populated when bindings were resolved

### Resolution logic

- [x] `get` mode: `connector.read(key, scope)` called; `entry.value` returned or `binding.default`
- [x] `scan` mode: `connector.search({}, scope)` + client-side prefix filter + suffix stripping + `max_items` truncation
- [x] `{input.field}`, `{state.field}`, `{run.run_id}` substituted in key/key_prefix before read
- [x] Unknown connector → `MemoryBindingResolutionError` with "unknown memory connector" in message
- [x] No bindings → `({}, [])` returned immediately
- [x] `memory_resolver is None` → `({}, [])` returned immediately

### Tests

- [x] 13 tests in `tests/orchestrator/test_template_memory_bindings.py` — all pass
- [x] Full regression: `uv run pytest -x -q` → **1298 passed, 0 failed**
- [x] Lint: `uv run ruff check src/...` → **All checks passed**

## Verification Status: PASSED
