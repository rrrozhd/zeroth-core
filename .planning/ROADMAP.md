# Roadmap: Zeroth

## Milestones

- v1.0 Runtime Foundation — Phases 1-9 (shipped 2026-03-27)
- v1.1 Production Readiness — Phases 11-21 (shipped 2026-04-09)
- v2.0 Zeroth Studio — Phases 22-26 (partially shipped: 22-23 done; 24-26 moved to `zeroth-studio` repo under v3.0)
- v3.0 Core Library Extraction, Studio Split & Documentation — Phases 27-32 (in progress)

## Phases

<details>
<summary>v1.0 Runtime Foundation (Phases 1-9) — SHIPPED 2026-03-27</summary>

- [x] Phase 1: Core Foundation (2/2 plans) — completed 2026-03-19
- [x] Phase 2: Execution Core (2/2 plans) — completed 2026-03-19
- [x] Phase 3: Platform Control (2/2 plans) — completed 2026-03-19
- [x] Phase 4: Deployment Surface (1/1 plan) — completed 2026-03-20
- [x] Phase 5: Integration & Polish (1/1 plan) — completed 2026-03-26
- [x] Phase 6: Identity & Tenant Governance (1/1 plan) — completed 2026-03-27
- [x] Phase 7: Transparent Governance & Provenance (1/1 plan) — completed 2026-03-27
- [x] Phase 8: Runtime Security Hardening (1/1 plan) — completed 2026-03-27
- [x] Phase 9: Durable Control Plane & Production Operations (1/1 plan) — completed 2026-03-27

</details>

<details>
<summary>v1.1 Production Readiness (Phases 11-21) — SHIPPED 2026-04-09</summary>

- [x] Phase 11: Config & Postgres Storage (3/3 plans) — completed 2026-04-06
- [x] Phase 12: Real LLM Providers & Retry (3/3 plans) — completed 2026-04-06
- [x] Phase 13: Regulus Economics Integration (3/3 plans) — completed 2026-04-07
- [x] Phase 14: Memory Connectors & Container Sandbox (5/5 plans) — completed 2026-04-07
- [x] Phase 15: Webhooks & Approval SLA (3/3 plans) — completed 2026-04-07
- [x] Phase 16: Distributed Dispatch & Horizontal Scaling (3/3 plans) — completed 2026-04-07
- [x] Phase 17: Deployment Packaging & Operations (3/3 plans) — completed 2026-04-07
- [x] Phase 18: Cross-Phase Integration Wiring (2/2 plans) — completed 2026-04-08
- [x] Phase 19: Agent Node LLM API Parity (3/3 plans) — completed 2026-04-08
- [x] Phase 20: Bootstrap Integration Wiring (1/1 plan) — completed 2026-04-09
- [x] Phase 21: Health Probe Fix & Tech Debt (1/1 plan) — completed 2026-04-09

</details>

<details>
<summary>v2.0 Zeroth Studio (Phases 22-26) — PARTIALLY SHIPPED; 24-26 moved to zeroth-studio repo</summary>

- [x] Phase 22: Canvas Foundation & Dev Infrastructure (6/6 plans) — completed 2026-04-09
- [x] Phase 23: Canvas Editing UX (4/4 plans) — completed 2026-04-09
- [→] Phase 24: Execution & AI Authoring — **moved to `zeroth-studio` repo** (part of v3.0 split)
- [→] Phase 25: Governance Visualization — **moved to `zeroth-studio` repo** (part of v3.0 split)
- [→] Phase 26: Versioning & Collaboration — **moved to `zeroth-studio` repo** (part of v3.0 split)

</details>

### v3.0 Core Library Extraction, Studio Split & Documentation (In Progress)

**Milestone Goal:** Ship Zeroth as a pip-installable Python library (`zeroth-core`) with in-depth documentation covering every major subsystem, while moving the Vue Studio UI into a separate repo so the two evolve independently.

