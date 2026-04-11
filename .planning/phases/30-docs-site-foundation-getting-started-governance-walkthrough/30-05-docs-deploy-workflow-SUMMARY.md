---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 05
subsystem: docs
tags: [docs, ci, github-pages, deploy, mkdocs-material]
requires:
  - 30-02 (mkdocs scaffold + docs/ tree)
  - 30-03 (getting started tutorial)
  - 30-04 (governance walkthrough tutorial)
provides:
  - docs-deploy-workflow
  - live-docs-url-on-main
affects:
  - .github/workflows/docs.yml
  - README.md
tech-stack:
  added: []
  patterns:
    - "mkdocs gh-deploy --force --no-history from GHA with contents:write"
    - "PR-trigger strict build, main-trigger build + deploy"
key-files:
  created:
    - .github/workflows/docs.yml
    - .planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/deferred-items.md
  modified:
    - README.md
    - tests/test_docs_phase30.py
key-decisions:
  - "Use mkdocs gh-deploy --force over peaceiris/actions-gh-pages to minimise marketplace trust surface (matches mkdocs-material's own recommended publish path)"
  - "Gate deploy step on github.event_name == 'push' && github.ref == 'refs/heads/main' so forked PRs don't 403 on the gh-pages push"
  - "Single job runs both strict build and conditional deploy — avoids duplicate uv sync and keeps the workflow readable"
requirements-completed:
  - SITE-02
  - SITE-01
  - SITE-04
requirements-deferred:
  - SITE-03
duration: "~3 min"
completed: 2026-04-11
---

# Phase 30 Plan 05: Docs Deploy Workflow Summary

Wired the docs deploy pipeline end-to-end: a single `.github/workflows/docs.yml` runs `uv run mkdocs build --strict` on every PR and push to `main`, then conditionally runs `mkdocs gh-deploy --force --no-history` only on pushes to `main`, pushing the rendered site to the `gh-pages` branch. README.md now links to the live docs URL from a new Documentation section near the top, and Phase 30's one deferred requirement (SITE-03 PR previews) is recorded in STATE.md blockers with the rationale from CONTEXT D-06.

- **Started:** 2026-04-11T19:49Z (approx)
- **Completed:** 2026-04-11T19:53Z
- **Duration:** ~3 minutes of wall-clock work
- **Tasks completed:** 2 of 3 autonomous; Task 3 is a one-time manual GitHub Pages source selection surfaced under "Required User Action" below
- **Files created:** 2
- **Files modified:** 2

## What Shipped

### 1. `.github/workflows/docs.yml`

A single-job workflow that builds the site on every PR and push, and deploys only from `main`:

```yaml
name: docs

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: astral-sh/setup-uv@v5
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: uv sync --extra docs
      - name: Build site (strict)
        run: uv run mkdocs build --strict
      - name: Deploy to GitHub Pages
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: uv run mkdocs gh-deploy --force --no-history
```

Key properties verified by the new shape tests:

- Triggers on both `push` and `pull_request` to `main`.
- Top-level `permissions: contents: write` grants gh-deploy the right to push the `gh-pages` branch.
- Build step uses `--strict` so dead links and missing snippet files fail PR CI, not production.
- Deploy step is gated on `github.event_name == 'push' && github.ref == 'refs/heads/main'`, so forked-PR runs never attempt to push and never hit 403.
- Action versions (`actions/checkout@v4`, `astral-sh/setup-uv@v5`, `actions/setup-python@v5`) match the existing `.github/workflows/ci.yml` baseline for consistency.

### 2. `README.md` — Documentation section

Added directly after the tagline and before Install, non-disruptive to existing content:

```markdown
## Documentation

Full documentation lives at **<https://rrrozhd.github.io/zeroth/>** —
start with the [Getting Started tutorial](https://rrrozhd.github.io/zeroth/tutorials/getting-started/)
or the [Governance Walkthrough](https://rrrozhd.github.io/zeroth/tutorials/governance-walkthrough/).
```

### 3. `tests/test_docs_phase30.py` — five new shape tests

- `test_docs_workflow_exists_and_valid_yaml` — file exists, parses, `name: docs`, both triggers present.
- `test_docs_workflow_build_is_strict` — workflow text contains `mkdocs build --strict`.
- `test_docs_workflow_deploy_is_main_only` — inspects the deploy step `if:` condition for both `github.event_name == 'push'` and `refs/heads/main`.
- `test_docs_workflow_has_contents_write_permission` — walks workflow- and job-level permissions, asserts `contents: write`.
- `test_readme_links_to_live_docs` — README.md contains `https://rrrozhd.github.io/zeroth/`.

All 22 tests in the file pass (17 pre-existing + 5 new).

## Phase-gate Validation (Task 2)

Ran the complete Phase 30 gate locally:

