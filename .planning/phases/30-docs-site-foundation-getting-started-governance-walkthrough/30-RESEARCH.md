# Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough — Research

**Researched:** 2026-04-11
**Domain:** Static site generators (mkdocs-material), GitHub Pages deploy, developer-tutorial authoring for a governed multi-agent runtime
**Confidence:** HIGH on infra, MEDIUM on tutorial API surface (constrained by what the runtime actually exposes today)

## Summary

Phase 30 ships three things at once: (1) the mkdocs-material site scaffolding and CI deploy to `rrrozhd.github.io/zeroth`, (2) a 3-section Getting Started tutorial, and (3) a Governance Walkthrough that exercises the approval gate, auditor, and policy block. The site infra is a well-worn path — mkdocs-material's own docs recommend `mkdocs gh-deploy --force` with `permissions: contents: write` and nothing else. The non-obvious work is the tutorial content.

**Key reality check on the tutorial code:** zeroth-core today has **no ~30-line public graph builder API**. `bootstrap_service()` is 432 LOC wiring up a GraphRepository, ContractRegistry, DeploymentService, RunRepository, ThreadRepository, AuditRepository, ApprovalService, RuntimeOrchestrator, AgentRunners, ExecutableUnitRunner, authenticator, LeaseManager, DeadLetterManager, rate limiter, quota enforcer, metrics collector, and optional durable worker. The public way to execute a graph is: **persist a Graph → create a Deployment → POST `/runs` → poll `GET /runs/{run_id}`**. `examples/hello.py` from Phase 28 deliberately sidesteps all of this by calling `litellm.completion()` directly and comments that "the full orchestrator/graph builder requires service bootstrap that does not belong in a 30-line example. Phase 30 will replace this with a proper graph walkthrough."

This constrains Getting Started's section 2 ("first graph with one agent/tool/LLM"). We have two realistic options, and the planner must pick:

- **Option A — HTTP-first tutorial.** Ship a small `examples/first_graph.py` that starts a test database, deploys a pre-built Graph fixture, boots the FastAPI app in-process, POSTs a run, polls to completion, and prints the terminal output. ~60–80 lines. Demonstrates the real API surface. Section 3 then reuses the same deployment under `uvicorn` with an approval node and a CLI `curl` approve command. **This is what the requirements actually ask for** and it's internally consistent.
- **Option B — Library-first tutorial that fakes a graph.** Keep using `litellm.completion()` directly as section 2 (like hello.py today), describe it as "the LLM call your Agent node would make," then introduce the graph/approval machinery only in section 3. Cheaper but doesn't actually show a graph.

**Primary recommendation:** **Option A**, with a runnable, CI-tested `examples/first_graph.py` + `examples/service_mode_with_approval.py` + `examples/governance_walkthrough.py`, all embedded into docs via `pymdownx.snippets`. The <5-minute-to-first-output target is met by the install-and-hello step, not by the graph example. OpenAI via litellm, with a SKIP-on-missing-key guard matching `examples/hello.py`.

**Site infra primary recommendation:** mkdocs-material 9.7.6, `mkdocs gh-deploy --force` from a single workflow triggered on push to `main`, `permissions: contents: write`, pin all docs deps in a new `[docs]` extra in `pyproject.toml`, gated by a PR-time `mkdocs build --strict` that catches dead links and missing snippet files without deploying.

## User Constraints (from CONTEXT.md)

### Locked Decisions (D-01..D-14)

- **D-01 Hosting:** GitHub Pages at `https://rrrozhd.github.io/zeroth/`. Custom domain deferred.
- **D-02 Domain:** default URL only; no DNS this phase.
- **D-03 Source location:** `docs/` at repo root; `mkdocs.yml` at repo root.
- **D-04 Static site generator:** `mkdocs-material` (latest stable) with Diátaxis IA.
- **D-05 IA:** Four top-level sections `Tutorials/`, `How-to Guides/`, `Concepts/`, `Reference/` — exact canonical Diátaxis names.
- **D-06 CI/CD:** GitHub Actions workflow `.github/workflows/docs.yml`. On push to `main`: build + deploy. On PR: build only (no deploy). PR previews deferred.
- **D-07 Landing page:** 10-line hello-world snippet + install snippet + "Choose your path" (Embed as library vs Run as service) split linking to Getting Started sections.
- **D-08 Getting Started:** single linear 3-section tutorial (Install → First graph → Service mode + approval), first working output <5 min, full tutorial <30 min.
- **D-09 Example LLM provider:** OpenAI via litellm fallback pattern from `examples/hello.py`; tutorial notes any litellm-supported provider works.
- **D-10 Approval gate UX:** CLI-driven (curl or `zeroth approve <run_id>`). No Studio dependency.
- **D-11 Governance Walkthrough:** single example exercises approval gate + auditor + policy block.
- **D-12 Reference quadrant:** stub each subsection with one-line "TBD — Phase 32" note.
- **D-13 Examples are CI-tested `.py` files** in `examples/`, embedded via `pymdownx.snippets`.
- **D-14 Plugin set (minimum):** `mkdocs-material`, `pymdownx.snippets`, `pymdownx.superfences`, `pymdownx.tabbed`, `mkdocs-section-index`.

### Claude's Discretion

- mkdocs.yml palette/font/social links
- Logo/favicon (defer if no asset)
- Approval CLI command name (`zeroth approve` vs `zeroth-cli approve`) — **no `[project.scripts]` entry exists today**, so if a CLI is introduced this phase it's a net-new addition
- "What's new" / changelog page inclusion
- Versioned docs via `mike` — defer