- [x] **Phase 27: Monolith Archive & Namespace Rename** — Preserve the monolithic repo in a multi-layer archive, then relocate all Python source from `zeroth.*` to `zeroth.core.*` (pure rename, zero deletions). Much of this is already done ad-hoc in `/tmp/zeroth-split/zeroth-core-build/` — this phase formalizes and verifies it. (4/4 plans complete, completed 2026-04-10)
- [x] **Phase 28: PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`)** — Publish both packages to PyPI via trusted publisher, with optional-dependency extras declared and verified installable end-to-end. (completed 2026-04-11)
- [x] **Phase 29: Studio Repo Split** — Create `rrrozhd/zeroth-studio` as a public repo with preserved git history, independent CI, HTTP-only consumption of `zeroth-core`, and cross-repo compatibility matrix. (completed 2026-04-11)
- [x] **Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough** — Stand up mkdocs-material with Diátaxis IA, deploy on every main commit, and ship the "first working path" pages: landing, Getting Started, governance walkthrough tutorial. (completed 2026-04-11)
- [ ] **Phase 31: Subsystem Concepts, Usage Guides, Cookbook & Examples** — Write Concept + Usage Guide pages for all ~20 subsystems, author 10+ cookbook recipes, and ship the CI-tested `examples/` directory.
- [ ] **Phase 32: Reference Docs, Deployment & Migration Guide** — Auto-generate Python API reference via mkdocstrings, render HTTP API reference from OpenAPI, auto-generate configuration reference from pydantic-settings, and write the deployment and migration guides.

## Phase Details

### Phase 22: Canvas Foundation & Dev Infrastructure
**Goal**: Users can create workflow graphs by placing nodes and drawing edges on an interactive canvas, save and load them via the API, and work within a responsive three-panel Studio layout served from Docker
**Depends on**: Phase 21 (v1.1 shipped)
**Requirements**: CANV-01, CANV-02, CANV-06, CANV-09, CANV-10, API-01, API-03, INFRA-01, INFRA-02
**Success Criteria** (what must be TRUE):
  1. User can drag nodes onto a pannable, zoomable canvas and draw edges between typed node ports
  2. User can save a workflow graph and reload it in a new browser session with all nodes and edges intact
  3. User can navigate the canvas with pan, zoom, fit-to-view, and minimap
  4. User sees a responsive three-panel layout (workflow rail, canvas, inspector) with collapsible panels
  5. Studio frontend is served via Nginx alongside FastAPI in Docker, and frontend types are generated from the backend OpenAPI spec
**Plans:** 6/6 plans complete
**UI hint**: yes

### Phase 23: Canvas Editing UX
**Goal**: Users have a complete editing experience with categorized node palette, property inspector, auto-layout, undo/redo, keyboard shortcuts, and validation feedback
**Depends on**: Phase 22
**Requirements**: CANV-03, CANV-04, CANV-05, CANV-07, CANV-08, AUTH-06
**Success Criteria** (what must be TRUE):
  1. User can browse and search node types in a categorized sidebar palette
  2. User can select a node and view/edit its properties in the inspector panel
  3. User can auto-layout the graph into a readable DAG arrangement
  4. User can undo and redo canvas operations, and use keyboard shortcuts for common actions (delete, select-all, copy/paste, duplicate)
  5. User can see validation indicators on nodes with missing fields, invalid connections, or type mismatches
**Plans:** 4/4 plans complete
**UI hint**: yes

### Phase 27: Monolith Archive & Namespace Rename
**Goal**: The monolithic Zeroth repo is preserved in a multi-layer archive, and all Python source is relocated into the `zeroth.core.*` PEP 420 namespace package with the full existing test suite passing on the renamed layout
**Depends on**: Phase 23 (v2.0 partial ship)
**Requirements**: ARCHIVE-01, ARCHIVE-02, ARCHIVE-03, RENAME-01, RENAME-02, RENAME-03, RENAME-04, RENAME-05
**Success Criteria** (what must be TRUE):
  1. The monolith is recoverable from three independent archive layers (local tarball, local bare mirror, `rrrozhd/zeroth-archive` on GitHub) and the archived repo carries a visible "archived — see zeroth-core/zeroth-studio" notice
  2. All 36 pre-existing worktree branches, both stashes, and the detached-HEAD worktree are preserved in the archive and can be checked out from the bare mirror
  3. All Python source lives under `zeroth.core.*` with no top-level `zeroth/__init__.py` (PEP 420 namespace package), no deletions, no functional changes
  4. The full existing test suite (280+ tests) passes against the renamed package with zero skips and zero regressions
  5. Docstring coverage on the `zeroth.core.*` public surface reaches ≥90% (measured by `interrogate`) using a single consistent style (Google-style)
**Plans**: 4/4 plans complete
**Notes**: Completed 2026-04-10. The final verification pass added the CI/docstring gate, captured post-rename interrogate/pytest artifacts, fixed the remaining codemod regressions in `live_scenarios/`, and proved the rename introduced no new `FAILED`/`ERROR`/`SKIPPED` entries versus baseline.

### Phase 28: PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`)
**Goal**: Both `econ-instrumentation-sdk` and `zeroth-core` are published to PyPI via GitHub Actions trusted publisher, a clean-venv install of `zeroth-core[all]` succeeds, and every declared optional extra is verified installable
**Depends on**: Phase 27
**Requirements**: PKG-01, PKG-02, PKG-03, PKG-04, PKG-05, PKG-06
**Success Criteria** (what must be TRUE):
  1. `econ-instrumentation-sdk` is live on PyPI at a stable version, and `zeroth-core` depends on it via a PyPI version constraint, not a local file path
  2. `pyproject.toml` declares the full set of optional-dependency extras (`[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]`) and each extra resolves and installs cleanly
  3. Repository root contains `CHANGELOG.md` (keepachangelog format), `LICENSE`, and `CONTRIBUTING.md` suitable for public PyPI consumption
  4. PyPI releases are published by GitHub Actions using trusted publisher (OIDC), with no long-lived API tokens in the repo or CI
  5. In a fresh virtualenv, `pip install zeroth-core` followed by running the Getting Started hello example produces working output end-to-end
