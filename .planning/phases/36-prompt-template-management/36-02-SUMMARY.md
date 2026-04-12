---
phase: 36-prompt-template-management
plan: 02
subsystem: templates/orchestrator/bootstrap
tags: [templates, orchestrator, redaction, audit, integration]
dependency_graph:
  requires: [36-01]
  provides: [template-resolution, audit-redaction, bootstrap-wiring]
  affects: [orchestrator, graph-models, service-bootstrap, audit]
tech_stack:
  added: []
  patterns: [save-restore-config, flattened-variable-redaction, tdd]
key_files:
  created:
    - src/zeroth/core/templates/redaction.py
    - tests/templates/test_redaction.py
    - tests/templates/test_integration.py
  modified:
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/orchestrator/runtime.py
    - src/zeroth/core/templates/__init__.py
    - src/zeroth/core/service/bootstrap.py
decisions:
  - "Use save/restore pattern for runner config (identical to existing cost instrumentation pattern)"
  - "Flatten nested template variables for secret name matching rather than deep traversal"
  - "Redaction uses simple string replacement -- pragmatically sufficient for secret values"
metrics:
  duration: ~10 minutes
  completed: "2026-04-12T23:53:00Z"
  tasks: 2/2
  tests_added: 27
  tests_total: 59 (templates) / 84 (relevant suites)
  files_changed: 7
  commits: 4
---

# Phase 36 Plan 02: Orchestrator Template Resolution & Audit Redaction Summary

Template resolution wired into agent runtime with secret redaction for audit records, TDD green across 59 template tests.

## What Was Done

### Task 1: Agent node template_ref field and orchestrator template resolution

- Added `template_ref: TemplateReference | None = None` to `AgentNodeData` (backward compatible)
- Added `template_registry` and `template_renderer` fields to `RuntimeOrchestrator` dataclass
- Implemented template resolution in `_dispatch_node`: resolves template from registry, renders with `SandboxedEnvironment`, overrides runner config instruction, restores original in `finally` block
- Template variables include `input` (node input payload), `state` (run metadata), `memory` (empty dict placeholder)
- Recorded `rendered_prompt` and `template_ref` in audit `execution_metadata`
- Wired `TemplateRegistry` and `TemplateRenderer` creation at bootstrap time in `ServiceBootstrap`
- Backward compatible: nodes without `template_ref` use raw instruction unchanged

### Task 2: Audit secret variable redaction for rendered prompts

- Created `redaction.py` module with `identify_secret_variables()` and `redact_rendered_prompt()`
- Default secret patterns: `secret`, `token`, `key`, `password`, `api_key`
- Case-insensitive pattern matching on flattened template variable names
- Deterministic redaction output (sorted variable iteration)
- Wired redaction into orchestrator: secret values replaced with `***REDACTED***` before audit storage
- Exported redaction functions from `templates/__init__.py`

## Commits

| Hash | Type | Description |
|------|------|-------------|
| 29c1d26 | test | Add failing integration tests for template resolution |
| 01753db | feat | Implement template resolution in orchestrator and bootstrap wiring |
| 851ede4 | test | Add failing tests for secret variable redaction |
| 2d264ce | feat | Implement audit secret variable redaction for rendered prompts |

## Test Results

- `tests/templates/` -- 59 passed (models, registry, renderer, redaction, integration)
- `tests/orchestrator/` -- 16 passed
- `tests/graph/` -- 9 passed
- Total relevant: 84 passed, 0 failed
- Lint: all checks passed (ruff)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed test assertion for enum status value**
- **Found during:** Task 1 GREEN phase
- **Issue:** `RunStatus.FAILED.value` returns `"FAILED"` (uppercase), test expected `"failed"` (lowercase)
- **Fix:** Used `.value.lower()` comparison
- **Files modified:** `tests/templates/test_integration.py`

**2. [Rule 1 - Bug] Fixed audit record navigation for nested execution_metadata**
- **Found during:** Task 1 GREEN phase
- **Issue:** `NodeAuditRecord.execution_metadata` contains the full audit dict; template metadata is nested under the `execution_metadata` key within that dict
- **Fix:** Added `inner_meta = exec_meta.get("execution_metadata", exec_meta)` to navigate correctly
- **Files modified:** `tests/templates/test_integration.py`

**3. [Rule 1 - Bug] Fixed Yoda condition lint error**
- **Found during:** Task 2 REFACTOR phase
- **Issue:** `assert "Hello Bob!" == rendered` flagged as SIM300 Yoda condition
- **Fix:** Reversed to `assert rendered == "Hello Bob!"`
- **Files modified:** `tests/templates/test_integration.py`

## Threat Mitigation Verification

| Threat ID | Status | Evidence |
|-----------|--------|----------|
| T-36-05 | Mitigated | `identify_secret_variables` scans flattened variable names; `redact_rendered_prompt` replaces values with `***REDACTED***` before audit write |
| T-36-07 | Mitigated | `TemplateNotFoundError` propagates from `registry.get()` through `_dispatch_node`, caught by `_drive` outer try/except, fails the run loudly |
| T-36-08 | Mitigated | Original runner config saved before override and restored in `finally` block, identical to existing cost instrumentation pattern |

## Self-Check: PASSED

All 7 files verified present on disk. All 4 commits verified in git log.