### Deferred Ideas (OUT OF SCOPE)

Custom domain, PR preview deploys, `mike` versioned docs, social cards, subsystem concept pages (Phase 31), cookbook (Phase 31), auto-generated Python API reference (Phase 32), auto-rendered HTTP API reference (Phase 32), auto-generated configuration reference (Phase 32), migration guide (Phase 32), logo/favicon/brand assets.

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SITE-01 | mkdocs-material with explicit Diátaxis IA (Tutorials / How-to Guides / Concepts / Reference) | mkdocs.yml `nav:` layout below; `mkdocs-section-index` makes section landings clean |
| SITE-02 | GH Actions builds and deploys on every commit to `main` | `mkdocs gh-deploy --force` workflow pattern from mkdocs-material's own docs |
| SITE-03 | PR previews of changed docs | **CONFLICT with D-06: CONTEXT.md defers this.** GitHub Pages has no native PR preview. See "Open Questions". |
| SITE-04 | Built-in search + sitemap | mkdocs-material ships both for free; `site_url` must be set for canonical/sitemap |
| DOCS-01 | Landing page: 10-line hello-world + install + Choose-your-path split | `pymdownx.tabbed` for the two paths; snippet embed of `examples/hello.py` |
| DOCS-02 | 3-section Getting Started (<5 min first output, <30 min total) | Install/verify = hello.py; Section 2 = `examples/first_graph.py`; Section 3 = `examples/service_mode_with_approval.py`. See Architecture Patterns. |
| DOCS-05 | Governance Walkthrough: approval gate + auditor + policy block | `examples/governance_walkthrough.py` exercising `HumanApprovalNode`, `GET /deployments/{ref}/audits`, `PolicyDefinition.denied_capabilities` |

> **REQ vs CONTEXT mismatch:** REQUIREMENTS.md SITE-03 asks for PR previews; CONTEXT.md D-06 explicitly defers them ("PR preview deploys via GitHub Pages are not natively supported; deferred. Deploy gate on `main` is sufficient."). **The planner must carry this as a known gap and mark SITE-03 Deferred in the traceability matrix for this phase**, or spin up a follow-up to host previews on Cloudflare Pages/Netlify. Flagging for the user to confirm.

## Project Constraints (from CLAUDE.md)

- **Use `uv`** for Python tooling (`uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`). All new docs-related commands in CI and docs should use `uv run`.
- **Source layout:** `src/zeroth/` (PEP 420 namespace — no top-level `__init__.py`).
- **progress-logger skill is mandatory** during implementation sessions.
- **PROGRESS.md** is the single source of truth for implementation session state.
- Conventional Commits (from Phase 28 history).

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `mkdocs` | `>=1.6,<2.0` | Site generator engine | [CITED: squidfunk.github.io/mkdocs-material] Hard dep of mkdocs-material 9.x. |
| `mkdocs-material` | `>=9.7.6,<10` | Theme + extension suite | [VERIFIED: web search 2026-04-11] 9.7.6 is the current release; note this is the last version before the project rebrands as Zensical. |
| `pymdown-extensions` | `>=10.8` | Ships `pymdownx.snippets`, `superfences`, `tabbed`, etc. | [CITED: squidfunk.github.io/mkdocs-material/setup/extensions/python-markdown-extensions] Required by D-14. |
| `mkdocs-section-index` | `>=0.3.9` | Clean section landing pages | [CITED] Required by D-14. Lets a section's `index.md` act as the section's own landing. |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `mkdocs-material[imaging]` | — | Social-cards extra | [ASSUMED] Defer per CONTEXT; heavy (cairo/libpango). |
| `mike` | — | Versioned docs | Deferred to post-1.0. |
| `mkdocs-include-markdown-plugin` | — | Alternative snippet embedding | `pymdownx.snippets` is the chosen path (D-14); skip. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `mkdocs gh-deploy --force` | `peaceiris/actions-gh-pages@v4` + `mkdocs build` | peaceiris is a marketplace action (extra trust surface). mkdocs-material's own docs recommend `gh-deploy`. [CITED: squidfunk.github.io/mkdocs-material/publishing-your-site/] |
| mkdocs-material | Sphinx + Furo, Docusaurus, Zola | mkdocs-material dominates Python-library docs in 2026, ships search + Diátaxis-friendly nav, integrates with mkdocstrings (Phase 32). Locked by D-04. |
| HTTP approve via curl | `zeroth approve <run_id>` CLI | A CLI would need a new `[project.scripts]` entry; today zeroth-core has none. Curl is zero-install for readers and works across platforms. **Recommend curl as the default shown path**, mention a CLI only if it's already planned for Phase 31+. |

### Installation

Add a new `[project.optional-dependencies]` extra to `pyproject.toml`:

```toml
[project.optional-dependencies]
docs = [
    "mkdocs>=1.6,<2.0",
    "mkdocs-material>=9.7.6,<10",
    "pymdown-extensions>=10.8",
    "mkdocs-section-index>=0.3.9",
]
```

Dev install: `uv sync --extra docs` → local preview via `uv run mkdocs serve` on `http://127.0.0.1:8000`.

**Version verification note:** 9.7.6 confirmed via WebSearch 2026-04-11 against PyPI's own project page (see Sources). The planner should rerun `uv pip index versions mkdocs-material` immediately before pinning — the project may publish a patch release between research and execution.

