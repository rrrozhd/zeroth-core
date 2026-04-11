---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 02
subsystem: docs
tags: [docs, mkdocs, site-scaffold, diataxis, landing-page]
requires:
  - zeroth.core.examples.quickstart
provides:
  - mkdocs-material site scaffold with Diataxis IA
  - docs/ tree with 14 stub pages
  - [docs] optional dependency group
  - landing page with Choose-Your-Path tabbed split
  - pymdownx.snippets hello.py embed
  - mkdocs shape test assertions in tests/test_docs_phase30.py
affects:
  - pyproject.toml optional-dependencies
  - .gitignore (adds site/)
tech-stack:
  added:
    - mkdocs>=1.6,<2.0
    - mkdocs-material>=9.7.6,<10
    - pymdown-extensions>=10.8
    - mkdocs-section-index>=0.3.9
  patterns:
    - Diataxis four-quadrant IA (Tutorials / How-to / Concepts / Reference) as canonical nav labels
    - pymdownx.snippets with check_paths=true guarantees docs/code snippet drift breaks the build
    - base_path=[., examples] lets snippet references use bare filenames (e.g. `--8<-- "hello.py"`)
    - mkdocs `exclude_docs` hides pre-existing internal docs/specs/ and docs/superpowers/ dirs from the public site without moving them
    - Reference quadrant stubs stay under 400 chars with a "TBD — populated in Phase 32" note to prevent fake-content rot (enforced by test)
key-files:
  created:
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
  modified:
    - pyproject.toml
    - tests/test_docs_phase30.py
    - .gitignore
    - uv.lock
key-decisions:
  - Adopt Diataxis canonical section names verbatim (Tutorials / How-to Guides / Concepts / Reference) so readers coming from diataxis.fr map to zeroth docs instantly; enforced by a test that asserts the top-level nav label set
  - Pin mkdocs-material >=9.7.6,<10 rather than tracking latest — the 9.7.x line is the stable pre-2.0 release and Material is warning about a breaking v2 rewrite; the <10 cap prevents an unreviewed jump
  - Keep pymdownx.snippets check_paths=true — silent "snippet not found" rendering is the #1 docs-code drift vector; the strict build must fail loudly
  - Exclude pre-existing docs/specs/ and docs/superpowers/ (internal design notes from prior phases) via `exclude_docs` instead of moving them, to avoid touching unrelated paths
  - Do NOT add `docs` into the `all` extra — `all` is production runtime backends, `docs` is dev-only tooling; devs opt in with `uv sync --extra docs`
  - Reference quadrant stubs are one-line "TBD — populated in Phase 32" notes; a shape test enforces they stay <400 chars to block fake-content rot
requirements-completed:
  - SITE-01
  - SITE-04
  - DOCS-01
duration: 10 min
completed: 2026-04-11
---

# Phase 30 Plan 02: Docs Site Scaffold Summary

Scaffolded the zeroth-core mkdocs-material documentation site with explicit Diataxis IA, wired the `[docs]` extra into `pyproject.toml`, shipped the landing page with an `examples/hello.py` snippet embed and a Choose-Your-Path tabbed split (library vs service), and stubbed every tutorial/how-to/concepts/reference page so `uv run mkdocs build --strict` is green and ready for plans 30-03, 30-04, and 30-05 to drop content into.

## What Shipped

### `[docs]` optional dependency group
- New `[project.optional-dependencies].docs` in `pyproject.toml`:
  - `mkdocs>=1.6,<2.0`
  - `mkdocs-material>=9.7.6,<10`
  - `pymdown-extensions>=10.8`
  - `mkdocs-section-index>=0.3.9`
- Installed via `uv sync --extra docs`; resolver + lockfile update committed in `uv.lock`.
- Intentionally NOT added to the `all` extra (runtime backends only).

