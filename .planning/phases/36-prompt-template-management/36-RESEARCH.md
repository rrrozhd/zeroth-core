# Phase 36: Prompt Template Management - Research

**Researched:** 2026-04-12
**Domain:** Versioned prompt template registry with Jinja2 sandboxed rendering, agent runtime integration, and audit redaction
**Confidence:** HIGH

## Summary

This phase adds a versioned prompt template registry to zeroth-core, allowing graph authors to define reusable prompt templates by name+version and reference them from agent nodes instead of hardcoding instruction strings. Templates are rendered at runtime using Jinja2's `SandboxedEnvironment` (with `StrictUndefined` for fail-loud behavior), and the rendered prompt appears in audit records with secret variables automatically redacted.

The codebase already contains every building block needed. The `ContractRegistry` pattern provides a proven versioned-registry architecture (name+version lookup, auto-increment, list operations). Jinja2 3.1.6 is already installed (transitively via litellm) and provides `SandboxedEnvironment` with `StrictUndefined`, `jinja2.meta.find_undeclared_variables()` for variable extraction, and `TemplateSyntaxError` for validation. The `PromptAssembler` in `agent_runtime/prompt.py` is the integration point where template resolution replaces the raw instruction string. The `audit/sanitizer.py` `PayloadSanitizer` with its `AuditRedactionConfig` provides the secret redaction infrastructure that will be extended with template-variable-aware patterns.

**Primary recommendation:** Create `zeroth.core.templates` package with 4 files (models.py, registry.py, errors.py, __init__.py), add a `TemplateRenderer` class using `SandboxedEnvironment(undefined=StrictUndefined)`, integrate template resolution into `PromptAssembler.assemble()`, extend `AuditRedactionConfig` with secret variable pattern matching, and wire the registry into `ServiceBootstrap`. Promote `jinja2>=3.1` to an explicit dependency in `pyproject.toml`.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `TemplateRegistry` is an in-memory registry analogous to `ContractRegistry` -- stores templates by name with version history. Methods: `register(name, version, template_str, metadata)`, `get(name, version?)`, `list()`, `get_latest(name)`.
- **D-02:** `PromptTemplate` Pydantic model with fields: `name` (str), `version` (int), `template_str` (str -- Jinja2 template source), `variables` (list[str] -- declared variable names), `metadata` (dict), `created_at` (datetime).
- **D-03:** Template registry placed in a new `zeroth.core.templates` package -- follows existing package pattern (models.py, registry.py, errors.py, __init__.py).
- **D-04:** Templates render using `jinja2.SandboxedEnvironment` -- prevents template injection attacks. No custom filters or extensions in v4.0.
- **D-05:** Variable sources at render time: node input payload, run state, and memory context -- resolved by the caller (agent runtime), not the template system itself.
- **D-06:** Undefined variables raise `TemplateRenderError` (not silently empty) -- consistent with fail-loud governance philosophy.
- **D-07:** `AgentNodeConfig` (or equivalent agent config model) gains an optional `template_ref: TemplateReference | None` field. When set, the agent runtime resolves the template, renders it with the current context, and uses the result as the instruction instead of the raw `instruction` field.
- **D-08:** `TemplateReference` is a lightweight model: `name` (str), `version` (int | None -- None means latest).
- **D-09:** The rendered prompt (post-interpolation) is included in audit records via the existing `execution_metadata` dict pattern -- key: `"rendered_prompt"`.
- **D-10:** Template variables whose names match secret patterns (configurable, default: contains "secret", "token", "key", "password", "api_key") are automatically redacted in the audit output. The `audit/sanitizer.py` patterns are extended.

### Claude's Discretion
- Error class hierarchy details (TemplateNotFoundError, TemplateVersionError, TemplateRenderError)
- Whether to add template validation at registration time (syntax check)
- Test fixture strategy

