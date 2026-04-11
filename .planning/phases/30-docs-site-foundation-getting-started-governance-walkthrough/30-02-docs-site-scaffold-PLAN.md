---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - pyproject.toml
  - mkdocs.yml
  - docs/index.md
  - docs/tutorials/index.md
  - docs/tutorials/getting-started/index.md
  - docs/tutorials/getting-started/01-install.md
  - docs/tutorials/getting-started/02-first-graph.md
  - docs/tutorials/getting-started/03-service-and-approval.md
  - docs/tutorials/governance-walkthrough.md
  - docs/how-to/index.md
  - docs/concepts/index.md
  - docs/reference/index.md
  - docs/reference/python-api.md
  - docs/reference/http-api.md
  - docs/reference/configuration.md
  - tests/test_docs_phase30.py
autonomous: true
requirements:
  - SITE-01
  - SITE-04
  - DOCS-01
tags: [docs, mkdocs, site-scaffold, diataxis]
must_haves:
  truths:
    - "`uv sync --extra docs` installs mkdocs, mkdocs-material, pymdown-extensions, mkdocs-section-index from the new [docs] extra"
    - "`uv run mkdocs build --strict` from repo root produces site/ with zero warnings"
    - "Navigation has exactly four top-level Diátaxis sections with the canonical names: Tutorials, How-to Guides, Concepts, Reference"
    - "Landing page `docs/index.md` embeds `examples/hello.py` via pymdownx.snippets, shows an install snippet, and has a tabbed Choose Your Path split (library vs service)"
    - "Reference quadrant has three stubbed pages, each containing only a one-line 'TBD — populated in Phase 32' note (no fake content)"
    - "Search plugin is enabled and `site_url` is set so the auto-generated sitemap is correct"
    - "Tutorial content pages (Getting Started 01-03, governance-walkthrough.md) exist as placeholder skeletons — plans 03 and 04 fill the real content"
  artifacts:
    - path: "pyproject.toml"
      provides: "[project.optional-dependencies].docs extra pinning mkdocs>=1.6,<2 / mkdocs-material>=9.7.6,<10 / pymdown-extensions>=10.8 / mkdocs-section-index>=0.3.9"
    - path: "mkdocs.yml"
      provides: "site_name, site_url, theme=material, plugins=[search, section-index], pymdownx.snippets with check_paths=true and base_path=[., examples], pymdownx.tabbed, pymdownx.superfences, and the 4-section Diátaxis nav"
    - path: "docs/index.md"
      provides: "Landing page — 10-line hello-world snippet embed, install snippet, Choose Your Path tabbed split"
    - path: "docs/reference/python-api.md"
      provides: "Stub: 'TBD — populated in Phase 32 via mkdocstrings.'"
    - path: "docs/reference/http-api.md"
      provides: "Stub: 'TBD — populated in Phase 32 from openapi/zeroth-core-openapi.json.'"
    - path: "docs/reference/configuration.md"
      provides: "Stub: 'TBD — populated in Phase 32 from pydantic-settings.'"
  key_links:
    - from: "mkdocs.yml"
      to: "examples/hello.py"
      via: "pymdownx.snippets base_path + --8<-- in docs/index.md"
      pattern: "base_path.*examples"
    - from: "tests/test_docs_phase30.py"
      to: "mkdocs.yml"
      via: "yaml.safe_load + nav shape assertions"
      pattern: "yaml\\.safe_load"
---

<objective>
Scaffold the mkdocs-material site with explicit Diátaxis IA, wire the
`[docs]` extra into `pyproject.toml`, ship the landing page with the
Choose Your Path tabbed split, and stub every other content page so the
site builds cleanly under `mkdocs build --strict`. Plans 03 and 04 fill
in the tutorial content; this plan gives them a working scaffold to drop
files into.

Purpose: Ship SITE-01 (Diátaxis IA), SITE-04 (search + sitemap), and
DOCS-01 (landing page) in a single self-contained plan. SITE-02 (GHA
deploy) is plan 05 because it depends on all content being present.

