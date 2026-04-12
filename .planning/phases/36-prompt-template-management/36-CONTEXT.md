# Phase 36: Prompt Template Management - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a versioned prompt template registry that stores, versions, and renders prompt templates using Jinja2 SandboxedEnvironment. Agent nodes can reference a template by name+version instead of providing a raw instruction string. Rendered prompts appear in audit records with secret variables redacted.

</domain>

<decisions>
## Implementation Decisions

### Template Registry
- **D-01:** `TemplateRegistry` is an in-memory registry analogous to `ContractRegistry` — stores templates by name with version history. Methods: `register(name, version, template_str, metadata)`, `get(name, version?)`, `list()`, `get_latest(name)`.
- **D-02:** `PromptTemplate` Pydantic model with fields: `name` (str), `version` (int), `template_str` (str — Jinja2 template source), `variables` (list[str] — declared variable names), `metadata` (dict), `created_at` (datetime).
- **D-03:** Template registry placed in a new `zeroth.core.templates` package — follows existing package pattern (models.py, registry.py, errors.py, __init__.py).

### Jinja2 Rendering
- **D-04:** Templates render using `jinja2.SandboxedEnvironment` — prevents template injection attacks. No custom filters or extensions in v4.0.
- **D-05:** Variable sources at render time: node input payload, run state, and memory context — resolved by the caller (agent runtime), not the template system itself.
- **D-06:** Undefined variables raise `TemplateRenderError` (not silently empty) — consistent with fail-loud governance philosophy.

### Agent Node Integration
- **D-07:** `AgentNodeConfig` (or equivalent agent config model) gains an optional `template_ref: TemplateReference | None` field. When set, the agent runtime resolves the template, renders it with the current context, and uses the result as the instruction instead of the raw `instruction` field.
- **D-08:** `TemplateReference` is a lightweight model: `name` (str), `version` (int | None — None means latest).

### Audit & Redaction
- **D-09:** The rendered prompt (post-interpolation) is included in audit records via the existing `execution_metadata` dict pattern — key: `"rendered_prompt"`.
- **D-10:** Template variables whose names match secret patterns (configurable, default: contains "secret", "token", "key", "password", "api_key") are automatically redacted in the audit output. The `audit/sanitizer.py` patterns are extended.

### Claude's Discretion
- Error class hierarchy details (TemplateNotFoundError, TemplateVersionError, TemplateRenderError)
- Whether to add template validation at registration time (syntax check)
- Test fixture strategy

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Registry Pattern
- `src/zeroth/core/contracts/registry.py` — `ContractRegistry` with version management — reference for TemplateRegistry design

### Agent Runtime
- `src/zeroth/core/agent_runtime/prompt.py` — `PromptBuilder` — where template resolution integrates
- `src/zeroth/core/agent_runtime/models.py` — Agent config models (where template_ref field goes)
- `src/zeroth/core/agent_runtime/runner.py` — Agent execution flow

### Audit & Redaction
- `src/zeroth/core/audit/sanitizer.py` — Audit sanitization (secret redaction patterns)
- `src/zeroth/core/audit/models.py` — `NodeAuditRecord.execution_metadata`

### Settings
- `src/zeroth/core/config/settings.py` — ZerothSettings (if template settings needed)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ContractRegistry` — Direct pattern reference for versioned registry with name+version lookup
- `PromptBuilder` — Current prompt assembly, where template rendering will integrate
- `jinja2` 3.1.6 already installed — `SandboxedEnvironment` available
- `audit/sanitizer.py` — Existing secret redaction infrastructure

### Established Patterns
- Versioned registries (ContractRegistry pattern)
- Pydantic ConfigDict(extra="forbid") on all models
- Package structure: models.py, errors.py, __init__.py
- execution_metadata dict for audit record extensions

### Integration Points
- `agent_runtime/prompt.py` — Template resolution in PromptBuilder
- `agent_runtime/models.py` — TemplateReference field on agent config
- `audit/sanitizer.py` — Secret variable redaction patterns
- `service/bootstrap.py` — Initialize template registry at startup

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 36-prompt-template-management*
*Context gathered: 2026-04-13*