### Deferred Ideas (OUT OF SCOPE)
- None -- discussion stayed within phase scope.

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TMPL-01 | A template registry stores and versions prompt templates by name, analogous to the contract registry, and templates can be created, retrieved, and listed via the registry API | `TemplateRegistry` in-memory class with register/get/get_latest/list methods; `PromptTemplate` Pydantic model; version auto-increment pattern from `ContractRegistry.latest_version()` |
| TMPL-02 | Templates support variable interpolation from node input, run state, or memory using Jinja2 SandboxedEnvironment, preventing template injection attacks | `TemplateRenderer` class wrapping `SandboxedEnvironment(undefined=StrictUndefined)`; `jinja2.meta.find_undeclared_variables()` for variable extraction at registration; `SecurityError` blocks `__class__`, `__bases__` etc. |
| TMPL-03 | An agent node can reference a template by name and version instead of providing a raw instruction string; the template is resolved and rendered at runtime before the LLM invocation | `TemplateReference` model on `AgentNodeData`; resolution in `PromptAssembler.assemble()` or orchestrator agent execution path; rendered instruction replaces `config.instruction` |
| TMPL-04 | The rendered prompt (post-interpolation) is available in audit records; template variables containing secrets are automatically redacted in audit output | `rendered_prompt` key in `execution_metadata` dict; secret variable name pattern matching (contains "secret", "token", "key", "password", "api_key") with configurable patterns; extends `AuditRedactionConfig.redact_keys` |

</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Build/test commands: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- All source goes under `src/zeroth/core/`
- Tests go under `tests/`
- Progress logging is mandatory via `progress-logger` skill during implementation
- Backward compatibility: existing tests must continue passing

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| jinja2 | 3.1.6 | Sandboxed template rendering with variable interpolation | Already installed (transitively); provides `SandboxedEnvironment`, `StrictUndefined`, `meta.find_undeclared_variables()`, `TemplateSyntaxError` -- all needed for this phase [VERIFIED: `uv pip show jinja2` shows 3.1.6] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| markupsafe | (jinja2 dep) | String escaping for Jinja2 internals | Automatically used by jinja2; autoescape must be disabled for prompt rendering [VERIFIED: `uv pip show jinja2` shows markupsafe as requirement] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Jinja2 SandboxedEnvironment | Python `str.format_map()` | No sandbox, no injection prevention, no syntax validation, no variable extraction -- unacceptable for production governance platform |
| Jinja2 SandboxedEnvironment | Mako/Chameleon | Different template syntax, not already installed, overkill for prompt rendering use case |
| In-memory TemplateRegistry | Database-backed registry (like ContractRegistry) | In-memory is sufficient for v4.0 -- templates are registered at graph setup time, not persisted across restarts. Database backing can be added later if needed. |

**Installation:**
```bash
# Add jinja2 as explicit dependency in pyproject.toml (currently transitive via litellm)
# Then: uv sync
```

**Version verification:**
- jinja2 3.1.6 installed [VERIFIED: `uv pip show jinja2` output 3.1.6, brought in by litellm/mkdocs]
- Per STATE.md: "Jinja2 promoted from transitive to explicit dependency (templates)" -- this is a project-level decision [VERIFIED: STATE.md]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/templates/
    __init__.py          # Public API re-exports
    models.py            # PromptTemplate, TemplateReference, TemplateRenderResult
    registry.py          # TemplateRegistry (in-memory, versioned)
    renderer.py          # TemplateRenderer (SandboxedEnvironment wrapper)
    errors.py            # Error hierarchy
```

### Pattern 1: In-Memory Versioned Registry (from ContractRegistry)
**What:** Store templates by name with version history in a dict-of-lists. Auto-increment version on register. Look up by name+version or name (latest).
**When to use:** Template registration and retrieval.
**Example:**
```python
# Source: Adapted from ContractRegistry pattern [VERIFIED: contracts/registry.py]
from __future__ import annotations
from datetime import UTC, datetime
from typing import Any

from zeroth.core.templates.errors import (
    TemplateNotFoundError,
    TemplateVersionExistsError,
)
from zeroth.core.templates.models import PromptTemplate