## Architecture Patterns

### Recommended Project Structure

```
zeroth/                           (repo root)
├── mkdocs.yml                    (NEW — site config)
├── docs/                         (NEW — markdown tree)
│   ├── index.md                  (landing: 10-line hello + install + Choose Your Path)
│   ├── tutorials/
│   │   ├── index.md              (section landing; mkdocs-section-index)
│   │   ├── getting-started/
│   │   │   ├── index.md          (overview + time budget)
│   │   │   ├── 01-install.md
│   │   │   ├── 02-first-graph.md
│   │   │   └── 03-service-and-approval.md
│   │   └── governance-walkthrough.md
│   ├── how-to/
│   │   └── index.md              (placeholder: "Phase 31")
│   ├── concepts/
│   │   └── index.md              (placeholder: "Phase 31")
│   └── reference/
│       ├── index.md              (landing)
│       ├── python-api.md         (stub: "TBD — Phase 32")
│       ├── http-api.md           (stub: "TBD — Phase 32")
│       └── configuration.md      (stub: "TBD — Phase 32")
├── examples/                     (EXISTS from Phase 28)
│   ├── hello.py                  (EXISTS — keep, embedded in index.md and 01-install.md)
│   ├── first_graph.py            (NEW — for 02-first-graph.md)
│   ├── service_mode_with_approval.py   (NEW — for 03-service-and-approval.md)
│   └── governance_walkthrough.py (NEW — for governance-walkthrough.md)
├── .github/workflows/
│   ├── docs.yml                  (NEW — build on PR, build+deploy on main)
│   └── examples.yml              (NEW or extend verify-extras.yml — CI-test examples/)
└── pyproject.toml                (add [docs] extra)
```

### Pattern 1: mkdocs.yml minimum viable config

```yaml
# Source: synthesized from squidfunk.github.io/mkdocs-material/setup + publishing-your-site
site_name: Zeroth
site_url: https://rrrozhd.github.io/zeroth/
site_description: Governed medium-code platform for production-grade multi-agent systems
repo_url: https://github.com/rrrozhd/zeroth-core
repo_name: rrrozhd/zeroth-core
edit_uri: edit/main/docs/

theme:
  name: material
  features:
    - navigation.sections
    - navigation.tabs
    - navigation.top
    - content.code.copy
    - content.code.annotate
    - content.action.edit
    - search.suggest
    - search.highlight
  palette:
    - scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    - scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-4
        name: Switch to light mode

plugins:
  - search
  - section-index

markdown_extensions:
  - admonition
  - attr_list
  - md_in_html
  - toc:
      permalink: true
  - pymdownx.highlight:
      anchor_linenums: true
      line_spans: __span
      pygments_lang_class: true
  - pymdownx.inlinehilite
  - pymdownx.snippets:
      base_path: [".", "examples"]
      check_paths: true
  - pymdownx.superfences
  - pymdownx.tabbed:
      alternate_style: true

nav:
  - Home: index.md
  - Tutorials:
    - tutorials/index.md
    - Getting Started:
      - tutorials/getting-started/index.md
      - Install: tutorials/getting-started/01-install.md
      - First graph: tutorials/getting-started/02-first-graph.md
      - Service mode & approval: tutorials/getting-started/03-service-and-approval.md
    - Governance Walkthrough: tutorials/governance-walkthrough.md
  - How-to Guides:
    - how-to/index.md
  - Concepts:
    - concepts/index.md
  - Reference:
    - reference/index.md
    - Python API: reference/python-api.md
    - HTTP API: reference/http-api.md
    - Configuration: reference/configuration.md
```

**Why `check_paths: true`:** `pymdownx.snippets` silently inserts nothing if the referenced file is missing. `check_paths: true` makes the build fail fast — critical because the PR-time build is the only gate on docs↔code drift.

### Pattern 2: GitHub Actions workflow (docs.yml)

```yaml
# Source: https://squidfunk.github.io/mkdocs-material/publishing-your-site/
name: docs
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: write  # gh-deploy pushes to gh-pages branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # required by git-revision-date plugins (future-proof)
      - uses: astral-sh/setup-uv@v5
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: uv sync --extra docs
      - name: Build site (strict — fails on warnings, dead links, missing snippets)
        run: uv run mkdocs build --strict
      - name: Deploy to GitHub Pages
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: uv run mkdocs gh-deploy --force --no-history
```

**Key points:**
- `--strict` on every run catches warnings (dead links, unknown references, missing snippets) before merge.
- `gh-deploy` only runs on direct push to `main` — PRs only build.
- `contents: write` is the only permission needed. `pages: write` / `id-token: write` are only required for the `actions/deploy-pages` flow (the "official" GitHub Pages deploy action), which is an alternative path and **not** what mkdocs-material's own docs recommend.
- One-time manual setup: in the repo's Settings → Pages, set source to the `gh-pages` branch. Document this in the plan as a user action.

### Pattern 3: Embedding runnable Python into markdown

```markdown
<!-- in docs/tutorials/getting-started/01-install.md -->

Verify your install by running the packaged hello example:

```python title="examples/hello.py"
--8<-- "examples/hello.py"
```
```

The `--8<--` token is `pymdownx.snippets`' scissors notation. With `base_path: [".", "examples"]` configured in `mkdocs.yml`, both `"examples/hello.py"` and (if we add `[start:func]`/`[end:func]` markers inside the .py) partial embeds work. For Phase 30 we embed whole files, not ranges — simpler, less error-prone.

