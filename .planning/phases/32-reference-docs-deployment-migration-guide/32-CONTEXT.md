# Phase 32: Reference Docs, Deployment & Migration Guide - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning
**Mode:** Inline discuss (autonomous workflow)

<domain>
## Phase Boundary

Complete the Reference quadrant of the docs site with auto-generated content (Python API, HTTP API, Configuration), then write the Deployment Guide covering all supported modes and the Migration Guide from the monolith layout.

This is the final phase of v3.0. It fills the Reference stubs created in Phase 30 and adds two narrative guides. All content is either auto-generated from source or hand-written narrative — no new library code.

</domain>

<decisions>
## Implementation Decisions

### D-01 Python API Reference — mkdocstrings + Griffe, full public surface
- Add `mkdocstrings[python]` to the `[docs]` extra
- Config mkdocs.yml with mkdocstrings plugin, Griffe handler
- Generate one page per major subsystem, each using `::: zeroth.core.<subsystem>` syntax
- Full public surface — all 20 subsystems from Phase 31 get a Reference page
- Missing docstrings render gracefully (mkdocstrings handles this)
- Cross-linked from Phase 31 Usage Guide "Reference cross-link" sections
- Replace `docs/reference/python-api.md` stub with a landing page listing all subsystem reference pages

### D-02 HTTP API Reference — Swagger UI embed
- Use the committed `openapi/zeroth-core-openapi.json` snapshot (from Phase 29-01)
- Embed interactive Swagger UI via `neoteroi-mkdocs` plugin (if available) or a static `<iframe>` / `<redoc>` element
- `scripts/dump_openapi.py` regenerates the snapshot; CI drift-check (reuse or extend the pattern from Phase 29's zeroth-studio workflow)
- Replace `docs/reference/http-api.md` stub

### D-03 Configuration Reference — introspect pydantic-settings
- Write `scripts/dump_config.py` that:
  - Imports `zeroth.core.config.settings` (and any sub-Settings classes)
  - Introspects via pydantic model_fields to extract: env var name, type, default, description, secret flag
  - Emits `docs/reference/configuration.md` as a markdown table
- Hook into CI: run the script, `git diff --exit-code` fails if drift
- Replace `docs/reference/configuration.md` stub

### D-04 Deployment Guide — all 5 modes
- `docs/how-to/deployment/index.md` — landing page
- `docs/how-to/deployment/local-dev.md` — `uv sync && uv run zeroth-core serve`
- `docs/how-to/deployment/docker-compose.md` — reuse docker-compose.yml from the repo
- `docs/how-to/deployment/standalone-service.md` — uvicorn entrypoint + reverse proxy
- `docs/how-to/deployment/embedded-library.md` — import zeroth.core in host app
- `docs/how-to/deployment/with-regulus.md` — enabling Regulus econ companion service

Each page ~400 words with runnable command blocks.

### D-05 Migration Guide — monolith to zeroth.core.*
- `docs/how-to/migration-from-monolith.md` — single comprehensive page
- Covers:
  1. Import rename pattern (`from zeroth import X` → `from zeroth.core import X`) with grep+sed recipe
  2. econ SDK path swap (local path → PyPI `econ-instrumentation-sdk>=0.1.1`)
  3. Env var changes (if any renamed between monolith and zeroth.core)
  4. Docker image retag guidance
- Concrete before/after examples

### D-06 mkdocstrings plugin config
- `mkdocs.yml` plugins section adds `mkdocstrings` with:
  - `python` handler
  - `options.docstring_style: google` (or numpy, whichever matches the codebase — planner checks)
  - `options.show_root_heading: true`
  - `options.show_source: false` (keep pages clean)
  - `options.members_order: source`

### D-07 CI drift gates
- Extend `.github/workflows/docs.yml` (from Phase 30) to also:
  - Run `python scripts/dump_openapi.py --check` (fails if openapi.json is stale)
  - Run `python scripts/dump_config.py --check` (fails if configuration.md is stale)
  - Run `uv run mkdocs build --strict` (must remain green)

### D-08 Navigation update
- Update `mkdocs.yml` nav to include:
  - Reference: Python API (20 subsystem pages), HTTP API, Configuration
  - How-to → Deployment (5 mode pages)
  - How-to → Migration (1 page)

### Claude's Discretion
- Exact mkdocstrings options and whether to generate one big page or per-subsystem pages (recommend per-subsystem for cross-linking)
- Whether Swagger UI via neoteroi-mkdocs or a static iframe/redoc tag
- Whether the HTTP API drift check uses the zeroth-core-side script (D-02) or extends an existing gate
- Page ordering inside Deployment section
- Whether to add a "Troubleshooting" sidebar to Migration Guide (nice-to-have)

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `scripts/dump_openapi.py` already exists (created in Phase 29-01)
- `openapi/zeroth-core-openapi.json` already committed (from Phase 29-01)
- `src/zeroth/core/config/settings.py` — pydantic-settings source of truth
- `mkdocs.yml` from Phase 30 with pymdownx.snippets and pymdownx.superfences
- `docs/reference/{python-api,http-api,configuration}.md` — stubs from Phase 30 to replace
- `.github/workflows/docs.yml` — extend with drift gates
- `docker-compose.yml` — reference for deployment guide
- `README.md` — has an Install section to cross-link from migration guide

### Established Patterns
- uv for Python ops
- mkdocs build --strict as the gate
- Phase 28 pattern for CI drift checks
- Diátaxis four quadrants

### Integration Points
- Phase 31 Usage Guides already have "Reference cross-link" placeholder \u2014 update those to link to new Python API pages
- mkdocs.yml nav must extend without breaking Phase 30/31 entries
- Phase 28's `econ-instrumentation-sdk>=0.1.1` pin is referenced in the migration guide

</code_context>

<specifics>
## Specific Ideas

- Migration guide's import rename recipe: `grep -rl "from zeroth\." src/ | xargs sed -i '' 's/from zeroth\./from zeroth.core./g'`
- Docker image retag: document the new registry path if one exists
- Configuration reference table columns: Env Var | Type | Default | Secret | Description
- Reference landing pages (python-api, http-api, configuration) each have a brief intro + TOC
- Keep the Reference quadrant discoverable from both the landing page and the Concepts/How-to section cross-links

</specifics>

<deferred>
## Deferred Ideas

- Versioned docs (mike)
- Custom domain
- API changelog auto-generation from git tags
- Per-release migration guides (this phase only documents monolith → zeroth.core, not future migrations)
- Advanced mkdocstrings customization (cross-refs, inherited member handling)
- Sphinx-style intersphinx to external libs
- Interactive configuration generator (defer)

</deferred>