class TemplateRegistry:
    """In-memory versioned prompt template registry."""

    def __init__(self) -> None:
        # name -> {version -> PromptTemplate}
        self._templates: dict[str, dict[int, PromptTemplate]] = {}

    def register(
        self,
        name: str,
        version: int,
        template_str: str,
        *,
        variables: list[str] | None = None,
        metadata: dict[str, Any] | None = None,
    ) -> PromptTemplate:
        """Register a new template version. Raises on duplicate."""
        versions = self._templates.setdefault(name, {})
        if version in versions:
            raise TemplateVersionExistsError(
                f"template {name!r} version {version} already exists"
            )
        # Auto-detect variables from template source if not provided
        detected_vars = variables or self._extract_variables(template_str)
        template = PromptTemplate(
            name=name,
            version=version,
            template_str=template_str,
            variables=detected_vars,
            metadata=metadata or {},
            created_at=datetime.now(UTC),
        )
        versions[version] = template
        return template

    def get(self, name: str, version: int | None = None) -> PromptTemplate:
        """Get a template by name and optional version. None = latest."""
        if version is None:
            return self.get_latest(name)
        versions = self._templates.get(name)
        if versions is None or version not in versions:
            raise TemplateNotFoundError(
                f"template {name!r} version {version} not found"
            )
        return versions[version]

    def get_latest(self, name: str) -> PromptTemplate:
        """Get the highest-versioned template for a name."""
        versions = self._templates.get(name)
        if not versions:
            raise TemplateNotFoundError(f"template {name!r} not found")
        latest_version = max(versions.keys())
        return versions[latest_version]

    def list(self) -> list[PromptTemplate]:
        """Return all templates across all names and versions."""
        result = []
        for versions in self._templates.values():
            result.extend(versions.values())
        return sorted(result, key=lambda t: (t.name, t.version))

    def _extract_variables(self, template_str: str) -> list[str]:
        """Extract undeclared variable names from template source."""
        from jinja2 import meta
        from jinja2.sandbox import SandboxedEnvironment
        env = SandboxedEnvironment()
        ast = env.parse(template_str)
        return sorted(meta.find_undeclared_variables(ast))
```

### Pattern 2: Sandboxed Template Renderer
**What:** Wrapper around `jinja2.SandboxedEnvironment(undefined=StrictUndefined)` that renders a template string with given variables and raises `TemplateRenderError` on undefined variables.
**When to use:** At agent runtime, just before LLM invocation.
**Example:**
```python
# Source: Jinja2 SandboxedEnvironment API [VERIFIED: runtime Python inspection]
from jinja2 import StrictUndefined, TemplateSyntaxError, UndefinedError
from jinja2.sandbox import SandboxedEnvironment, SecurityError

from zeroth.core.templates.errors import TemplateRenderError, TemplateSyntaxValidationError
from zeroth.core.templates.models import PromptTemplate


class TemplateRenderer:
    """Renders Jinja2 templates in a sandboxed environment."""

    def __init__(self) -> None:
        # autoescape=False is critical -- prompts are plain text, not HTML
        self._env = SandboxedEnvironment(undefined=StrictUndefined)

    def render(
        self,
        template: PromptTemplate,
        variables: dict[str, object],
    ) -> str:
        """Render a template with the given variables.

        Raises TemplateRenderError if a variable is undefined or if
        the template attempts unsafe operations.
        """
        try:
            jinja_template = self._env.from_string(template.template_str)
            return jinja_template.render(variables)
        except UndefinedError as exc:
            raise TemplateRenderError(
                f"undefined variable in template {template.name!r} v{template.version}: {exc}"
            ) from exc
        except SecurityError as exc:
            raise TemplateRenderError(
                f"security violation in template {template.name!r} v{template.version}: {exc}"
            ) from exc

    def validate_syntax(self, template_str: str) -> list[str]:
        """Validate template syntax and return list of undeclared variables.

        Raises TemplateSyntaxValidationError if the template has syntax errors.
        """
        try:
            from jinja2 import meta
            ast = self._env.parse(template_str)
            return sorted(meta.find_undeclared_variables(ast))
        except TemplateSyntaxError as exc:
            raise TemplateSyntaxValidationError(
                f"template syntax error: {exc}"
            ) from exc
```

### Pattern 3: Agent Runtime Template Resolution
**What:** In the agent execution path, check if the agent config has a `template_ref`. If so, resolve the template from the registry, render it with context variables, and use the result as the instruction.
**When to use:** Before `PromptAssembler.assemble()` is called in the orchestrator's agent execution path.
**Example:**
```python
# Source: Integration point identified in agent_runtime/prompt.py and orchestrator/runtime.py
# [VERIFIED: both files read and analyzed]

