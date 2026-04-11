---
phase: 32-reference-docs-deployment-migration-guide
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - pyproject.toml
  - mkdocs.yml
  - docs/reference/python-api.md
  - docs/reference/python-api/graph.md
  - docs/reference/python-api/orchestrator.md
  - docs/reference/python-api/agents.md
  - docs/reference/python-api/execution-units.md
  - docs/reference/python-api/conditions.md
  - docs/reference/python-api/mappings.md
  - docs/reference/python-api/memory.md
  - docs/reference/python-api/storage.md
  - docs/reference/python-api/contracts.md
  - docs/reference/python-api/runs.md
  - docs/reference/python-api/policy.md
  - docs/reference/python-api/approvals.md
  - docs/reference/python-api/audit.md
  - docs/reference/python-api/guardrails.md
  - docs/reference/python-api/identity.md
  - docs/reference/python-api/secrets.md
  - docs/reference/python-api/dispatch.md
  - docs/reference/python-api/econ.md
  - docs/reference/python-api/service.md
  - docs/reference/python-api/webhooks.md
  - docs/how-to/graph.md
  - docs/how-to/orchestrator.md
  - docs/how-to/agents.md
  - docs/how-to/execution-units.md
  - docs/how-to/conditions.md
  - docs/how-to/mappings.md
  - docs/how-to/memory.md
  - docs/how-to/storage.md
  - docs/how-to/contracts.md
  - docs/how-to/runs.md
  - docs/how-to/policy.md
  - docs/how-to/approvals.md
  - docs/how-to/audit.md
  - docs/how-to/guardrails.md
  - docs/how-to/identity.md
  - docs/how-to/secrets.md
  - docs/how-to/dispatch.md
  - docs/how-to/econ.md
  - docs/how-to/service.md
  - docs/how-to/webhooks.md
autonomous: true
requirements:
  - DOCS-07
must_haves:
  truths:
    - "Python API reference pages exist for all 20 subsystems, each rendered via mkdocstrings"
    - "`uv run mkdocs build --strict` passes with mkdocstrings resolving zeroth.core.* modules"
    - "Every Phase 31 Usage Guide 'Reference cross-link' section points to a real (non-stub) python-api/<subsystem>.md page"
    - "docs/reference/python-api.md is a landing page listing all 20 subsystem reference pages"
  artifacts:
    - path: "pyproject.toml"
      provides: "mkdocstrings[python] added to [docs] extra"
      contains: "mkdocstrings"
    - path: "mkdocs.yml"
      provides: "mkdocstrings plugin config + nav entries for 20 python-api pages"
      contains: "mkdocstrings"
    - path: "docs/reference/python-api.md"
      provides: "Python API landing page with TOC of all subsystems"
    - path: "docs/reference/python-api/graph.md"
      provides: "Graph subsystem API reference"
      contains: "::: zeroth.core.graph"
  key_links:
    - from: "docs/how-to/graph.md"
      to: "docs/reference/python-api/graph.md"
      via: "relative markdown link in 'Reference cross-link' section"
      pattern: "reference/python-api/graph"
    - from: "mkdocs.yml"
      to: "mkdocstrings plugin"
      via: "plugins block"
      pattern: "mkdocstrings"
---

<objective>
Wire up mkdocstrings + Griffe and produce per-subsystem Python API reference pages for the full `zeroth.core.*` public surface (20 subsystems). Replace the Phase 30 stub with a real landing page, add nav entries, and update Phase 31 Usage Guides so the "Reference cross-link" sections point at actual pages.

Purpose: Closes DOCS-07. Gives users auto-generated, always-current API docs driven by docstrings (≥90% coverage already gated by interrogate in Phase 27).

Output: 21 new markdown files under `docs/reference/python-api/`, updated `mkdocs.yml` (plugin + nav), updated `pyproject.toml` (`[docs]` extra), updated cross-links in 20 Usage Guides.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-CONTEXT.md

@pyproject.toml
@mkdocs.yml
@docs/reference/python-api.md
@docs/how-to/graph.md