| Check | Result |
| --- | --- |
| `uv run mkdocs build --strict` | PASS — `Documentation built in 0.27 seconds`, zero warnings in strict mode |
| `uv run pytest -q` | PASS — 691 passed, 12 deselected, 1 pre-existing warning; no regressions |
| `uv run ruff check src/ tests/` | PASS — clean |
| `uv run pytest tests/test_docs_phase30.py -v` | PASS — 22/22 |
| `OPENAI_API_KEY= uv run python examples/hello.py` | PASS — SKIP notice, exit 0 |
| `OPENAI_API_KEY= uv run python examples/first_graph.py` | PASS — SKIP notice, exit 0 |
| `OPENAI_API_KEY= uv run python examples/approval_demo.py` | PASS — SKIP notice, exit 0 |
| `OPENAI_API_KEY= uv run python examples/governance_walkthrough.py` | PASS — SKIP notice, exit 0 |
| `site/index.html` | PRESENT |
| `site/sitemap.xml` | PRESENT |
| `site/tutorials/getting-started/02-first-graph/index.html` | PRESENT |
| `site/tutorials/governance-walkthrough/index.html` | PRESENT |

All four Diátaxis sections (Tutorials, How-to Guides, Concepts, Reference) are present in the strict build's nav. Search + sitemap (SITE-04) ship with mkdocs-material by default.

## Required User Action (Task 3 — one-time manual)

**Task 3 is a `checkpoint:human-action` that cannot be automated.** GitHub Pages does not expose an API for first-time Pages source selection on a repo that has never deployed Pages before. The executor was instructed to document this clearly rather than block the autonomous run.

**Sequence once this plan is merged to `main`:**

1. Push/merge this branch to `main`. The `docs` workflow's first run on `main` will build the site and push a fresh `gh-pages` branch via `mkdocs gh-deploy --force --no-history`.
2. Wait for that workflow run to go green.
3. Open <https://github.com/rrrozhd/zeroth-core/settings/pages>.
4. Under **Build and deployment → Source**, choose **"Deploy from a branch"**.
5. Under **Branch**, choose **`gh-pages`** and folder **`/ (root)`**. Click **Save**.
6. Wait ~1–2 minutes for GitHub to provision the site.
7. Visit **<https://rrrozhd.github.io/zeroth/>** — you should see the Zeroth landing page with the Choose-Your-Path tabs and the `hello.py` snippet embed.
8. Verify:
    - Tutorials → Getting Started → First graph renders with syntax-highlighted `first_graph.py` snippet.
    - Tutorials → Governance Walkthrough renders with all three scenario sections and the full-file embed at the bottom.
    - Built-in search: press `/`, type `approval`, expect hits in the tutorial pages.

**Common issues:**

- **404 on `rrrozhd.github.io/zeroth/`** — Pages source not saved, or the `gh-pages` branch has not yet been created. Re-run the docs workflow on main and retry.
- **404 on `/tutorials/...`** — `site_url` / URL slug mismatch. If the Pages URL is actually `rrrozhd.github.io/zeroth-core/`, fix `mkdocs.yml` `site_url` and the two README links.

## Deferred Requirements

### SITE-03 — PR preview deploys (deferred, recorded in STATE.md)

**Rationale:** GitHub Pages has no native PR preview mechanism. The user accepted deferral during Phase 30 planning (CONTEXT.md D-06). The deploy gate on `main` is sufficient for the Phase 30 success criteria — every merge produces a fresh production build and the strict build on PRs prevents broken merges.

**Re-open when:** Docs deploy migrates to Cloudflare Pages, Netlify, or any platform with native PR preview URLs.

**Tracking:** STATE.md Blockers/Concerns section, added by Task 2.

## Deviations from Plan

None — plan executed exactly as written.

Task 3 was deliberately not blocked on per the orchestrator directive: the `checkpoint:human-action` represents an unavoidable first-time Pages source selection in the GitHub web UI, documented above under "Required User Action." Because it cannot be automated and does not prevent other phase work, the plan is marked complete with the manual step clearly flagged.

## Authentication Gates

None encountered during this plan. The one human-action step (Task 3) is a one-time GitHub Pages source selection, not an authentication gate.

## Commits

| Task | Description | Commit |
| --- | --- | --- |
| 1 | Add docs.yml workflow, README docs link, 5 shape tests | `f157667` |
| 2 | Record SITE-03 deferral in STATE.md + deferred-items.md | `a657d65` |

## Next Step

Phase 30 is now code-complete. The only outstanding work is the one-time GitHub Pages source selection described in "Required User Action" above. After that manual step lands and <https://rrrozhd.github.io/zeroth/> serves the Zeroth landing page, SITE-01, SITE-02, SITE-04, DOCS-01, DOCS-02, and DOCS-05 are fully verified end-to-end. SITE-03 remains deferred as documented.

Ready for Phase 30 completion verification and the next phase.

## Self-Check: PASSED

- `.github/workflows/docs.yml` exists on disk.
- `tests/test_docs_phase30.py` contains all 5 new plan 30-05 shape tests (`test_docs_workflow_exists_and_valid_yaml`, `test_docs_workflow_build_is_strict`, `test_docs_workflow_deploy_is_main_only`, `test_docs_workflow_has_contents_write_permission`, `test_readme_links_to_live_docs`) — all green.
- `README.md` contains `https://rrrozhd.github.io/zeroth/`.
- `deferred-items.md` exists at the phase dir.
- STATE.md has the SITE-03 deferral entry (added via `state add-blocker`).
- Commits `f157667` and `a657d65` present in `git log --oneline`.