### Pattern 4: How the Getting Started code examples should look

The planner will write these. Sketched here so the plan knows the shape.

**`examples/first_graph.py`** (Section 2 of Getting Started)

The honest version builds a `Graph` Pydantic model with one `AgentNode`, persists it via `GraphRepository`, creates a `Deployment` via `DeploymentService`, bootstraps the service in-process, POSTs to `/runs`, polls, and prints the terminal output. Realistically ~80–120 lines, not 30. This is the cost of the actual governance machinery — and is fine for tutorial section 2, which explicitly says <30 min total, not <5 min for this specific step.

Alternative: ship a hand-crafted `zeroth.core.examples.quickstart.build_minimal_graph(...)` helper in `src/zeroth/core/` as part of this phase so the tutorial can say `from zeroth.core.examples.quickstart import run_first_graph; run_first_graph()` in ~10 user-facing lines. **This is the cleanest outcome but adds scope.** The planner should explicitly call this out as a plan-level decision.

**`examples/service_mode_with_approval.py`** (Section 3)

Two-process flow:
1. Terminal A: run `uv run python -m zeroth.core.service.entrypoint` against a SQLite DB pre-seeded with a graph containing a `HumanApprovalNode`.
2. Terminal B: POST `/runs`, observe `RunPublicStatus.PAUSED_FOR_APPROVAL`, GET `/deployments/{ref}/approvals` to find the `approval_id`, POST `/deployments/{ref}/approvals/{approval_id}/resolve` with `{"decision": "approve"}`, then GET `/runs/{run_id}` to see `succeeded`.

The example file itself should automate Terminal B and print curl equivalents alongside so the tutorial can show both.

**`examples/governance_walkthrough.py`** (Governance Walkthrough tutorial)

Single script runs three scenarios against one deployed graph:
1. **Approval gate:** submit a run that hits a `HumanApprovalNode`, show the pause, approve, show completion.
2. **Auditor review:** after the run completes, `GET /runs/{run_id}/timeline` to fetch the ordered `NodeAuditRecord` list, print each entry's node_id/status/policy decisions.
3. **Policy block:** deploy a variant of the same graph with a `PolicyDefinition` that has `denied_capabilities: [NETWORK_WRITE]` bound to the tool node, submit a run that triggers the denied capability, show the run terminating with `RunPublicStatus.TERMINATED_BY_POLICY`, then `GET /deployments/{ref}/audits` to show the denial record.

### Pattern 5: LLM provider gating (from hello.py)

```python
# Source: examples/hello.py (Phase 28)
import os, sys
if not os.environ.get("OPENAI_API_KEY"):
    print("SKIP: set OPENAI_API_KEY to run this example", file=sys.stderr)
    return 0
```

Every new example file in Phase 30 must follow this pattern so PR CI from forks (no secrets) exits 0 rather than red. The main-branch CI run can be configured with the org's OpenAI key to actually exercise the LLM path.

### Anti-Patterns to Avoid

- **Hand-rolled HTML landing page.** mkdocs-material ships a `home.html` override pattern; for Phase 30 a plain `index.md` with `pymdownx.tabbed` for Choose-Your-Path is sufficient and won't rot.
- **Inlined code blocks duplicated from `examples/`.** Defeats the whole point of `pymdownx.snippets`. Every non-trivial code block in the tutorial must be a snippet embed.
- **"SITE-03 PR preview via `peaceiris/actions-gh-pages` under a separate branch."** Technically possible but collides with GH Pages' "single source branch" model and leaves stale preview artifacts. Defer to Cloudflare Pages or Netlify in a follow-up phase.
- **Writing `docs/reference/*.md` with fake content.** D-12 says: stub with one-line TBD note. Fake content becomes ground truth and rots.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Documentation site scaffold | Custom Jinja templates | mkdocs-material theme | 10k+ hours of a11y, i18n, search, dark mode, mobile. |
| Code-block syntax highlighting | Custom pygments config | `pymdownx.highlight` + `pymdownx.superfences` | Handles line numbers, annotations, copy buttons, diff marks. |
| Embedding source files into docs | Bash script that generates .md | `pymdownx.snippets` | Works at build time, fails loudly on missing files with `check_paths: true`. |
| GitHub Pages deploy | Custom push-to-gh-pages shell script | `mkdocs gh-deploy --force` | Calls `ghp-import` under the hood; handles orphan branch semantics. |
| Tabbed "Choose your path" UI | Custom CSS/JS | `pymdownx.tabbed` with `alternate_style: true` | a11y-correct, prints correctly, keyboard-navigable. |
| Site search | Algolia / custom JS | Built-in `search` plugin (lunr.js) | Free, offline, ships with mkdocs-material. Algolia is a Phase-31+ option if traffic demands. |
| Sitemap / canonical URLs | Manual sitemap.xml | mkdocs-material auto-generates from `site_url` | SEO-correct out of the box. |

**Key insight:** Every component of the site infra for this phase is commodity. The scope risk is entirely in the tutorial code examples, not in the site config.

## Runtime State Inventory

Phase 30 is a greenfield docs phase. **Nothing to rename, migrate, or update in running systems.** No section applies.