<interfaces>
Phase 31 Usage Guides all end with this block (see docs/how-to/graph.md:74):

```markdown
## Reference cross-link

See the [Python API reference for `zeroth.core.<subsystem>`](../reference/python-api.md#zerothcore<subsystem>) (generated in Phase 32).
```

This plan replaces the `../reference/python-api.md#zerothcore<subsystem>` anchor with `../reference/python-api/<subsystem>.md` and drops the "(generated in Phase 32)" parenthetical.

Subsystem → module mapping (20 pages):

| Subsystem slug | Module |
|---|---|
| graph | zeroth.core.graph |
| orchestrator | zeroth.core.orchestrator |
| agents | zeroth.core.agent_runtime |
| execution-units | zeroth.core.execution_units |
| conditions | zeroth.core.conditions |
| mappings | zeroth.core.mappings |
| memory | zeroth.core.memory |
| storage | zeroth.core.storage |
| contracts | zeroth.core.contracts |
| runs | zeroth.core.runs |
| policy | zeroth.core.policy |
| approvals | zeroth.core.approvals |
| audit | zeroth.core.audit |
| guardrails | zeroth.core.guardrails |
| identity | zeroth.core.identity |
| secrets | zeroth.core.secrets |
| dispatch | zeroth.core.dispatch |
| econ | zeroth.core.econ |
| service | zeroth.core.service |
| webhooks | zeroth.core.webhooks |