# Option A: Resolve in PromptAssembler.assemble() -- simpler
def assemble(self, config, input_payload, *, thread_state=None, runtime_context=None):
    # If config has template_ref and a template_registry + renderer are available,
    # resolve and render the template to replace config.instruction
    effective_instruction = config.instruction
    if (
        self.template_registry is not None
        and self.template_renderer is not None
        and getattr(config, 'template_ref', None) is not None
    ):
        template = self.template_registry.get(
            config.template_ref.name,
            config.template_ref.version,
        )
        render_context = {
            "input": normalized_input_dump,
            "state": thread_state_dump,
            "memory": runtime_context.get("memory", {}),
        }
        effective_instruction = self.template_renderer.render(template, render_context)
    # Use effective_instruction instead of config.instruction in system prompt
    ...

# Option B: Resolve in orchestrator before creating AgentRunner -- requires
# modifying config.instruction, which is frozen. Would need model_copy().
```

### Pattern 4: Secret Variable Redaction in Audit
**What:** Before writing the rendered prompt to audit records, scan the template's declared variable names for secret patterns and redact their values in the rendered output.
**When to use:** When attaching `rendered_prompt` to `execution_metadata`.
**Example:**
```python
# Source: Pattern from audit/sanitizer.py PayloadSanitizer [VERIFIED: file read]
import re

DEFAULT_SECRET_PATTERNS: tuple[str, ...] = (
    "secret", "token", "key", "password", "api_key",
)

def identify_secret_variables(
    variable_names: list[str],
    secret_patterns: tuple[str, ...] = DEFAULT_SECRET_PATTERNS,
) -> set[str]:
    """Identify which template variables are secrets based on name patterns."""
    secrets = set()
    for var in variable_names:
        var_lower = var.lower()
        for pattern in secret_patterns:
            if pattern in var_lower:
                secrets.add(var)
                break
    return secrets

def redact_rendered_prompt(
    rendered: str,
    variables: dict[str, object],
    secret_variable_names: set[str],
) -> str:
    """Replace secret variable values in the rendered prompt with redaction markers."""
    redacted = rendered
    for var_name in secret_variable_names:
        value = variables.get(var_name)
        if value is not None:
            redacted = redacted.replace(str(value), "***REDACTED***")
    return redacted