**Plans:** 3/3 plans complete

Plans:
- [x] 28-01-pyproject-metadata-and-extras-PLAN.md — pyproject.toml rewrite: bump to 0.1.1, carve six extras, Apache-2.0 SPDX, py.typed, keep wheel target (resolved Q3)
- [x] 28-02-repo-metadata-and-hello-example-PLAN.md — LICENSE/CHANGELOG/CONTRIBUTING, examples/hello.py, STATE.md blocker reconciliation
- [x] 28-03-release-workflow-and-extras-verification-PLAN.md — trusted-publisher release workflow + verify-extras CI matrix + README install section (USER ACTION: register pypi + testpypi publishers)

### Phase 29: Studio Repo Split
**Goal**: `rrrozhd/zeroth-studio` exists as a public repo with preserved git history, passing independent CI, HTTP-only consumption of `zeroth-core`, and a documented cross-repo compatibility matrix
**Depends on**: Phase 28
**Requirements**: STUDIO-01, STUDIO-02, STUDIO-03, STUDIO-04, STUDIO-05
**Success Criteria** (what must be TRUE):
  1. `rrrozhd/zeroth-studio` is a public GitHub repository containing the Vue 3 + Vue Flow frontend with full git history preserved (via subtree or git-filter-repo)
  2. `zeroth-studio` has its own CI pipeline (lint, typecheck, build, test) that passes on its default branch without touching or importing from `zeroth-core`
  3. `zeroth-studio`'s only contract with `zeroth-core` is HTTP/OpenAPI — its frontend types are generated from the `zeroth-core` OpenAPI spec via `openapi-typescript`
  4. Both repos' READMEs cross-link, and a `zeroth-studio × zeroth-core` compatibility matrix is documented and maintained
  5. A developer can clone `zeroth-studio`, run `npm install && npm run dev`, and develop against a running `zeroth-core` service without any cross-repo source dependencies
**Plans:** 4/4 plans complete
Plans:
- [x] 29-01-preflight-in-zeroth-core-PLAN.md — add scripts/dump_openapi.py + commit snapshot; wire VITE_API_BASE_URL through apps/studio; add ESLint flat config + split typecheck/build + bundle standalone nginx.conf (in-place, so filter-repo carries it)
- [x] 29-02-filter-repo-extract-and-create-remote-PLAN.md — fresh --no-local clone to /tmp/zeroth-studio-split, run git filter-repo for the three paths, gh repo create rrrozhd/zeroth-studio, two-step push to main
- [x] 29-03-bootstrap-new-repo-ci-and-types-PLAN.md — copy openapi snapshot + generate types.gen.ts, add LICENSE/CHANGELOG/CONTRIBUTING/README with compat matrix, add GitHub Actions CI (lint/typecheck/build/test/drift-check), push and verify green
- [x] 29-04-cleanup-zeroth-core-PLAN.md — safety-gate on zeroth-studio CI green, delete apps/studio, apps/studio-mockups, tests/studio from zeroth-core (preserving tests/test_studio_api.py), add Studio section to zeroth-core README
**UI hint**: yes (frontend repo, but no new UI features in this phase — move only)

### Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough
**Goal**: The `zeroth-core` documentation site is live on a public URL, built with mkdocs-material using explicit Diátaxis IA, and contains the complete "first working path" — landing page, 3-section Getting Started tutorial, and a Governance Walkthrough showcasing Zeroth's differentiator
**Depends on**: Phase 28
**Requirements**: SITE-01, SITE-02, SITE-03, SITE-04, DOCS-01, DOCS-02, DOCS-05
**Success Criteria** (what must be TRUE):
  1. The docs site is built by mkdocs-material with four top-level Diátaxis sections (Tutorials / How-to Guides / Concepts / Reference), has built-in search, and auto-generates a site map
  2. A GitHub Actions workflow builds and deploys the docs to a public URL on every commit to `main`, and pull requests get preview deploys
  3. The landing page shows a 10-line hello-world, install snippet, and a "Choose your path" split between embedding as a library and running as a service
  4. Getting Started is a single linear 3-section tutorial (install → first graph with one agent/tool/LLM → run in service mode with an approval gate) that produces first working output in under 5 minutes and completes in under 30
  5. A Governance Walkthrough tutorial runs end-to-end with an approval gate stopping execution, an auditor reviewing the trail, and a policy blocking a tool call
**Plans:** 5/5 plans complete
Plans:
- [x] 30-01-quickstart-helper-module-PLAN.md — ship zeroth.core.examples.quickstart tutorial helper + Wave 0 test scaffold (DOCS-02)
- [x] 30-02-docs-site-scaffold-PLAN.md — mkdocs.yml, [docs] extra, Diátaxis doc tree, landing page with Choose Your Path (SITE-01, SITE-04, DOCS-01)
- [x] 30-03-getting-started-tutorial-PLAN.md — examples/first_graph.py + examples/approval_demo.py, Getting Started pages, examples.yml CI (DOCS-01, DOCS-02)
- [x] 30-04-governance-walkthrough-tutorial-PLAN.md — examples/governance_walkthrough.py covering approval+auditor+policy block + tutorial page (DOCS-05)
- [x] 30-05-docs-deploy-workflow-PLAN.md — docs.yml GHA (build-on-PR + deploy-on-main), README link, phase-gate validation, SITE-03 deferral recorded, GH Pages enablement checkpoint (SITE-02; SITE-03 deferred)
**UI hint**: yes (docs site is the UI)

### Phase 31: Subsystem Concepts, Usage Guides, Cookbook & Examples
**Goal**: Every major `zeroth.core.*` subsystem has a paired Concept page and Usage Guide on the docs site, the Cookbook contains at least 10 cross-subsystem recipes, and an `examples/` directory with CI-tested runnable `.py` files covers the main subsystems
**Depends on**: Phase 30
**Requirements**: DOCS-03, DOCS-04, DOCS-06, DOCS-12
**Success Criteria** (what must be TRUE):
  1. Every major subsystem (graph, orchestrator, agents, execution units, memory, contracts, runs, conditions, mappings, policy, approvals, audit, secrets, identity, guardrails, dispatch, economics, storage, service, threads) has a Concept page explaining what it is, why it exists, and where it fits
  2. Every major subsystem has a Usage Guide (Overview → Minimal example → Common patterns → Pitfalls → Reference cross-link) paired with its Concept page
  3. The Cookbook section contains at least 10 cross-subsystem recipes covering the most common Zeroth tasks (approval steps, memory attachment, budget capping, sandboxing, webhook retry, etc.)
  4. The repo root contains an `examples/` directory with runnable `.py` files (no notebooks) exercising the main subsystems
  5. A CI job smoke-tests every file in `examples/` on every commit to `main`, and the job is green
**Plans:** 2/5 plans executed
Plans:
- [x] 31-01-subsystems-batch-a-graph-execution-PLAN.md — Concept + Usage Guide for graph, orchestrator, agents, execution_units, conditions (10 pages)
- [x] 31-02-subsystems-batch-b-data-state-PLAN.md — Concept + Usage Guide for mappings, memory, storage, contracts, runs (10 pages)
- [ ] 31-03-subsystems-batch-c-governance-PLAN.md — Concept + Usage Guide for policy, approvals, audit, guardrails, identity (10 pages)
- [x] 31-04-subsystems-batch-d-platform-PLAN.md — Concept + Usage Guide for secrets, dispatch, econ, service, webhooks (10 pages; threads→webhooks substitution)
- [ ] 31-05-cookbook-examples-and-nav-finalize-PLAN.md — 10 cookbook recipes, 10 runnable examples, CI matrix extension, nav finalize, mkdocs strict build gate
**UI hint**: yes (docs content)