Output: A working mkdocs site, `uv run mkdocs serve` previewable at
http://127.0.0.1:8000, with the full Diátaxis nav and stubbed tutorial
pages ready for plans 03/04 to replace.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-CONTEXT.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-RESEARCH.md
@pyproject.toml
@examples/hello.py
@README.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add [docs] extra + mkdocs.yml + docs/ tree scaffold</name>
  <files>
    pyproject.toml
    mkdocs.yml
    docs/index.md
    docs/tutorials/index.md
    docs/tutorials/getting-started/index.md
    docs/tutorials/getting-started/01-install.md
    docs/tutorials/getting-started/02-first-graph.md
    docs/tutorials/getting-started/03-service-and-approval.md
    docs/tutorials/governance-walkthrough.md
    docs/how-to/index.md
    docs/concepts/index.md
    docs/reference/index.md
    docs/reference/python-api.md
    docs/reference/http-api.md
    docs/reference/configuration.md
  </files>
  <action>
    1. Edit `pyproject.toml` — add a new entry under
       `[project.optional-dependencies]`:

       ```toml
       docs = [
           "mkdocs>=1.6,<2.0",
           "mkdocs-material>=9.7.6,<10",
           "pymdown-extensions>=10.8",
           "mkdocs-section-index>=0.3.9",
       ]
       ```

       Do NOT add `docs` into the existing `all` extra (all = production
       runtime backends; docs is dev-only). Before pinning `mkdocs-material`,
       run `uv pip index versions mkdocs-material` and adopt the latest 9.x
       if newer than 9.7.6 is out; stay `<10`.

    2. Run `uv sync --extra docs` and confirm the lock resolves.

    3. Create `mkdocs.yml` at repo root using the config from
       30-RESEARCH.md "Pattern 1" verbatim, with these concrete values:
         - `site_name: Zeroth`
         - `site_url: https://rrrozhd.github.io/zeroth/`   # per resolved_open_questions
         - `site_description:` copied from pyproject description field
         - `repo_url: https://github.com/rrrozhd/zeroth-core`
         - `repo_name: rrrozhd/zeroth-core`
         - `edit_uri: edit/main/docs/`
         - `plugins: [search, section-index]`
         - `pymdownx.snippets` with `check_paths: true` and
           `base_path: [".", "examples"]`
         - `pymdownx.tabbed` with `alternate_style: true`
         - Full `nav:` block with FOUR canonical top-level Diátaxis
           section names, matching the file tree below.

    4. Create the `docs/` tree (use Write tool per file):

       - `docs/index.md` — landing page. Include:
         * H1: "Zeroth"
         * Short tagline (one line from README).
         * An install code block (`pip install zeroth-core`).
         * A `pymdownx.tabbed` block:
           ```
           === "Embed as library"
               Placeholder — plan 03 will replace with links/snippets.
               [Getting Started →](tutorials/getting-started/)
           === "Run as service"
               Placeholder — plan 03 will replace with links/snippets.
               [Service mode walkthrough →](tutorials/getting-started/03-service-and-approval.md)
           ```
         * A code block with the hello-world snippet embed:
           ```python title="examples/hello.py"
           --8<-- "hello.py"
           ```
           (snippets base_path already includes `examples`, so the path
           is relative to base_path.)

       - `docs/tutorials/index.md` — section landing, one paragraph
         explaining Diátaxis "learning-oriented" and linking to the two
         tutorials (Getting Started + Governance Walkthrough).

       - `docs/tutorials/getting-started/index.md` — overview with time
         budget ("<5 min first output, <30 min total") and a TOC-style
         list of the three sections. Plan 03 will replace the body.

       - `docs/tutorials/getting-started/01-install.md` — placeholder
         with H1 "Install" and a one-line note "Content added in plan
         30-03." Include the hello.py snippet embed so the strict build
         has something real to verify.

       - `docs/tutorials/getting-started/02-first-graph.md` — placeholder
         H1 "First graph with an agent, a tool, and an LLM call" + one
         line "Content added in plan 30-03."

       - `docs/tutorials/getting-started/03-service-and-approval.md` —
         placeholder H1 + "Content added in plan 30-03."

       - `docs/tutorials/governance-walkthrough.md` — placeholder H1
         "Governance Walkthrough" + "Content added in plan 30-04."

       - `docs/how-to/index.md` — Diátaxis landing: "How-to Guides are
         task-oriented. Populated in Phase 31."

       - `docs/concepts/index.md` — "Concept pages are
         understanding-oriented. Populated in Phase 31."

       - `docs/reference/index.md` — Diátaxis landing listing the three
         subsections.

       - `docs/reference/python-api.md` — single line:
         "TBD — populated in Phase 32 via mkdocstrings."

       - `docs/reference/http-api.md` — single line:
         "TBD — populated in Phase 32 from `openapi/zeroth-core-openapi.json`."

       - `docs/reference/configuration.md` — single line:
         "TBD — populated in Phase 32 from pydantic-settings."

    5. Run `uv run mkdocs build --strict`. It MUST succeed with zero
       warnings. If `check_paths: true` complains about a missing snippet,
       the most likely cause is the `--8<-- "hello.py"` path resolution
       against `base_path`. Fix by adjusting the snippet path (try
       `examples/hello.py` without the base_path trick or keep the trick
       and the bare filename — do NOT disable check_paths).

    6. Run `uv run mkdocs serve --dirtyreload` briefly in a subshell to
       sanity-check the nav renders; kill it immediately. Not required
       for CI — only for the executor's own verification.
  </action>
  <verify>
    <automated>uv sync --extra docs &amp;&amp; uv run mkdocs build --strict</automated>
  </verify>
  <done>
    - `pyproject.toml` has a `[docs]` extra, `uv sync --extra docs` succeeds
    - `mkdocs.yml` exists at repo root with four Diátaxis sections, search + section-index plugins, pymdownx.snippets with check_paths=true, and site_url set
    - All 14 stub docs pages exist
    - `uv run mkdocs build --strict` completes with zero warnings
    - `site/sitemap.xml` exists after build and contains an entry for `https://rrrozhd.github.io/zeroth/`
  </done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Add mkdocs config shape tests to tests/test_docs_phase30.py</name>
  <files>
    tests/test_docs_phase30.py
  </files>
  <behavior>
    - `test_mkdocs_config_has_four_diataxis_sections`: parses mkdocs.yml,
      asserts top-level nav contains exactly {Tutorials, How-to Guides,
      Concepts, Reference} (ignoring the Home entry).
    - `test_mkdocs_config_has_search_plugin`: asserts `search` in
      `plugins` list.
    - `test_mkdocs_config_has_snippets_check_paths_true`: asserts the
      pymdownx.snippets extension is configured with `check_paths: true`
      and `base_path` includes `examples`.
    - `test_mkdocs_site_url_set`: asserts `site_url` is non-empty and
      begins with `https://`.
    - `test_docs_extra_declared_in_pyproject`: parses pyproject.toml,
      asserts `optional-dependencies.docs` exists and contains
      `mkdocs-material`.
    - `test_landing_page_has_tabbed_split_and_hello_snippet`: reads
      `docs/index.md`, asserts both `=== "Embed as library"` and
      `=== "Run as service"` strings present AND the `--8<--` scissors
      token appears (with `hello.py`).
    - `test_reference_quadrant_stubs_are_minimal`: reads the three
      reference stub pages, asserts each contains "TBD" or "Phase 32"
      and is shorter than 400 chars (prevents fake content rot).
  </behavior>
  <action>
    1. Extend `tests/test_docs_phase30.py` (created in plan 01) with the
       seven tests listed above. Use `PyYAML` (already a base dep) to
       parse mkdocs.yml and the stdlib `tomllib` (Python 3.11+) to parse
       pyproject.toml.
    2. Guard against running in an environment where mkdocs.yml does
       not yet exist by using `pathlib.Path(__file__).parents[1] /
       "mkdocs.yml"` — do NOT use CWD-relative paths.
    3. Run `uv run pytest tests/test_docs_phase30.py -v`. All tests must
       pass.
    4. Run `uv run ruff check tests/test_docs_phase30.py` and format.
  </action>
  <verify>
    <automated>uv run pytest tests/test_docs_phase30.py -v &amp;&amp; uv run ruff check tests/test_docs_phase30.py</automated>
  </verify>
  <done>
    - 7 new shape tests pass
    - Tests locate config files via absolute repo-root paths, not CWD
    - Ruff clean
  </done>
</task>

</tasks>

<verification>
- `uv run mkdocs build --strict` → clean build
- `uv run pytest tests/test_docs_phase30.py -v` → all green
- `grep -c "TBD" docs/reference/*.md` → 3
- `site/sitemap.xml` lists `https://rrrozhd.github.io/zeroth/`
</verification>

<success_criteria>
- SITE-01: mkdocs-material with four Diátaxis sections, canonical names
- SITE-04: search plugin enabled + sitemap generated + site_url set
- DOCS-01: landing page has 10-line hello snippet embed + install snippet
  + Choose Your Path tabbed split
- Strict build is green and enforced by CI (wiring in plan 05)
- Reference stubs exist and are minimal (prevent fake content rot)
</success_criteria>

<output>
After completion, create `.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-SUMMARY.md` documenting: the final pinned versions, mkdocs.yml structure decisions (palette, features used), and any snippet-path gotchas encountered.
</output>