```

### Anti-Patterns to Avoid
- **Don't use `Environment` instead of `SandboxedEnvironment`:** Regular Jinja2 `Environment` allows arbitrary Python attribute access (e.g., `{{ [].__class__.__bases__[0].__subclasses__() }}`), enabling template injection attacks. `SandboxedEnvironment` blocks access to unsafe attributes. [VERIFIED: runtime test confirmed `SecurityError` on `__class__` access]
- **Don't leave `undefined` as default:** Jinja2's default `Undefined` silently renders missing variables as empty strings. For a governance platform, undefined variables must raise `UndefinedError` via `StrictUndefined`. [VERIFIED: runtime test confirmed silent empty vs UndefinedError behavior]
- **Don't enable `autoescape`:** Auto-escaping converts `<` to `&lt;` etc., which corrupts prompts. Prompts are plain text, not HTML. Default `autoescape=False` is correct. [VERIFIED: runtime test confirmed escaping behavior]
- **Don't make `TemplateRenderer` stateful per-template:** The `SandboxedEnvironment` instance should be created once and reused across all template renders. `env.from_string()` compiles each template on the fly. No need to cache compiled templates in v4.0.
- **Don't add custom Jinja2 filters or extensions:** D-04 explicitly excludes these for v4.0. Built-in Jinja2 filters (`join`, `length`, `upper`, `lower`, `default`, etc.) are sufficient.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Template rendering engine | Custom string interpolation | `jinja2.SandboxedEnvironment.from_string().render()` | Jinja2 handles parsing, compilation, variable resolution, error handling, and sandbox enforcement in ~3 lines of code [VERIFIED: runtime API inspection] |
| Template syntax validation | Custom regex-based parser | `SandboxedEnvironment.parse()` + catch `TemplateSyntaxError` | Jinja2's parser catches all syntax errors (unclosed tags, invalid expressions, etc.) that a regex approach would miss [VERIFIED: runtime test confirmed TemplateSyntaxError on malformed templates] |
| Variable extraction from templates | Manual regex for `{{ var }}` patterns | `jinja2.meta.find_undeclared_variables(env.parse(source))` | Jinja2's AST-based extraction handles nested variables, filters, and complex expressions that regex cannot [VERIFIED: runtime test extracted `{'memory', 'role', 'name'}` from complex template] |
| Template injection prevention | Custom string escaping / blocklists | `SandboxedEnvironment` | Blocks `__class__`, `__bases__`, `__subclasses__`, and other dangerous Python dunder attributes at the AST level, not string level [VERIFIED: runtime test confirmed SecurityError on `[].__class__`] |

**Key insight:** Jinja2's `SandboxedEnvironment` with `StrictUndefined` provides all three critical requirements (safe rendering, fail-loud on undefined, syntax validation) in a single well-tested library that is already installed. The custom code in this phase is entirely about the registry, integration wiring, and audit redaction -- not about template rendering itself.

## Common Pitfalls

### Pitfall 1: Silent Empty Variables with Default Undefined
**What goes wrong:** Template variables that are not provided render as empty strings, silently producing incorrect prompts.
**Why it happens:** Jinja2's default `Undefined` class renders to empty string. Without `StrictUndefined`, typos in variable names go undetected.
**How to avoid:** Always use `SandboxedEnvironment(undefined=StrictUndefined)`. D-06 mandates this. [VERIFIED: runtime test confirmed behavior difference]
**Warning signs:** Prompts with missing context sections, LLM producing unexpected output because instruction was partially empty.

### Pitfall 2: Template Instruction Overwrite Breaks Existing Agents
**What goes wrong:** Adding `template_ref` to `AgentNodeData` or `AgentConfig` with `extra="forbid"` breaks existing graph definitions that don't include the field.
**Why it happens:** Pydantic `extra="forbid"` rejects unknown fields, but adding a NEW optional field is safe. The risk is in deserialization of existing serialized graphs that were saved before the field existed.
**How to avoid:** `template_ref: TemplateReference | None = None` with default `None` is backward compatible. Existing graphs without the field will deserialize correctly because Pydantic fills the default. The `instruction` field remains required -- when `template_ref` is set, the rendered result replaces the instruction at runtime, but the raw `instruction` field can serve as a fallback description. [VERIFIED: Pydantic ConfigDict(extra="forbid") allows new optional fields with defaults]
**Warning signs:** `ValidationError` during graph deserialization in tests.

### Pitfall 3: Circular Dependency Between Templates and Agent Runtime
**What goes wrong:** If `TemplateRenderer` is instantiated inside `PromptAssembler`, and `PromptAssembler` is imported by the templates package, you get a circular import.
**Why it happens:** Tight coupling between the template system and the agent runtime.
**How to avoid:** Keep `zeroth.core.templates` a standalone package with zero imports from `agent_runtime`. The `PromptAssembler` imports from `templates`, not the other way around. The `TemplateRegistry` and `TemplateRenderer` are injected into `PromptAssembler` at construction time (dependency injection), not imported at module level. [VERIFIED: existing pattern -- `PromptAssembler` has no external imports beyond its own package and pydantic]
**Warning signs:** `ImportError` during module loading.

### Pitfall 4: Secret Redaction by String Replacement Has Edge Cases
**What goes wrong:** Simple `str.replace(secret_value, "***REDACTED***")` can miss partial matches or produce incorrect redaction when secret values are substrings of other content.
**Why it happens:** String replacement is greedy and position-unaware.
**How to avoid:** For v4.0, string replacement is pragmatically sufficient -- secret values in prompts are typically distinct strings (API keys, tokens) that won't be substrings of normal text. Document the limitation. A more robust approach (tracking render-time positions) is out of scope for v4.0.
**Warning signs:** Partial redaction or over-redaction in audit records.

### Pitfall 5: PromptAssembler Constructor Signature Change Breaks Tests
**What goes wrong:** Adding `template_registry` and `template_renderer` parameters to `PromptAssembler.__init__()` breaks all existing tests that instantiate it.
**Why it happens:** Existing tests create `PromptAssembler()` with no arguments.
**How to avoid:** Make both parameters optional with `None` default: `template_registry: TemplateRegistry | None = None`, `template_renderer: TemplateRenderer | None = None`. When both are `None`, template resolution is skipped entirely, preserving existing behavior. [VERIFIED: `PromptAssembler.__init__` currently takes no parameters -- agent_runtime/prompt.py line 48]
**Warning signs:** Mass test failures in `tests/agent_runtime/`.

### Pitfall 6: Registry Not Wired at Bootstrap Time
**What goes wrong:** Templates are registered programmatically but the registry is never passed to the orchestrator/agent runtime, so template resolution never fires.
**Why it happens:** Missing bootstrap wiring -- the registry must be created in `bootstrap_service()` and passed through to the `PromptAssembler` instances.
**How to avoid:** Follow the established pattern: add `template_registry: object | None = None` to `ServiceBootstrap` dataclass, create the registry in `bootstrap_service()`, and wire it to the orchestrator. The orchestrator passes it to `PromptAssembler` when creating agent runners. [VERIFIED: bootstrap.py pattern from Phase 34/35 -- artifact_store and http_client both follow this pattern]
**Warning signs:** Template references on agent nodes silently fall back to raw instruction.

## Code Examples

### PromptTemplate Model
```python
# Source: Pattern from ContractVersion model [VERIFIED: contracts/registry.py]
from __future__ import annotations
from datetime import datetime
from typing import Any

