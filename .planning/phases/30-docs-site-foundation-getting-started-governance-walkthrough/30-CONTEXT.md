# Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning
**Mode:** Inline discuss (autonomous workflow)

<domain>
## Phase Boundary

Stand up the zeroth-core documentation site as a public, CI-deployed mkdocs-material build with explicit Diátaxis IA, then ship the complete "first working path": a landing page, a 3-section Getting Started tutorial, and a Governance Walkthrough that demonstrates Zeroth's differentiator (approval gate, auditor, policy block).

This phase ships the docs SITE infrastructure plus the FIRST tutorial content. Subsystem concept pages, usage guides, cookbook, and auto-generated reference content come in Phases 31 and 32.

</domain>

<decisions>
## Implementation Decisions

### D-01 Hosting platform
- **GitHub Pages** — free, native GH Actions integration, easy custom domain later
- Default URL: `https://rrrozhd.github.io/zeroth/` (the existing zeroth repo)
- Custom domain deferred (D-02)

### D-02 Domain strategy
- **Default URL for now** (`rrrozhd.github.io/zeroth`)
- Custom domain (e.g. `zeroth.dev`) deferred to a follow-up — DNS not configured today

### D-03 Docs source location
- **`docs/` directory in the zeroth repo** (single source of truth, version-locked with code)
- mkdocs config at repo root: `mkdocs.yml`

### D-04 Static site generator
- **mkdocs-material** (latest stable) with Diátaxis IA
- Required plugins: `search` (built-in), `material[imaging]` if needed for social cards (defer if heavy), `mkdocs-material` extensions

### D-05 IA structure (Diátaxis four quadrants)
- `Tutorials/` — learning-oriented (Getting Started + Governance Walkthrough live here in Phase 30)
- `How-to Guides/` — task-oriented (placeholder section, content in Phase 31)
- `Concepts/` — understanding-oriented (placeholder, Phase 31)
- `Reference/` — information-oriented (stubbed subsections in Phase 30 per D-12)

### D-06 CI/CD
- GitHub Actions workflow `.github/workflows/docs.yml`
- On push to `main`: build + deploy to `gh-pages` branch (via `peaceiris/actions-gh-pages` or `mkdocs gh-deploy`)
- On PR: build only (no deploy) — verifies the build passes
- (PR preview deploys via GitHub Pages are not natively supported; deferred. Deploy gate on `main` is sufficient.)

### D-07 Landing page
- 10-line hello-world snippet
- Install snippet (`pip install zeroth-core`)
- "Choose your path" split: **Embed as library** vs **Run as service**
- Each path links to its respective Getting Started section

### D-08 Getting Started structure
- Single linear 3-section tutorial:
  1. **Install** — `pip install zeroth-core[all]`, verify with hello example
  2. **First graph with one agent/tool/LLM** — minimal multi-step graph with one LLM call and one tool call
  3. **Run in service mode with approval gate** — boot zeroth-core as a service, send a run, approve via CLI prompt
- Time targets: first working output <5 min, full tutorial <30 min

### D-09 Example LLM provider
- **OpenAI** (`OPENAI_API_KEY` gated)
- Same litellm fallback pattern as `examples/hello.py` for portability
- Tutorial explicitly notes: works with any litellm-supported provider; OpenAI shown for familiarity

### D-10 Approval gate UX
- **CLI-driven** — service mode example pauses execution, prints an approval URL + token, user approves via `curl` or a small `zeroth approve <run_id>` CLI command
- No Studio UI dependency (Studio ships separately, lower friction for tutorial readers)

### D-11 Governance Walkthrough
- End-to-end tutorial demonstrating three Zeroth differentiators:
  1. **Approval gate** stops execution mid-graph
  2. **Auditor** reviews the trail (logs/decisions/inputs/outputs visible in a query/CLI)
  3. **Policy** blocks a tool call before execution
- All three exercised against a single example workflow
- Lives in `docs/tutorials/governance-walkthrough.md`