- **Stored data:** None.
- **Live service config:** None — new workflow file, new repo pages settings.
- **OS-registered state:** None.
- **Secrets/env vars:** New optional `OPENAI_API_KEY` repo secret for main-branch `examples.yml` CI; not required for `docs.yml`.
- **Build artifacts:** The `gh-pages` branch is created fresh by `mkdocs gh-deploy` on first run.

## Common Pitfalls

### Pitfall 1: `pymdownx.snippets` silently dropping missing files
**What goes wrong:** A rename in `examples/` leaves the docs referencing a dead snippet. Build succeeds; page shows a blank code block.
**Why it happens:** Default `check_paths: false`.
**How to avoid:** Set `check_paths: true` and `base_path: [".", "examples"]`. The `--strict` flag on `mkdocs build` turns any warning into an error.
**Warning signs:** Empty code fences in preview.

### Pitfall 2: `mkdocs gh-deploy` on a fork PR
**What goes wrong:** Workflow tries to push to `gh-pages`, gets 403 because forked PRs can't write to the base repo.
**Why it happens:** `contents: write` is scoped to the base repo's token, which forked PRs don't get.
**How to avoid:** Guard the deploy step with `if: github.event_name == 'push' && github.ref == 'refs/heads/main'`. PR runs only build, don't deploy. (This is what the workflow snippet above does.)
**Warning signs:** Red CI on any fork PR.

### Pitfall 3: `site_url` unset → broken canonical URLs and sitemap
**What goes wrong:** Search engines see every page as its own root; the XML sitemap is half-broken.
**Why it happens:** mkdocs treats `site_url` as optional but mkdocs-material's SEO features depend on it.
**How to avoid:** Always set `site_url: https://rrrozhd.github.io/zeroth/` (trailing slash matters).

### Pitfall 4: Example Python files drift from the tutorial narrative
**What goes wrong:** A contributor edits `examples/first_graph.py` but not the surrounding prose in `02-first-graph.md`; the prose describes a function that no longer exists.
**Why it happens:** Snippet embedding prevents *code* drift but not *narrative* drift.
**How to avoid:** Split examples into small numbered files (`examples/getting_started/01_*.py`, `02_*.py`) or use `[start:label]`/`[end:label]` range markers so each prose paragraph embeds exactly the chunk it describes. Lean on the CI `examples.yml` job to re-execute each file on every PR.

### Pitfall 5: First-time `gh-pages` branch creation requires a human step
**What goes wrong:** First deploy runs, creates `gh-pages`, but the site 404s because Settings → Pages still has source = `None`.
**Why it happens:** GH doesn't auto-enable Pages on branch creation.
**How to avoid:** Plan must include a USER ACTION task: "After the first green `docs.yml` run on `main`, go to Settings → Pages, set source = Deploy from a branch → `gh-pages` → `/ (root)`, save."
**Warning signs:** Green CI, dead public URL.

### Pitfall 6: OpenAI example running in CI on every PR
**What goes wrong:** Every PR burns ~$0.001 of OpenAI credits, forks' PRs fail with 401 because secrets aren't exposed.
**Why it happens:** `examples.yml` doesn't guard on `secrets.OPENAI_API_KEY`.
**How to avoid:** Match `examples/hello.py`'s existing pattern — each example file checks `os.environ.get("OPENAI_API_KEY")` at top and prints `SKIP:` and exits 0 if missing. Don't gate the job itself on the secret; gate the LLM call inside the script.

### Pitfall 7: SITE-03 requirement vs D-06 decision mismatch
**What goes wrong:** Phase closes with SITE-03 unmet; traceability matrix shows a red box.
**Why it happens:** The requirement predates the decision to defer PR previews.
**How to avoid:** Planner explicitly marks SITE-03 as **Deferred to follow-up phase** in PLAN.md and the traceability table, with rationale "GH Pages has no native PR preview; requires Cloudflare Pages or Netlify alternative." Get user confirmation during plan review.

## Code Examples

Everything in this section is verified against files in the repo or against mkdocs-material official docs. The tutorial code snippets live in the "Architecture Patterns" section above; this section documents the runtime API surface the examples will call.

### Resolving an approval (HTTP, from `src/zeroth/core/service/approval_api.py`)

```
GET  /deployments/{deployment_ref}/approvals?run_id={run_id}
GET  /deployments/{deployment_ref}/approvals/{approval_id}
POST /deployments/{deployment_ref}/approvals/{approval_id}/resolve
     body: {"decision": "approve"|"reject", "edited_payload": {...}|null}
```

The tutorial's curl snippet:
```bash
# From examples/service_mode_with_approval.py narrative
curl -X POST http://localhost:8000/deployments/default/approvals/$APPROVAL_ID/resolve \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"decision": "approve"}'
```

### Creating a run (HTTP, from `src/zeroth/core/service/run_api.py`)

```
POST /runs  (scope-bound to deployment via URL path in other endpoints, body-free deployment for /runs)
     body: {"input_payload": {...}, "thread_id": "optional"}
     → 202 Accepted with RunStatusResponse
GET  /runs/{run_id}
     → RunStatusResponse with status in {queued, running, paused_for_approval, succeeded, failed, terminated_by_policy, ...}
```

### Querying the audit trail (HTTP, from `src/zeroth/core/service/audit_api.py`)

```
GET /runs/{run_id}/timeline
    → AuditTimelineResponse { deployment_ref, run_id, entries: [NodeAuditRecord] }
GET /deployments/{deployment_ref}/audits?run_id=...&node_id=...
    → AuditRecordListResponse
GET /deployments/{deployment_ref}/timeline
    → AuditTimelineResponse (whole deployment)
```

