---
phase: 36-prompt-template-management
plan: 01
subsystem: templates
tags: [templates, jinja2, registry, renderer, tdd, pydantic]
dependency_graph:
  requires: []
  provides: [zeroth.core.templates, PromptTemplate, TemplateRegistry, TemplateRenderer]
  affects: [pyproject.toml]
tech_stack:
  added: [jinja2>=3.1 (explicit dependency)]
  patterns: [SandboxedEnvironment, StrictUndefined, frozen Pydantic models, TDD red-green-refactor]
key_files:
  created:
    - src/zeroth/core/templates/__init__.py
    - src/zeroth/core/templates/models.py
    - src/zeroth/core/templates/errors.py
    - src/zeroth/core/templates/registry.py
    - src/zeroth/core/templates/renderer.py
    - tests/templates/__init__.py
    - tests/templates/test_models.py
    - tests/templates/test_registry.py
    - tests/templates/test_renderer.py
  modified:
    - pyproject.toml
decisions:
  - "Error hierarchy follows contracts pattern: TemplateError base with 4 specific subclasses"
  - "Registry uses in-memory dict[str, dict[int, PromptTemplate]] storage (Plan 02 may add persistence)"
  - "Renderer keeps autoescape off (default) since prompts are plain text for LLM consumption"
  - "validate_syntax returns sorted variable names for deterministic output"
metrics:
  duration: 6m 7s
  completed: 2026-04-12
  tasks: 2/2
  tests_added: 33
  tests_total: 724
  files_changed: 10
---

# Phase 36 Plan 01: Template Models, Registry & Renderer Summary

Versioned prompt template subsystem with Jinja2 SandboxedEnvironment, StrictUndefined, frozen Pydantic models, and in-memory registry -- all built via TDD with 33 new tests.

## What Was Built

### Models (`models.py`)
- **PromptTemplate**: Frozen, extra-forbid Pydantic model with name, version, template_str, auto-extracted variables, metadata, and created_at
- **TemplateReference**: Lightweight pointer by name + optional version (None = latest)
- **TemplateRenderResult**: Render output with provenance (template name, version, variables used)

### Error Hierarchy (`errors.py`)
- **TemplateError**: Base class for all template errors
- **TemplateNotFoundError**: Missing template or version
- **TemplateVersionExistsError**: Duplicate name+version registration
- **TemplateRenderError**: Undefined variable or security violation during render
- **TemplateSyntaxValidationError**: Invalid Jinja2 syntax at registration time

### Registry (`registry.py`)
- `register()`: Validates syntax, auto-extracts variables via `jinja2.meta`, checks duplicates, stores immutable PromptTemplate
- `get(name, version)`: Exact version lookup; delegates to `get_latest` when version is None
- `get_latest(name)`: Returns highest version number
- `list()`: Returns all templates sorted by (name, version)

### Renderer (`renderer.py`)
- `render()`: Renders via SandboxedEnvironment, catches UndefinedError and SecurityError, returns TemplateRenderResult
- `validate_syntax()`: Parses template, extracts and returns sorted variable names
- Autoescape off (prompts are plain text, not HTML)
- StrictUndefined ensures missing variables raise errors (not silent empty strings)

### Dependency Promotion
- `jinja2>=3.1` added as explicit dependency in pyproject.toml (was previously only a transitive dependency)

## Task Commits

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 (RED) | Failing model/registry tests | f726755 | tests/templates/test_models.py, tests/templates/test_registry.py |
| 1 (GREEN) | Models, errors, registry impl | a1b483a | models.py, errors.py, registry.py, __init__.py |
| 2 (RED) | Failing renderer tests | c041c3e | tests/templates/test_renderer.py |
| 2 (GREEN) | Renderer impl + jinja2 dep | 65c9f38 | renderer.py, __init__.py, pyproject.toml |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed test regex for Pydantic extra validation error**
- **Found during:** Task 1 GREEN phase
- **Issue:** Test used `extra_fields_not_permitted` but Pydantic 2.12 error type is `extra_forbidden`
- **Fix:** Updated regex match to `extra_forbidden`
- **Files modified:** tests/templates/test_models.py
- **Commit:** a1b483a (included in GREEN commit)

## Threat Mitigations Verified

| Threat ID | Mitigation | Verified |
|-----------|-----------|----------|
| T-36-01 | SandboxedEnvironment blocks dunder access | test_render_injection_attack_raises passes |
| T-36-02 | StrictUndefined raises on missing vars | test_render_undefined_variable_raises passes |
| T-36-04 | PromptTemplate frozen=True, duplicate raises error | test_frozen_immutable + test_register_duplicate_raises pass |

## Known Stubs

None -- all code is fully wired and functional.

## Self-Check: PASSED

All 9 created files verified on disk. All 4 commit hashes verified in git log.