### D-12 Reference quadrant in Phase 30
- **Stub each subsection** — empty pages for Python API / HTTP API / Configuration
- Each stub has a one-line "TBD — populated in Phase 32" note
- Phase 32 fills these via mkdocstrings, OpenAPI render, and pydantic-settings auto-gen

### D-13 Examples are CI-tested
- Tutorial code lives as runnable `.py` files in `examples/` (extending the `examples/` directory created in Phase 28)
- CI job runs each example end-to-end with mocked or real LLM (gated on env keys)
- mkdocs uses `pymdownx.snippets` or `mkdocs-include-markdown-plugin` to embed the example files into the markdown so docs can never drift from code
- Examples without env keys exit cleanly with SKIP message (matching `examples/hello.py` pattern)

### D-14 mkdocs theme + plugin set (minimum)
- `mkdocs-material` (theme)
- `mkdocs-material[imaging]` if social cards enabled (defer)
- `pymdownx.snippets` for embedding example files
- `pymdownx.superfences` for code blocks with line numbers + diff highlighting
- `pymdownx.tabbed` for "Choose your path" tabs
- `mkdocs-section-index` for clean section landing pages

### Claude's Discretion
- Exact mkdocs.yml config details (palette, font, social links)
- Whether to add a logo / favicon (defer to Phase 31+ if no asset exists)
- Specific approval CLI command name (`zeroth approve` vs `zeroth-cli approve`)
- Whether to add a "What's new" / changelog page in Phase 30 or defer
- Versioned docs via mike — defer (not in this phase's success criteria)

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `examples/hello.py` — exists from Phase 28, sets the convention for env-gated runnable examples
- `pyproject.toml` — has `[all]` extra, can pin docs deps as a `[docs]` extra
- `.github/workflows/release-zeroth-core.yml` and `verify-extras.yml` — Phase 28 reference patterns for new GHA workflow
- `openapi/zeroth-core-openapi.json` — committed in Phase 29-01, will be referenced in Phase 32

### Established Patterns
- Apache-2.0 license, Phase 28 LICENSE/CHANGELOG/CONTRIBUTING conventions
- Conventional Commits
- uv for Python tooling (CLAUDE.md)
- Optional extras pattern for opt-in dependencies

### Integration Points
- mkdocs build needs to import `zeroth.core` to render docstrings (Phase 32) — Phase 30 doesn't need this yet, just installs base + a `[docs]` extra
- examples/ directory grows with tutorial scripts; CI must run them
- README.md should link to the live docs site URL once deployed

</code_context>

<specifics>
## Specific Ideas

- The Diátaxis four-quadrant section names match the canonical names exactly: Tutorials, How-to Guides, Concepts, Reference
- Getting Started must produce visible output in <5 min; the install + verify hello example should be the gate
- Governance Walkthrough is the differentiator demo — make it feel substantive, not toy
- Add a `[docs]` extra to pyproject.toml so devs can `uv sync --extra docs`
- mkdocs.yml `site_url` should be set to the GH Pages URL so canonical URLs and sitemap work
- Add a small "Edit this page on GitHub" link via mkdocs-material's `edit_uri`

</specifics>

<deferred>
## Deferred Ideas

- Custom domain (zeroth.dev or similar) and DNS setup
- PR preview deploys (GitHub Pages doesn't natively support them; would require Cloudflare Pages or Netlify)
- mike for versioned docs
- Social cards (heavy deps; defer to Phase 31+)
- Subsystem concept pages and usage guides (Phase 31)
- Cookbook recipes (Phase 31)
- Auto-generated Python API reference via mkdocstrings (Phase 32)
- Auto-rendered HTTP API reference from openapi.json (Phase 32)
- Auto-generated configuration reference from pydantic-settings (Phase 32)
- Migration Guide from monolith layout (Phase 32)
- Logo / favicon / brand assets

</deferred>