### Policy denial shape (from `src/zeroth/core/policy/models.py`)

```python
from zeroth.core.policy import PolicyDefinition, Capability

block_network = PolicyDefinition(
    policy_id="block-network-write",
    denied_capabilities=[Capability.NETWORK_WRITE],
)
# Bind via NodeBase.policy_bindings = ["block-network-write"]
```

When the orchestrator evaluates a node whose capability bindings include a denied capability, `RuntimeOrchestrator` terminates the run with `RunStatus.TERMINATED_BY_POLICY`, and the denial is recorded as a `NodeAuditRecord` visible via the audit endpoints above.

### Booting the service (from `src/zeroth/core/service/entrypoint.py`)

```bash
# Minimal — starts uvicorn on :8000 via the app_factory
uv run python -m zeroth.core.service.entrypoint
```

Environment:
- `ZEROTH_DEPLOYMENT_REF` — which deployment to bind (default `"default"`)
- `PORT` — uvicorn port
- `ZEROTH_DATABASE__BACKEND=sqlite` (default) or `postgres`
- `OPENAI_API_KEY` — for the Getting Started agent to actually call an LLM

**No console script entry exists in `pyproject.toml` today.** If the plan wants to shorten this to `zeroth-core serve`, it must add a `[project.scripts]` entry — a deliberate scope addition to flag.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Sphinx + Furo for Python-library docs | mkdocs-material + mkdocstrings | ~2022 onwards | Lower config burden, nicer default theme, Markdown-first. |
| `actions/deploy-pages` artifact flow | `mkdocs gh-deploy --force` to `gh-pages` branch | Both valid; mkdocs-material's own docs recommend `gh-deploy` | Simpler permissions (`contents: write` only), no `id-token: write` gymnastics. |
| Manual OpenAI SDK calls in examples | `litellm.completion()` | Phase 28 adopted this for `hello.py` | One example works across OpenAI, Anthropic, Bedrock, Vertex, etc. |
| Hand-written API reference | `mkdocstrings + Griffe` | Phase 32 will adopt | Not this phase. |

**Deprecated/outdated:**
- `mkdocs-material` itself is approaching its last feature release (per 2026-02-18 blog post: team is moving to Zensical). 9.7.6 remains on long-term maintenance. No action needed this phase — it's stable and dominant — but the `valid until` estimate on this research is shorter than normal (see Metadata).

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `mkdocs-material 9.7.6` is the correct version to pin | Standard Stack | Low — even if a patch lands, semver-minor bump won't break; `uv pip index versions mkdocs-material` before commit resolves it. |
| A2 | `pymdown-extensions >=10.8` is the floor for all needed extensions | Standard Stack | Low — could pin higher if a snippets bug is found; not blocking. |
| A3 | `mkdocs-section-index >=0.3.9` is stable | Standard Stack | Low. |
| A4 | mkdocs-material docs still recommend `mkdocs gh-deploy --force` in 2026 | Architecture Pattern 2 | LOW — confirmed via WebFetch of the publishing-your-site page. |
| A5 | Creating a minimal graph in ~30 lines of *user-facing* code is not currently possible without a new helper module | Summary | MEDIUM — the only way to shrink this is to introduce `zeroth.core.examples.quickstart` in this phase, which is a scope addition the planner must weigh. |
| A6 | The approval API is callable with just a Bearer token (no mTLS / no cookie session) | Code Examples | LOW — confirmed against `authorization.py` import pattern in approval_api.py. |
| A7 | SQLite default backend is viable for the tutorial (no Postgres required) | Pattern 4 | LOW — `pyproject.toml` has `aiosqlite>=0.22` as a base dep and `entrypoint.py` treats postgres as the opt-in branch. |
| A8 | "PR preview deploys" in SITE-03 can be marked Deferred with user consent | Phase Requirements table | MEDIUM — needs user confirmation at plan review time. If the user insists, the phase scope expands to a Cloudflare Pages or Netlify path. |
| A9 | No `[project.scripts]` CLI entry exists today | Claude's Discretion | LOW — verified via grep. |
| A10 | GitHub Pages for the `rrrozhd/zeroth` repo still maps to `rrrozhd.github.io/zeroth/` (i.e. the repo is named `zeroth`, not `zeroth-core`) | Standard Stack / Pattern 1 site_url | MEDIUM — CONTEXT.md D-01 says `rrrozhd.github.io/zeroth/`. `pyproject.toml` Homepage is `github.com/rrrozhd/zeroth-core`. Repo name discrepancy (`zeroth` vs `zeroth-core`) means `site_url` may actually be `rrrozhd.github.io/zeroth-core/`. **Planner must verify the actual GitHub repo slug before setting `site_url`.** |

**Flagged for user confirmation before locking:** A5 (scope of a quickstart helper module), A8 (SITE-03 deferral), A10 (canonical site_url).

## Open Questions

1. **Repo slug for `site_url`** (A10)
   - What we know: CONTEXT.md D-01 says `rrrozhd.github.io/zeroth/`. pyproject says `github.com/rrrozhd/zeroth-core`.
   - What's unclear: Whether the GH repo is `zeroth` or `zeroth-core`.
   - Recommendation: Planner runs `gh repo view rrrozhd/zeroth-core --json url,name` or asks the user; sets `site_url` to the matching GH Pages URL. Do not guess.