### `mkdocs.yml`
- `site_name: Zeroth`, `site_url: https://rrrozhd.github.io/zeroth/`, `repo_url`, `edit_uri: edit/main/docs/`.
- `theme: material` with navigation.sections + navigation.tabs + content.code.copy + content.action.edit + search features, indigo palette with light/dark toggle.
- `plugins: [search, section-index]`.
- `markdown_extensions`: admonition, attr_list, md_in_html, toc(permalink), pymdownx.highlight, pymdownx.inlinehilite, **pymdownx.snippets with `check_paths: true` and `base_path: [".", "examples"]`**, pymdownx.superfences, **pymdownx.tabbed with `alternate_style: true`**.
- `exclude_docs:` hides pre-existing `docs/specs/` and `docs/superpowers/` internal notes from the public site.
- `nav:` — four Diataxis top-level sections matching canonical names exactly:
  - Tutorials (Getting Started 01-03 + Governance Walkthrough)
  - How-to Guides (Phase 31 placeholder)
  - Concepts (Phase 31 placeholder)
  - Reference (Python API / HTTP API / Configuration stubs for Phase 32)

### Landing page (`docs/index.md`)
- H1 + tagline, `pip install zeroth-core` install block.
- `pymdownx.tabbed` Choose-Your-Path split: "Embed as library" vs "Run as service" tabs, each linking into the Getting Started tutorial.
- Hello-world snippet embedded via `--8<-- "hello.py"` (resolved through `base_path` → `examples/hello.py`), proving the snippet path works end-to-end against the strict build.

### Docs tree skeletons
- `docs/tutorials/index.md` — Diataxis "learning-oriented" landing + links.
- `docs/tutorials/getting-started/index.md` — overview with <5 min / <30 min time budget + section TOC (Plan 30-03 fills body).
- `docs/tutorials/getting-started/01-install.md` — placeholder with hello.py snippet embed so strict build has something real to verify (Plan 30-03 fills body).
- `docs/tutorials/getting-started/02-first-graph.md`, `03-service-and-approval.md` — Plan 30-03 placeholders.
- `docs/tutorials/governance-walkthrough.md` — Plan 30-04 placeholder.
- `docs/how-to/index.md`, `docs/concepts/index.md` — Phase 31 placeholders.
- `docs/reference/index.md` + `python-api.md` + `http-api.md` + `configuration.md` — all three subsection pages are one-line "TBD — populated in Phase 32" stubs (<400 chars each, enforced by test).

### Shape tests (`tests/test_docs_phase30.py`)
Seven new assertions appended to the Plan 30-01 scaffold (the plan-01 placeholder is preserved):

1. `test_mkdocs_config_has_four_diataxis_sections` — top-level nav labels == {Tutorials, How-to Guides, Concepts, Reference}.
2. `test_mkdocs_config_has_search_plugin` — `search` in `plugins`.
3. `test_mkdocs_config_has_snippets_check_paths_true` — pymdownx.snippets has `check_paths: true` and `examples` in `base_path`.
4. `test_mkdocs_site_url_set` — `site_url` is non-empty and begins with `https://`.
5. `test_docs_extra_declared_in_pyproject` — `optional-dependencies.docs` exists and pins `mkdocs-material`.
6. `test_landing_page_has_tabbed_split_and_hello_snippet` — `docs/index.md` contains both `=== "Embed as library"` and `=== "Run as service"` AND `--8<--` with `hello.py`.
7. `test_reference_quadrant_stubs_are_minimal` — each of the three reference stubs contains `TBD`/`Phase 32` and is shorter than 400 chars.

All files resolve via `Path(__file__).resolve().parents[1]` so tests work regardless of pytest invocation CWD.

## Tasks & Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add [docs] extra + mkdocs.yml + docs/ tree scaffold | `d7744b3` | `pyproject.toml`, `mkdocs.yml`, `docs/index.md`, `docs/tutorials/**`, `docs/how-to/index.md`, `docs/concepts/index.md`, `docs/reference/**`, `.gitignore`, `uv.lock` |
| 2 | Add mkdocs config shape tests to tests/test_docs_phase30.py | `dcfb5ab` | `tests/test_docs_phase30.py` |

