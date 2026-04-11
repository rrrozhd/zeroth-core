---
phase: 32-reference-docs-deployment-migration-guide
plan: 01
subsystem: docs-reference
tags: [docs, mkdocs, mkdocstrings, python-api, reference]
requires:
  - Phase 31 Usage Guides (cross-link targets)
  - mkdocs-material docs site from Phase 30
provides:
  - Auto-generated Python API reference for all 20 zeroth.core subsystems
  - Landing page at docs/reference/python-api.md with grouped subsystem TOC
  - mkdocstrings plugin wired in mkdocs.yml
  - mkdocs.yml nav block for 20 Python API pages
affects:
  - docs/how-to/*.md (20 Usage Guides updated cross-links)
  - pyproject.toml [docs] extra
  - mkdocs.yml plugins + nav
tech-stack:
  added:
    - "mkdocstrings[python]>=0.26,<1"
    - mkdocstrings-python 2.0.3
    - griffe / griffelib 2.0.2
    - mkdocs-autorefs 1.4.4
  patterns:
    - "::: zeroth.core.<module> mkdocstrings directive per subsystem"
    - Google docstring style (matches ruff pydocstyle.convention)
key-files:
  created:
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
  modified:
    - pyproject.toml
    - uv.lock
    - mkdocs.yml
    - docs/reference/python-api.md
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
key-decisions:
  - Target zeroth.core.agent_runtime (not zeroth.core.agents) for the Agents reference page per Phase 27 rename
  - Preserve existing bullet-list cross-link tails in how-to guides that had richer "Related" sections, but lead each with the canonical "See the Python API reference for ..." line pointing at the new per-subsystem page
  - Identity how-to also calls out that zeroth.core.service.auth lives on the Service reference page (avoids a broken #service-auth anchor)
requirements-completed:
  - DOCS-07
metrics:
  duration: "~25 min"
  completed: 2026-04-11
  tasks: 3
  files_created: 20
  files_modified: 25
---

# Phase 32 Plan 01: Python API Reference via mkdocstrings Summary

Wire up mkdocstrings + Griffe so every `zeroth.core.*` subsystem renders an auto-generated API reference page from its docstrings, and replace the Phase 30 stub + Phase 31 "generated in Phase 32" placeholder cross-links with real links.

## What shipped

**Task 1 - Plugin wiring (commit `ef6eef8`).** Added `mkdocstrings[python]>=0.26,<1` to the `[docs]` extra in `pyproject.toml`, ran `uv sync --extra docs` (pulled in `mkdocstrings` 0.30.1, `mkdocstrings-python` 2.0.3, `griffelib` 2.0.2, `mkdocs-autorefs` 1.4.4), and configured the plugin in `mkdocs.yml` with Google docstring style (matching `tool.ruff.lint.pydocstyle.convention`), `paths: [src]`, `show_root_heading: true`, `show_source: false`, `members_order: source`, `show_signature_annotations: true`, `separate_signature: true`, and `merge_init_into_class: true`.

**Task 2 - 20 subsystem pages + landing (commit `6ad37cc`).** Replaced `docs/reference/python-api.md` (previously a single "TBD" line) with a landing page grouping the 20 subsystems under four categories (Execution core, Data & state, Governance, Platform). Created 20 per-subsystem pages under `docs/reference/python-api/` each containing only the canonical `::: zeroth.core.<module>` directive with `show_root_heading` and `members_order: source`. The Agents page targets `zeroth.core.agent_runtime` (Phase 27 rename). `uv run mkdocs build --strict` exits 0; inspected `site/reference/python-api/graph/index.html` and confirmed it renders real symbols (`GraphDiff`, `GraphRepository`, `GraphStatus`, 176 doc elements).

**Task 3 - Nav + how-to cross-links (commit `c876833`).** Expanded the `Reference` nav block in `mkdocs.yml` to nest all 20 Python API pages under a `Python API:` parent (landing page listed first via bare `reference/python-api.md` entry), leaving `HTTP API` and `Configuration` stubs untouched for plans 32-02/32-03. Rewrote the `## Reference cross-link` section in all 20 `docs/how-to/*.md` files to point at the real `../reference/python-api/<slug>.md` pages, dropping every `(generated in Phase 32)` parenthetical and every `python-api.md#...` anchor. Files that previously had richer "Related: ..." tails (policy, approvals, audit, guardrails, identity) keep that context, but are now led by the canonical `See the [Python API reference for ...]` line.

## Verification

- `uv run mkdocs build --strict` exits 0 (only warning is the unrelated MkDocs-v2-propaganda notice emitted by Material).
- `grep -rc "generated in Phase 32" docs/` → clean (0 matches).
- `grep -rc "reference/python-api.md#zerothcore" docs/` → clean.
- `ls docs/reference/python-api/*.md | wc -l` → 20.
- `site/reference/python-api/graph/index.html` and `site/reference/python-api/webhooks/index.html` both exist.
- Rendered graph HTML contains 176 `class="doc"` elements and includes real symbols (`GraphDiff`, `GraphRepository`, `GraphStatus`).

## Deviations from Plan

None - plan executed exactly as written. Minor wording fidelity: for the five how-to files whose pre-existing cross-link section contained a multi-bullet `Related:` list (policy, approvals, audit, guardrails, identity, plus the special-cased service-auth mention in identity), I preserved the useful "Related" context below the new canonical line instead of deleting it. This matches the plan's intent (replace the placeholder anchor, drop the "generated in Phase 32" tail) without losing curated context.

## Authentication Gates

None.

## Deferred Issues

None.

## Known Stubs

None. The Python API pages are intentionally body-less because mkdocstrings fills them at build time — this is the design, not a stub.

## Next

Ready for `32-02-http-api-reference` (Wave 1, parallel). Zero file overlap with this plan — that work touches `docs/reference/http-api.md`, `mkdocs.yml` nav for HTTP API only, and any Swagger-embed plumbing.

## Self-Check: PASSED

- All 20 `docs/reference/python-api/*.md` files exist on disk.
- `docs/reference/python-api.md` landing page exists and lists all 20 subsystems.
- `pyproject.toml` contains `mkdocstrings[python]>=0.26,<1`.
- `mkdocs.yml` plugins block contains `mkdocstrings` and nav block lists all 20 pages.
- Commits `ef6eef8`, `6ad37cc`, `c876833` all present in `git log`.
- `uv run mkdocs build --strict` exits 0; `site/reference/python-api/{graph,webhooks}/index.html` generated.
- Zero residual `generated in Phase 32` or `python-api.md#zerothcore*` strings in docs/.