2. **Should Phase 30 ship `zeroth.core.examples.quickstart` helper?** (A5)
   - What we know: A 10-line user-facing graph example requires a helper. Without it, the tutorial example is ~80–120 lines.
   - What's unclear: Whether the user values tutorial brevity over additional scope in this phase.
   - Recommendation: Ship a thin helper (~50 LOC, `build_minimal_graph(agent_instruction, llm_model) -> (Graph, Deployment)` + `run_example_graph(...)`) so section 2 can be 10-line user-facing. Raise as a plan-level decision.

3. **SITE-03 PR previews: deferred or implemented via third-party?** (A8, Pitfall #7)
   - What we know: REQUIREMENTS demands it; CONTEXT defers it; GH Pages can't do it natively.
   - What's unclear: User's tolerance for deferral vs. scope expansion.
   - Recommendation: Default to deferral (honor CONTEXT.md D-06) and raise as explicit plan-review question. If the user insists on previews, open a follow-up phase that migrates docs deploy to Cloudflare Pages or Netlify.

4. **Approval CLI — this phase or later?**
   - What we know: No `[project.scripts]` entry exists. Curl works everywhere.
   - What's unclear: Whether to add a `zeroth approve` CLI now.
   - Recommendation: Curl only in Phase 30. Mention in the tutorial: "A `zeroth` CLI is planned; until then, approve via curl." Simpler, defer the CLI to a future phase.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `uv` | All docs builds and example runs | ✓ | 0.11.3 | — |
| `python3` (3.12+) | zeroth-core base, mkdocs, examples | ✓ (3.9 system; `uv` manages 3.12) | uv-managed | — |
| `gh` CLI | Optional for plan-time repo slug check | ✓ | present | — |
| `curl` | Tutorial approval flow narrative | ✓ | system | — |
| `mkdocs` / `mkdocs-material` | Site build | ✗ (not yet; added this phase via `[docs]` extra) | — | `uv sync --extra docs` installs |
| `OPENAI_API_KEY` | Running examples end-to-end in CI | ✗ locally | — | SKIP pattern from `examples/hello.py` — no fallback required for PR CI |
| GitHub Pages enablement on repo | Deploy | ? (assumed not yet enabled) | — | USER ACTION to enable after first deploy |

**Missing dependencies with no fallback:** None blocking this phase — all are normal setup steps.

**Missing dependencies with fallback:** `OPENAI_API_KEY` on forks → SKIP pattern.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `pytest` 8.x (`pytest-asyncio`, asyncio_mode=auto) |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `uv run pytest tests/test_docs_phase30.py -x -q` (new test file to add) |
| Full suite command | `uv run pytest -v --no-header -ra` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SITE-01 | `mkdocs build --strict` succeeds; nav has 4 top-level Diátaxis sections | integration | `uv run mkdocs build --strict && uv run python -c "import yaml; cfg=yaml.safe_load(open('mkdocs.yml')); top={list(x.keys())[0] if isinstance(x,dict) else x for x in cfg['nav']}; assert {'Tutorials','How-to Guides','Concepts','Reference'}<=top"` | ❌ Wave 0 (new test_docs_build.py) |
| SITE-02 | `.github/workflows/docs.yml` exists, builds on PR, deploys on main | unit | `uv run pytest tests/test_docs_phase30.py::test_docs_workflow_shape -x` | ❌ Wave 0 |
| SITE-03 | (Deferred — see Open Question 3) | — | — | — |
| SITE-04 | Built-in search plugin present; site_url set | unit | `uv run pytest tests/test_docs_phase30.py::test_mkdocs_config -x` | ❌ Wave 0 |
| DOCS-01 | `docs/index.md` embeds `examples/hello.py`, has install snippet, has tabbed split | unit | `uv run pytest tests/test_docs_phase30.py::test_landing_page_shape -x` | ❌ Wave 0 |
| DOCS-02 | 3 numbered tutorial pages exist; each embeds a runnable example; examples exit 0 when env keys present and SKIP cleanly otherwise | integration | `uv run pytest tests/test_docs_phase30.py::test_getting_started_structure -x` + `uv run python examples/first_graph.py` (SKIP path) | ❌ Wave 0 |
| DOCS-05 | Governance walkthrough page exists and embeds `examples/governance_walkthrough.py`; example covers all three scenarios | integration | `uv run pytest tests/test_docs_phase30.py::test_governance_walkthrough_shape -x` + `uv run python examples/governance_walkthrough.py` (SKIP path) | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `uv run pytest tests/test_docs_phase30.py -x` (fast — just config and shape checks)
- **Per wave merge:** `uv run mkdocs build --strict` + `uv run python examples/hello.py` + `uv run python examples/first_graph.py` + `uv run python examples/service_mode_with_approval.py` + `uv run python examples/governance_walkthrough.py` (all SKIP-clean)
- **Phase gate:** full suite `uv run pytest -v` + `uv run mkdocs build --strict` green

### Wave 0 Gaps

- [ ] `tests/test_docs_phase30.py` — covers SITE-01, SITE-02, SITE-04, DOCS-01, DOCS-02, DOCS-05 via config/shape assertions
- [ ] `tests/conftest.py` — no changes needed (existing pytest setup is sufficient)
- [ ] Framework install: none — pytest 8 + pytest-asyncio already in `[dependency-groups].dev`

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes (tutorial shows Bearer token to approve) | Existing `ServiceAuthenticator` / `JWTBearerTokenVerifier` — tutorial must show how to get/use a token, not hard-code secrets |
| V3 Session Management | no | — |
| V4 Access Control | yes (tutorial implicitly exercises `Permission.APPROVAL_RESOLVE`) | Existing `require_permission` in service layer |
| V5 Input Validation | yes (mkdocs config is yaml, plan must validate via `mkdocs build --strict`) | `mkdocs build --strict` + `pymdownx.snippets.check_paths: true` |
| V6 Cryptography | no | — |
| V10 Malicious Code | yes (examples/ runs against real LLMs; fork PRs must not exfiltrate secrets) | Forked PR CI cannot access repo secrets; the SKIP pattern is the standard mitigation (matches `examples/hello.py`) |

### Known Threat Patterns for mkdocs-material + GH Pages

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Committed API key in example file | Information Disclosure | `os.environ.get(...)` + SKIP, never hardcode; CI lints via `gitleaks` (future — out of scope this phase) |
| Supply-chain: mkdocs plugin ships malicious code | Tampering | Pin all `[docs]` extras to exact or tight ranges; rely on `uv`'s hash-locking |
| Leaked docs preview shows in-progress sensitive info | Information Disclosure | PR preview is deferred, so there's no preview to leak; main-branch deploy is intended public |
| Markdown XSS via user-supplied content | Tampering | `mkdocs-material` sanitizes by default; no user-submitted content in this phase |
| Forked-PR docs build runs arbitrary Python via `pymdownx.snippets` | RCE | Snippets extension only reads files, doesn't exec them. The `examples.yml` job *does* exec examples — mitigated by fork-CI secrets isolation + SKIP guard |

## Sources

### Primary (HIGH confidence — codebase)

- `/Users/dondoe/coding/zeroth/examples/hello.py` — Phase 28 example pattern (env-gated SKIP)
- `/Users/dondoe/coding/zeroth/pyproject.toml` — base deps, extras structure, no `[project.scripts]`
- `/Users/dondoe/coding/zeroth/src/zeroth/core/service/entrypoint.py` — uvicorn factory, `PORT` / `ZEROTH_DEPLOYMENT_REF` env contract
- `/Users/dondoe/coding/zeroth/src/zeroth/core/service/approval_api.py` — approval HTTP routes verified
- `/Users/dondoe/coding/zeroth/src/zeroth/core/service/run_api.py` — `POST /runs`, `GET /runs/{id}`, `RunPublicStatus` enum
- `/Users/dondoe/coding/zeroth/src/zeroth/core/service/audit_api.py` — `GET /runs/{id}/timeline`, `GET /deployments/{ref}/audits`
- `/Users/dondoe/coding/zeroth/src/zeroth/core/service/bootstrap.py` — 432 LOC bootstrap (constrains tutorial shape)
- `/Users/dondoe/coding/zeroth/src/zeroth/core/orchestrator/runtime.py` — `RuntimeOrchestrator` dataclass
- `/Users/dondoe/coding/zeroth/src/zeroth/core/graph/models.py` — `Graph`, `AgentNode`, `HumanApprovalNode`, `Edge`
- `/Users/dondoe/coding/zeroth/src/zeroth/core/policy/models.py` — `Capability`, `PolicyDefinition`, `PolicyDecision`
- `/Users/dondoe/coding/zeroth/.github/workflows/ci.yml` — existing `uv sync --all-groups` + `uv run pytest -v` pattern
- `/Users/dondoe/coding/zeroth/README.md` — existing "Choose your path" framing already in README

### Secondary (MEDIUM confidence — external docs)

- [Material for MkDocs — Publishing your site](https://squidfunk.github.io/mkdocs-material/publishing-your-site/) — recommended workflow, `mkdocs gh-deploy --force`, `contents: write`
- [PyMdown Extensions — Snippets](https://facelessuser.github.io/pymdown-extensions/extensions/snippets/) — `--8<--` syntax, `base_path`, `check_paths`
- [Material for MkDocs — Python Markdown Extensions](https://squidfunk.github.io/mkdocs-material/setup/extensions/python-markdown-extensions/) — plugin list alignment with D-14
- [mkdocs-material · PyPI](https://pypi.org/project/mkdocs-material/) — 9.7.6 release confirmation (WebSearch)
- [What MkDocs 2.0 means — Material for MkDocs blog](https://squidfunk.github.io/mkdocs-material/blog/2026/02/18/mkdocs-2.0/) — context for the "last version before Zensical rebrand" note
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — considered as an alternative, rejected in favor of built-in `gh-deploy`

### Tertiary (LOW confidence)

None this phase — every claim above is either verified in the codebase or cited against an official source.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — mkdocs-material / pymdownx / section-index are commodity, verified against official docs.
- Architecture (site): HIGH — workflow and mkdocs.yml shapes are lifted from mkdocs-material's own recommendations.
- Architecture (tutorial examples): MEDIUM — the exact Python API shape is verified, but whether a 30-line user-facing example is achievable depends on a scope decision (quickstart helper module — see Open Question 2).
- Pitfalls: HIGH — all seven pitfalls are derived from verified sources or direct repo inspection.
- Validation architecture: HIGH — pytest 8 is already wired; shape tests are trivial to add.

**Research date:** 2026-04-11
**Valid until:** 2026-04-25 (14 days — shorter than a typical 30 because mkdocs-material is entering end-of-feature-life; the underlying advice is stable but the ecosystem is in transition)