from pydantic import BaseModel, ConfigDict, Field


class PromptTemplate(BaseModel):
    """A single versioned prompt template record."""
    model_config = ConfigDict(extra="forbid", frozen=True)

    name: str
    version: int
    template_str: str
    variables: list[str] = Field(default_factory=list)
    metadata: dict[str, Any] = Field(default_factory=dict)
    created_at: datetime
```

### TemplateReference Model
```python
# Source: Pattern from ContractReference model [VERIFIED: contracts/registry.py]
from pydantic import BaseModel, ConfigDict


class TemplateReference(BaseModel):
    """Lightweight pointer to a template by name and optional version."""
    model_config = ConfigDict(extra="forbid", frozen=True)

    name: str
    version: int | None = None  # None = latest
```

### Error Hierarchy
```python
# Source: Pattern from contracts/errors.py [VERIFIED: file read]

class TemplateError(Exception):
    """Base error for the templates subsystem."""

class TemplateNotFoundError(TemplateError):
    """Raised when a template name or version cannot be found."""

class TemplateVersionExistsError(TemplateError):
    """Raised when registering a duplicate name+version."""

class TemplateRenderError(TemplateError):
    """Raised when template rendering fails (undefined var, security violation)."""

class TemplateSyntaxValidationError(TemplateError):
    """Raised when template source has invalid Jinja2 syntax."""
```

### AgentNodeData Extension
```python
# Source: Existing AgentNodeData in graph/models.py [VERIFIED: file read, line 109]
# Add optional template_ref field:

class AgentNodeData(BaseModel):
    instruction: str
    model_provider: str
    # ... existing fields ...
    template_ref: TemplateReference | None = None  # NEW - Phase 36
```

Note: The `TemplateReference` import must be from `zeroth.core.templates.models`. Since `graph/models.py` currently has no imports from the templates package, this is a clean addition with no circular dependency risk. [VERIFIED: graph/models.py imports only from `governai`, `pydantic`, and `zeroth.core.mappings.models`]

### Bootstrap Wiring
```python
# Source: Pattern from bootstrap.py Phase 34/35 wiring [VERIFIED: bootstrap.py]
# In ServiceBootstrap dataclass:
template_registry: object | None = None  # TemplateRegistry instance

# In bootstrap_service():
from zeroth.core.templates import TemplateRegistry
template_registry = TemplateRegistry()
orchestrator.template_registry = template_registry