Note: "agents" Usage Guide maps to `zeroth.core.agent_runtime` (per Phase 27 rename) — confirm module name at implementation time; fall back to `zeroth.core.agents` if that is the actual path.
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add mkdocstrings to [docs] extra and configure mkdocs.yml plugin</name>
  <files>pyproject.toml, mkdocs.yml</files>
  <action>
    Per D-01 and D-06:

    1. In `pyproject.toml`, add `mkdocstrings[python]>=0.26,<1` to the `[docs]` optional-dependency extra (alongside existing `mkdocs`, `mkdocs-material`, `pymdown-extensions`, `mkdocs-section-index`).

    2. Run `uv sync --extra docs` to install.

    3. In `mkdocs.yml`, add `mkdocstrings` to the `plugins:` list with this configuration:
       ```yaml
       - mkdocstrings:
           default_handler: python
           handlers:
             python:
               paths: [src]
               options:
                 docstring_style: google
                 show_root_heading: true
                 show_root_toc_entry: true
                 show_source: false
                 members_order: source
                 show_signature_annotations: true
                 separate_signature: true
                 merge_init_into_class: true
       ```

       Google style is chosen per pyproject.toml `tool.ruff.lint.pydocstyle.convention = "google"` (D-06).

    4. Do NOT touch the nav section yet — that is Task 3.
  </action>
  <verify>
    <automated>uv run python -c "import mkdocstrings; print(mkdocstrings.__version__)"</automated>
  </verify>
  <done>mkdocstrings installed, pyproject.toml [docs] extra updated, mkdocs.yml plugins block includes mkdocstrings with Google style options. `uv sync --extra docs` succeeds.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Generate 20 subsystem API pages + landing page</name>
  <files>
    docs/reference/python-api.md,
    docs/reference/python-api/graph.md,
    docs/reference/python-api/orchestrator.md,
    docs/reference/python-api/agents.md,
    docs/reference/python-api/execution-units.md,
    docs/reference/python-api/conditions.md,
    docs/reference/python-api/mappings.md,
    docs/reference/python-api/memory.md,
    docs/reference/python-api/storage.md,
    docs/reference/python-api/contracts.md,
    docs/reference/python-api/runs.md,
    docs/reference/python-api/policy.md,
    docs/reference/python-api/approvals.md,
    docs/reference/python-api/audit.md,
    docs/reference/python-api/guardrails.md,
    docs/reference/python-api/identity.md,
    docs/reference/python-api/secrets.md,
    docs/reference/python-api/dispatch.md,
    docs/reference/python-api/econ.md,
    docs/reference/python-api/service.md,
    docs/reference/python-api/webhooks.md
  </files>
  <action>
    Per D-01:

    1. Replace `docs/reference/python-api.md` (currently a single "TBD" line) with a landing page:

       ```markdown
       # Python API Reference

       Auto-generated from docstrings via [mkdocstrings](https://mkdocstrings.github.io/) + [Griffe](https://mkdocstrings.github.io/griffe/). Every public symbol in `zeroth.core.*` is documented here, grouped by subsystem.

       ## Subsystems

       ### Execution core
       - [Graph](python-api/graph.md) — `zeroth.core.graph`
       - [Orchestrator](python-api/orchestrator.md) — `zeroth.core.orchestrator`
       - [Agents](python-api/agents.md) — `zeroth.core.agent_runtime`
       - [Execution units](python-api/execution-units.md) — `zeroth.core.execution_units`
       - [Conditions](python-api/conditions.md) — `zeroth.core.conditions`

       ### Data & state
       - [Mappings](python-api/mappings.md) — `zeroth.core.mappings`
       - [Memory](python-api/memory.md) — `zeroth.core.memory`
       - [Storage](python-api/storage.md) — `zeroth.core.storage`
       - [Contracts](python-api/contracts.md) — `zeroth.core.contracts`
       - [Runs](python-api/runs.md) — `zeroth.core.runs`

       ### Governance
       - [Policy](python-api/policy.md) — `zeroth.core.policy`
       - [Approvals](python-api/approvals.md) — `zeroth.core.approvals`
       - [Audit](python-api/audit.md) — `zeroth.core.audit`
       - [Guardrails](python-api/guardrails.md) — `zeroth.core.guardrails`
       - [Identity](python-api/identity.md) — `zeroth.core.identity`

       ### Platform
       - [Secrets](python-api/secrets.md) — `zeroth.core.secrets`
       - [Dispatch](python-api/dispatch.md) — `zeroth.core.dispatch`
       - [Economics](python-api/econ.md) — `zeroth.core.econ`
       - [Service](python-api/service.md) — `zeroth.core.service`
       - [Webhooks](python-api/webhooks.md) — `zeroth.core.webhooks`

       ## How this is generated

       Pages are rendered at build time from Python docstrings. See `mkdocs.yml` (`mkdocstrings` plugin) for configuration. Docstring coverage is gated at ≥90% via `interrogate` (see Phase 27).
       ```

    2. For each subsystem, create `docs/reference/python-api/<slug>.md` with exactly this body (using the mapping in <interfaces>):

       ```markdown
       # <Human Name>

       ::: zeroth.core.<module>
           options:
             show_root_heading: true
             members_order: source
       ```

       Example `docs/reference/python-api/graph.md`:
       ```markdown
       # Graph

       ::: zeroth.core.graph
           options:
             show_root_heading: true
             members_order: source
       ```

    3. For the `agents.md` page, target `zeroth.core.agent_runtime` (the actual module per Phase 27 rename, visible in `src/zeroth/core/agent_runtime/`). Verify the directory exists before writing; if the public export is re-exposed as `zeroth.core.agents`, prefer that.

    4. Do NOT write any body content beyond the `::: ` directive — mkdocstrings fills the page from docstrings. Missing docstrings render gracefully per D-01.

    5. Run `uv run mkdocs build --strict` and fix any unresolved-import warnings (e.g., if a subsystem has no top-level package, target its concrete module like `zeroth.core.agent_runtime.agents`).
  </action>
  <verify>
    <automated>uv run mkdocs build --strict 2>&1 | tee /tmp/mkdocs-32-01.log && test -f site/reference/python-api/graph/index.html && test -f site/reference/python-api/webhooks/index.html</automated>
  </verify>
  <done>21 markdown files exist under docs/reference/python-api/ (+ landing), strict mkdocs build succeeds, rendered HTML contains module class/function signatures from docstrings.</done>
</task>