### Phase 32: Reference Docs, Deployment & Migration Guide
**Goal**: The docs site has a complete Reference quadrant (Python API auto-generated from docstrings, HTTP API rendered from OpenAPI, Configuration reference auto-generated from pydantic-settings), plus a Deployment Guide covering every supported mode and a Migration Guide from the monolith layout
**Depends on**: Phase 30 (and Phase 27 for the renamed import paths referenced by the API reference)
**Requirements**: DOCS-07, DOCS-08, DOCS-09, DOCS-10, DOCS-11
**Success Criteria** (what must be TRUE):
  1. Python API Reference is auto-generated from docstrings via mkdocstrings + Griffe for the full `zeroth.core.*` public surface, cross-linked from narrative pages
  2. HTTP API Reference is rendered from the `zeroth-core` FastAPI OpenAPI spec and published alongside the Python reference on the docs site
  3. Configuration Reference is auto-generated from the pydantic-settings schemas and documents every env var, its default, and whether it is a secret
  4. Deployment Guide covers local development, docker-compose, standalone service mode, embedded-in-host-app mode, and deployments with and without the Regulus companion service
  5. Migration Guide walks an existing monolith user through the switch to `zeroth.core.*` — import rename pattern, econ SDK path swap, env var changes, and Docker image retag
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order. v3.0 runs 27 → 28 → 29/30 (parallelizable after 28) → 31 → 32.

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Core Foundation | v1.0 | 2/2 | Complete | 2026-03-19 |
| 2. Execution Core | v1.0 | 2/2 | Complete | 2026-03-19 |
| 3. Platform Control | v1.0 | 2/2 | Complete | 2026-03-19 |
| 4. Deployment Surface | v1.0 | 1/1 | Complete | 2026-03-20 |
| 5. Integration & Polish | v1.0 | 1/1 | Complete | 2026-03-26 |
| 6. Identity & Tenant Governance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 7. Transparent Governance & Provenance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 8. Runtime Security Hardening | v1.0 | 1/1 | Complete | 2026-03-27 |
| 9. Durable Control Plane & Production Operations | v1.0 | 1/1 | Complete | 2026-03-27 |
| 11. Config & Postgres Storage | v1.1 | 3/3 | Complete | 2026-04-06 |
| 12. Real LLM Providers & Retry | v1.1 | 3/3 | Complete | 2026-04-06 |
| 13. Regulus Economics Integration | v1.1 | 3/3 | Complete | 2026-04-07 |
| 14. Memory Connectors & Container Sandbox | v1.1 | 5/5 | Complete | 2026-04-07 |
| 15. Webhooks & Approval SLA | v1.1 | 3/3 | Complete | 2026-04-07 |
| 16. Distributed Dispatch & Horizontal Scaling | v1.1 | 3/3 | Complete | 2026-04-07 |
| 17. Deployment Packaging & Operations | v1.1 | 3/3 | Complete | 2026-04-07 |
| 18. Cross-Phase Integration Wiring | v1.1 | 2/2 | Complete | 2026-04-08 |
| 19. Agent Node LLM API Parity | v1.1 | 3/3 | Complete | 2026-04-08 |
| 20. Bootstrap Integration Wiring | v1.1 | 1/1 | Complete | 2026-04-09 |
| 21. Health Probe Fix & Tech Debt | v1.1 | 1/1 | Complete | 2026-04-09 |
| 22. Canvas Foundation & Dev Infrastructure | v2.0 | 6/6 | Complete | 2026-04-09 |
| 23. Canvas Editing UX | v2.0 | 4/4 | Complete | 2026-04-09 |
| 24. Execution & AI Authoring | v2.0 | — | Moved to `zeroth-studio` | — |
| 25. Governance Visualization | v2.0 | — | Moved to `zeroth-studio` | — |
| 26. Versioning & Collaboration | v2.0 | — | Moved to `zeroth-studio` | — |
| 27. Monolith Archive & Namespace Rename | v3.0 | 4/4 | Complete | 2026-04-10 |
| 28. PyPI Publishing (econ-sdk + zeroth-core) | v3.0 | 3/3 | Complete   | 2026-04-11 |
| 29. Studio Repo Split | v3.0 | 4/4 | Complete   | 2026-04-11 |
| 30. Docs Site Foundation, Getting Started & Governance Walkthrough | v3.0 | 5/5 | Complete   | 2026-04-11 |
| 31. Subsystem Concepts, Usage Guides, Cookbook & Examples | v3.0 | 2/5 | In Progress|  |
| 32. Reference Docs, Deployment & Migration Guide | v3.0 | 0/? | Not started | — |