## Verification Results

- `uv sync --extra docs` → resolver succeeds, lockfile updated.
- `uv run mkdocs build --strict` → **exit 0**, "Documentation built in 0.16 seconds", zero warnings. (The red-bordered "MkDocs 2.0" notice printed to stderr is an informational message from the mkdocs-material team about a future upstream rewrite; it is NOT a mkdocs strict-mode warning and does not affect the build exit code. `DISABLE_MKDOCS_2_WARNING=true` will suppress it in CI.)
- `grep -c https://rrrozhd.github.io/zeroth/ site/sitemap.xml` → **13** canonical URLs listed (SITE-04 sitemap verified).
- `grep -c TBD docs/reference/*.md` → **3** stubs flagged for Phase 32 (python-api, http-api, configuration; index has 0 which is expected).
- `uv run pytest tests/test_docs_phase30.py -v` → **8 passed** (1 scaffold placeholder from plan 01 + 7 new Plan 30-02 shape tests).
- `uv run ruff check tests/test_docs_phase30.py` → clean.

## Success Criteria

- [x] SITE-01: mkdocs-material with four Diataxis sections, canonical names — verified by `test_mkdocs_config_has_four_diataxis_sections`.
- [x] SITE-04: search plugin enabled + sitemap generated + `site_url` set — 13 URLs in sitemap.xml, `search` plugin enabled, `site_url: https://rrrozhd.github.io/zeroth/` set.
- [x] DOCS-01: landing page has hello snippet embed + install snippet + Choose Your Path tabbed split — verified by `test_landing_page_has_tabbed_split_and_hello_snippet`.
- [x] Reference quadrant has three stubbed pages, each minimal "TBD — populated in Phase 32" (<400 chars, enforced by test).
- [x] `uv run mkdocs build --strict` is green (CI wiring deferred to Plan 30-05).
- [x] Tutorial content skeletons exist for Plans 30-03/30-04 to replace in place.

## Deviations from Plan

**[Rule 3 - Blocking] Exclude pre-existing docs/specs/ and docs/superpowers/ from strict build**

- **Found during:** Task 1
- **Issue:** `docs/` already contained pre-existing internal design notes (`docs/specs/phase-6-identity-access-model.md`, `docs/superpowers/plans/...`, etc.) left over from prior phases. With `--strict` on, mkdocs treats any docs-tree file not referenced in `nav` as a warning, which would fail the build.
- **Fix:** Added an `exclude_docs:` block to `mkdocs.yml` hiding `specs/` and `superpowers/` from the site. This was preferable to moving the files out of `docs/` (which would touch unrelated paths) or adding them to nav (which would publish internal design docs). The plan did not anticipate this pre-existing content.
- **Files modified:** `mkdocs.yml`
- **Verification:** `uv run mkdocs build --strict` exits 0; only the 14 intended pages are rendered into `site/`.
- **Commit:** `d7744b3`

**[Rule 2 - Missing Critical] Add `site/` to `.gitignore`**