<task type="auto">
  <name>Task 3: Wire nav + update 20 Usage Guide cross-links</name>
  <files>
    mkdocs.yml,
    docs/how-to/graph.md,
    docs/how-to/orchestrator.md,
    docs/how-to/agents.md,
    docs/how-to/execution-units.md,
    docs/how-to/conditions.md,
    docs/how-to/mappings.md,
    docs/how-to/memory.md,
    docs/how-to/storage.md,
    docs/how-to/contracts.md,
    docs/how-to/runs.md,
    docs/how-to/policy.md,
    docs/how-to/approvals.md,
    docs/how-to/audit.md,
    docs/how-to/guardrails.md,
    docs/how-to/identity.md,
    docs/how-to/secrets.md,
    docs/how-to/dispatch.md,
    docs/how-to/econ.md,
    docs/how-to/service.md,
    docs/how-to/webhooks.md
  </files>
  <action>
    Per D-01 and D-08:

    1. In `mkdocs.yml`, under `nav:` → `Reference:`, replace the existing `- Python API: reference/python-api.md` entry with a nested structure:
       ```yaml
       - Python API:
         - reference/python-api.md
         - Graph: reference/python-api/graph.md
         - Orchestrator: reference/python-api/orchestrator.md
         - Agents: reference/python-api/agents.md
         - Execution units: reference/python-api/execution-units.md
         - Conditions: reference/python-api/conditions.md
         - Mappings: reference/python-api/mappings.md
         - Memory: reference/python-api/memory.md
         - Storage: reference/python-api/storage.md
         - Contracts: reference/python-api/contracts.md
         - Runs: reference/python-api/runs.md
         - Policy: reference/python-api/policy.md
         - Approvals: reference/python-api/approvals.md
         - Audit: reference/python-api/audit.md
         - Guardrails: reference/python-api/guardrails.md
         - Identity: reference/python-api/identity.md
         - Secrets: reference/python-api/secrets.md
         - Dispatch: reference/python-api/dispatch.md
         - Economics: reference/python-api/econ.md
         - Service: reference/python-api/service.md
         - Webhooks: reference/python-api/webhooks.md
       ```
       Leave `HTTP API` and `Configuration` entries alone — Plans 32-02 and 32-03 own those.

    2. For each of the 20 Usage Guides under `docs/how-to/<slug>.md`, update the "Reference cross-link" section. The current pattern (see docs/how-to/graph.md:74-76) is:
       ```markdown
       ## Reference cross-link

       See the [Python API reference for `zeroth.core.<x>`](../reference/python-api.md#zerothcore<x>) (generated in Phase 32).
       ```

       Replace with:
       ```markdown
       ## Reference cross-link

       See the [Python API reference for `zeroth.core.<x>`](../reference/python-api/<slug>.md).
       ```

       Use the subsystem→slug mapping from <interfaces>. Drop the "(generated in Phase 32)" parenthetical.

       Note the `execution-units.md` Usage Guide links to `../reference/python-api/execution-units.md` (hyphen, not underscore — match the doc filename).

    3. Run `uv run mkdocs build --strict` — no broken links, no missing nav targets.
  </action>
  <verify>
    <automated>uv run mkdocs build --strict 2>&1 | grep -Ei "(warn|error)" | grep -v "INFO" ; uv run mkdocs build --strict</automated>
  </verify>
  <done>mkdocs.yml nav shows all 20 Python API pages, every Usage Guide links to its real reference page (zero references to "#zerothcore" anchors remain), strict build green.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build --strict` passes
- `grep -r "generated in Phase 32" docs/` returns zero matches
- `grep -r "reference/python-api.md#zerothcore" docs/` returns zero matches
- `ls docs/reference/python-api/*.md | wc -l` == 20
- Rendered `site/reference/python-api/graph/index.html` contains real symbol docs (not a TBD placeholder)
</verification>

<success_criteria>
DOCS-07 satisfied: Python API Reference is auto-generated from docstrings via mkdocstrings + Griffe for the full `zeroth.core.*` public surface, cross-linked from narrative pages.
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-01-SUMMARY.md`
</output>