# In ServiceBootstrap return:
template_registry=template_registry,
```

### Audit Metadata Extension
```python
# Source: NodeAuditRecord.execution_metadata pattern [VERIFIED: audit/models.py line 126]
# After rendering a template in the agent execution path:
audit_record["execution_metadata"]["rendered_prompt"] = redacted_rendered_prompt
audit_record["execution_metadata"]["template_ref"] = {
    "name": template.name,
    "version": template.version,
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Hardcoded instruction strings in agent config | Versioned prompt templates with registry | Phase 36 (new) | Enables prompt reuse across agents, version management, audit trail of which prompt version was used |
| `jinja2.Environment` for templates | `jinja2.SandboxedEnvironment` | Long-standing best practice | SandboxedEnvironment blocks template injection; no reason to ever use base Environment for user-authored templates [VERIFIED: Jinja2 docs] |
| Jinja2 default `Undefined` | `StrictUndefined` | Best practice for production systems | Prevents silent empty-string rendering of undefined variables [VERIFIED: runtime test] |

**Deprecated/outdated:**
- `jinja2.Environment` (without sandbox): Never use for user-authored templates -- allows arbitrary Python access via `__class__.__bases__` chains [VERIFIED: runtime test confirmed vulnerability]

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | In-memory `TemplateRegistry` (dict-of-dicts) is sufficient for v4.0 -- no database persistence needed | Architecture Patterns | LOW -- templates are registered at graph setup time; if persistence is needed later, the registry interface is stable and can be backed by DB |
| A2 | String replacement for secret redaction in rendered prompts is sufficient for v4.0 | Pitfall 4 / Architecture Pattern 4 | LOW -- secret values are typically distinct strings (API keys, tokens); edge cases with substring matches are unlikely in practice |
| A3 | `PromptAssembler` is the correct integration point (vs. orchestrator's `_execute_node`) | Architecture Pattern 3 | LOW -- PromptAssembler already assembles the instruction; adding template resolution there keeps the template concern in one place. If wrong, refactoring to orchestrator is straightforward |
| A4 | The `instruction` field on `AgentNodeData` should remain required (not made optional) even when `template_ref` is set | Pitfall 2 | LOW -- keeping instruction required means existing validation passes; the raw instruction serves as documentation/fallback when template_ref is set |

## Open Questions

1. **Where exactly to resolve templates: PromptAssembler vs Orchestrator?**
   - What we know: `PromptAssembler.assemble()` receives `AgentConfig` (which has `instruction`) and builds the prompt. The orchestrator's `_execute_node()` creates the `AgentRunner` and calls `runner.run()` which calls `prompt_assembler.assemble()`.
   - What's unclear: `AgentConfig` (runtime model) and `AgentNodeData` (graph model) are separate. The `template_ref` field goes on `AgentNodeData`. The orchestrator would need to resolve the template and either (a) pass the rendered instruction to `AgentConfig.instruction` when constructing the runner, or (b) pass the `TemplateReference` through to `PromptAssembler`.
   - Recommendation: **Option (a)** -- resolve in the orchestrator before constructing the agent runner. The orchestrator already has access to the run state and input payload (the template variable sources per D-05). Render the template there and pass the rendered string as `instruction` when building `AgentConfig`. This avoids modifying `AgentConfig` at all and keeps template resolution in the orchestrator's wiring layer where it belongs. The `PromptAssembler` does not need to know about templates.

2. **Should `TemplateRegistry` validate syntax at registration time?**
   - What we know: `SandboxedEnvironment.parse()` catches syntax errors. This is cheap and prevents registering broken templates.
   - What's unclear: Whether validation should be mandatory or optional. D-04 says "no custom filters or extensions" but doesn't explicitly say "validate at registration."
   - Recommendation: **Yes, validate at registration time.** Call `env.parse(template_str)` during `register()` and raise `TemplateSyntaxValidationError` on failure. This is a ~0ms operation and prevents runtime failures from bad templates. This is Claude's discretion per CONTEXT.md.

3. **How should `AgentConfig` and `AgentNodeData` relate for template_ref?**
   - What we know: `AgentNodeData` is the graph-level config (persisted in graph JSON). `AgentConfig` is the runtime config (constructed by the orchestrator when creating `AgentRunner`). Currently, the orchestrator manually maps `AgentNodeData` fields to `AgentConfig` fields.
   - What's unclear: Should `AgentConfig` also get `template_ref`, or should the orchestrator resolve the template before constructing `AgentConfig`?
   - Recommendation: Add `template_ref` only to `AgentNodeData` (graph model). The orchestrator resolves the template during agent execution setup and passes the rendered instruction as the `instruction` field of `AgentConfig`. This means `AgentConfig` does not change, which avoids breaking existing `AgentRunner` tests.

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | Template system does not handle auth |
| V3 Session Management | no | Stateless template rendering |
| V4 Access Control | no | Template access controlled by graph authoring permissions (existing) |
| V5 Input Validation | yes | Template syntax validated at registration; variable types validated at render; `SandboxedEnvironment` blocks unsafe attribute access |
| V6 Cryptography | no | No cryptographic operations |

### Known Threat Patterns for Template Rendering

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Template injection (SSTI) | Elevation of Privilege | `SandboxedEnvironment` blocks `__class__`, `__bases__`, `__subclasses__` and other dunder access [VERIFIED: runtime test] |
| Secret leakage in rendered prompts | Information Disclosure | Secret variable pattern matching + value redaction in audit records (D-10) |
| Denial of service via template complexity | Denial of Service | Jinja2's sandbox limits recursion; no custom extensions allowed (D-04); templates are author-registered, not user-input |
| Undefined variable information disclosure | Information Disclosure | `StrictUndefined` raises error rather than silently proceeding with partial data |

## Sources

### Primary (HIGH confidence)
- Jinja2 3.1.6 runtime -- inspected `SandboxedEnvironment`, `StrictUndefined`, `meta.find_undeclared_variables()`, `TemplateSyntaxError`, `SecurityError`, `UndefinedError` APIs via Python runtime [VERIFIED: runtime inspection and execution tests]
- `src/zeroth/core/contracts/registry.py` -- `ContractRegistry` versioned registry pattern, `ContractReference` model [VERIFIED: file read]
- `src/zeroth/core/contracts/errors.py` -- Error hierarchy pattern [VERIFIED: file read]
- `src/zeroth/core/agent_runtime/prompt.py` -- `PromptAssembler` integration point [VERIFIED: file read]
- `src/zeroth/core/agent_runtime/models.py` -- `AgentConfig`, `PromptConfig`, `PromptAssembly` models [VERIFIED: file read]
- `src/zeroth/core/agent_runtime/runner.py` -- `AgentRunner` execution flow [VERIFIED: file read]
- `src/zeroth/core/orchestrator/runtime.py` -- `RuntimeOrchestrator._execute_node()` and agent execution path [VERIFIED: file read]
- `src/zeroth/core/graph/models.py` -- `AgentNodeData`, `AgentNode`, `NodeBase` models [VERIFIED: file read]
- `src/zeroth/core/audit/sanitizer.py` -- `PayloadSanitizer` and `AuditRedactionConfig` [VERIFIED: file read]
- `src/zeroth/core/audit/models.py` -- `NodeAuditRecord.execution_metadata` dict pattern [VERIFIED: file read]
- `src/zeroth/core/config/settings.py` -- `ZerothSettings` sub-model pattern [VERIFIED: file read]
- `src/zeroth/core/service/bootstrap.py` -- `ServiceBootstrap` wiring pattern, Phase 34/35 additions [VERIFIED: file read]
- `pyproject.toml` -- Direct dependencies list (jinja2 NOT present, needs promotion) [VERIFIED: file read]
- `.planning/STATE.md` -- "Jinja2 promoted from transitive to explicit dependency" decision [VERIFIED: file read]

### Secondary (MEDIUM confidence)
- [Jinja2 Sandbox documentation](https://jinja.palletsprojects.com/en/3.1.x/sandbox/) -- SandboxedEnvironment security model [CITED: docs.palletsprojects.com]

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- jinja2 3.1.6 already installed and API verified via runtime tests; no new dependencies beyond promoting jinja2 to explicit
- Architecture: HIGH -- all integration points verified in codebase; registry pattern proven by ContractRegistry; PromptAssembler and orchestrator paths fully traced
- Pitfalls: HIGH -- all pitfalls verified via runtime tests (SandboxedEnvironment behavior, StrictUndefined, autoescape) and codebase analysis (backward compatibility, import structure)

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- Jinja2 3.1.x and codebase patterns unlikely to change rapidly)