- **Found during:** Task 1 (after first `mkdocs build`)
- **Issue:** `mkdocs build` emits the rendered HTML site into `site/` at repo root. Without a gitignore entry, `git status` would flood with hundreds of untracked build-output files and a future `git add .` could commit them.
- **Fix:** Added `site/` entry to `.gitignore` (per task_commit_protocol's "check for untracked generated files" step).
- **Files modified:** `.gitignore`
- **Commit:** `d7744b3`

**Total deviations:** 2 auto-fixed (1 Rule 3 Blocking, 1 Rule 2 Missing Critical). **Impact:** none — both are scaffold correctness fixes that leave the plan's intent intact.

## Known Stubs

Intentional stubs — documented in plan and referenced by future plans:

| Stub | File | Reason | Resolved by |
|------|------|--------|-------------|
| "TBD — populated in Phase 32 via mkdocstrings." | `docs/reference/python-api.md` | Python API auto-gen requires mkdocstrings setup | Phase 32 |
| "TBD — populated in Phase 32 from `openapi/zeroth-core-openapi.json`." | `docs/reference/http-api.md` | HTTP API render from committed OpenAPI spec | Phase 32 |
| "TBD — populated in Phase 32 from pydantic-settings." | `docs/reference/configuration.md` | Config reference auto-gen from Settings classes | Phase 32 |
| "Content added in plan 30-03." | `docs/tutorials/getting-started/01-install.md`, `02-first-graph.md`, `03-service-and-approval.md` | Plan 30-02 scaffolds; Plan 30-03 fills tutorial body | Plan 30-03 |
| "Content added in plan 30-04." | `docs/tutorials/governance-walkthrough.md` | Plan 30-02 scaffolds; Plan 30-04 fills walkthrough body | Plan 30-04 |
| "Populated in Phase 31." | `docs/how-to/index.md`, `docs/concepts/index.md` | Diataxis quadrants not in scope for Phase 30 | Phase 31 |

None of these block SITE-01/SITE-04/DOCS-01 — all three success criteria are met by the scaffold + landing page alone. The test `test_reference_quadrant_stubs_are_minimal` enforces the Phase 32 stubs stay minimal to prevent fake-content rot.

## Threat Flags

None — no new network endpoints, auth paths, file-access patterns, or schema changes at trust boundaries. This plan is static-site scaffolding only, built from markdown and YAML.

## Ready for Next Plan

Plan 30-03 (Getting Started tutorial) can now:

1. Replace the bodies of `docs/tutorials/getting-started/01-install.md`, `02-first-graph.md`, and `03-service-and-approval.md` in place — no new files, no nav changes needed.
2. Import `zeroth.core.examples.quickstart.build_demo_graph` (shipped in Plan 30-01) for tutorial snippets.
3. Embed additional `examples/*.py` files via `--8<-- "<name>.py"` thanks to the `base_path: [".", "examples"]` configuration.
4. Rely on `uv run mkdocs build --strict` as the CI gate for docs/code drift (CI workflow wired in Plan 30-05).

## Self-Check: PASSED

- [x] `mkdocs.yml` exists on disk
- [x] `docs/index.md` exists on disk
- [x] `docs/tutorials/index.md` exists on disk
- [x] `docs/tutorials/getting-started/index.md` exists on disk
- [x] `docs/tutorials/getting-started/01-install.md` exists on disk
- [x] `docs/tutorials/getting-started/02-first-graph.md` exists on disk
- [x] `docs/tutorials/getting-started/03-service-and-approval.md` exists on disk
- [x] `docs/tutorials/governance-walkthrough.md` exists on disk
- [x] `docs/how-to/index.md` exists on disk
- [x] `docs/concepts/index.md` exists on disk
- [x] `docs/reference/index.md` exists on disk
- [x] `docs/reference/python-api.md` exists on disk
- [x] `docs/reference/http-api.md` exists on disk
- [x] `docs/reference/configuration.md` exists on disk
- [x] `pyproject.toml` has `[project.optional-dependencies].docs`
- [x] `tests/test_docs_phase30.py` contains 7 Plan 30-02 shape assertions
- [x] Commit `d7744b3` present in git log (feat(30-02): scaffold mkdocs-material site with Diataxis IA)
- [x] Commit `dcfb5ab` present in git log (test(30-02): add mkdocs config + landing page + reference stub shape assertions)
- [x] `uv run mkdocs build --strict` → exit 0, `site/sitemap.xml` lists 13 entries for `https://rrrozhd.github.io/zeroth/`
- [x] `uv run pytest tests/test_docs_phase30.py -v` → 8 passed
