# Architecture Research — v3.0 Core Library Extraction, Studio Split & Documentation

**Domain:** Python library packaging + monorepo-to-multirepo migration + documentation hosting
**Researched:** 2026-04-10
**Confidence:** HIGH (verified against existing specs, current PROJECT.md decisions, and the scratch `/tmp/zeroth-split/zeroth-core-build/` that already proves feasibility)

---

## Scope disclaimer

This document does **not** re-research the internal Zeroth runtime (graph/orchestrator/service/etc.) — that architecture is validated and unchanged. It focuses exclusively on the **packaging, repo topology, CI/CD, docs, and cross-repo release coordination** required by the v3.0 milestone.

Important pivot vs. the older superpowers spec (`2026-04-10-zeroth-core-platform-split-design.md`): the earlier design split core vs. platform into two Python repos. PROJECT.md supersedes that — the decision is now a **pure rename** of everything into `zeroth.core.*` with no file-level core/platform split. The Vue 3 Studio frontend is the only piece that moves to its own repo. All Python code (including FastAPI service, storage, migrations, econ, etc.) stays in `zeroth-core`.

---

## System Overview — the v3.0 topology

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              GitHub (rrrozhd)                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐      │
│  │  zeroth-archive  │   │   zeroth-core    │   │  zeroth-studio   │      │
│  │ (private, RO)    │   │ (public, active) │   │ (public, active) │      │
│  │ full history +   │   │ Python library + │   │ Vue 3 + Vue Flow │      │
│  │ worktrees blob   │   │ FastAPI service  │   │ frontend only    │      │
│  └──────────────────┘   │ + Alembic + docs │   └────────┬─────────┘      │
│                         └────────┬─────────┘            │                │
│  ┌──────────────────┐            │              HTTP (v1 API only)       │
│  │     regulus      │            │                      │                │
│  │ (public)         │            │                      │                │
│  │ sdk/python only  │            │                      │                │
│  │ published to PyPI│            │                      │                │
│  └────────┬─────────┘            │                      │                │
│           │                      │                      │                │
└───────────┼──────────────────────┼──────────────────────┼────────────────┘
            │                      │                      │
            │                      │                      │
     ┌──────▼──────┐         ┌─────▼──────┐        ┌──────▼──────┐
     │    PyPI     │         │    PyPI    │        │   static    │
     │econ-instr.. │◄────dep─│ zeroth-core│        │  (GH Pages  │
     │             │         │            │        │  or ghcr)   │
     └─────────────┘         └─────┬──────┘        └─────────────┘
                                   │
                             ┌─────▼──────┐
                             │  GH Pages  │
                             │ (docs site │
                             │  for core) │
                             └────────────┘
```

### Component responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| `zeroth-archive` (private GH repo) | Immutable historical record of pre-split monolith including `.claude/worktrees/*` | `gh repo create --private`, push `pre-split-head`, `gh repo edit --archived` |
| `zeroth-core` (public GH repo) | Full Python library: `zeroth.core.*` namespace, FastAPI service, Alembic migrations, Docker image, Python tests, docstring-driven docs site | Hatchling wheel, published to PyPI as `zeroth-core` |
| `zeroth-studio` (public GH repo) | Vue 3 + Vue Flow frontend, Dockerfile for nginx image, studio-specific E2E tests, Studio roadmap (v2.0 phases 24–26) | Vite build, static assets, no Python |
| `regulus` (public GH repo) | Hosts `econ-instrumentation-sdk` Python package (under `sdk/python/`) plus the backend/demo/dashboard that the user already has locally | `setuptools` build from `sdk/python/`, published to PyPI as `econ-instrumentation-sdk` |
| PyPI distributions | `econ-instrumentation-sdk` (prereq), `zeroth-core` (consumer) | Trusted Publishing via OIDC from each repo's GH Actions |
| GH Pages (on `zeroth-core`) | Public, versioned docs site for the library | MkDocs Material + mkdocstrings, auto-published from `main` + release tags |

---

## Answers to the 10 architecture questions

### 1. Repo topology — two repos vs. uv workspace monorepo

**Recommendation: TWO repos (`zeroth-core`, `zeroth-studio`) — confirm the user's stated intent.**

A uv workspace monorepo would be simpler for *local* dev (single `uv sync` installs both; cross-package refactors are atomic), but it is the wrong shape for this milestone's goals:

| Concern | Two repos | uv workspace monorepo |
|---|---|---|
| Independent release cadence (core on PyPI, Studio on static hosting) | OK native | requires path-filtered workflows + per-package tag schemes |
| Clear public surface area for Python consumers | OK `pip install zeroth-core` is obviously a Python SDK | users land in a repo with Vue code and get confused |
| CI simplicity | OK each repo has one ci.yml and one publish.yml | one repo with matrix builds + path filters + skipped jobs |
| Studio phases 24–26 evolve on their own roadmap | OK separate PROGRESS.md and issue tracker | mingled issues, mingled PRs |
| Cross-cutting refactors (change API + Studio client at once) | two PRs, coordinate versions | OK atomic |
| License, README, README badges, stargazers | OK each tells its own story | one README has to cover both |
| Contributor onboarding for Studio-only devs | OK `git clone zeroth-studio && npm install` | forced to install Python toolchain |

The monorepo wins on atomic refactors, but that's only a real win when the two packages share code. Here they share **only an HTTP contract** — no shared Python modules, no shared Vue code. The OpenAPI schema is the coupling point, not source code. Two repos is the right topology.

**Keep:** local parent directory `/Users/dondoe/coding/zeroth/` as a non-repo container with `zeroth-core/` and `zeroth-studio/` as sibling subdirs. This matches the existing scratch build at `/tmp/zeroth-split/zeroth-core-build/` and matches the superpowers plan's final layout.

### 2. PEP 420 namespace — `zeroth.core.*` with no top-level `zeroth/__init__.py`

**Recommendation: yes, ship as PEP 420 implicit namespace. Confirm.**

The scratch build already does this correctly: `src/zeroth/core/__init__.py` exists, but `src/zeroth/__init__.py` does **not**. Hatchling's `packages = ["src/zeroth/core"]` (not `["src/zeroth"]`) is the right wheel layout — it publishes only the `core` subtree, leaving `zeroth` as a namespace owned by no one.

**Gotchas to watch:**

1. **Never add `src/zeroth/__init__.py`.** The moment any distribution ships that file, `zeroth` becomes a regular package owned by that wheel and sibling namespaces collide. The current `tool.hatch.build.targets.wheel.packages` must say `["src/zeroth/core"]`, not `["src/zeroth"]`. This needs a phase-1 grep and an importlinter or custom CI check.
2. **Editable installs.** `uv pip install -e .` with hatchling must use `packages = ["src/zeroth/core"]`. Verify by `uv build && unzip -l dist/*.whl | grep __init__` — only `zeroth/core/__init__.py` (and its subpackages) should appear, never top-level `zeroth/__init__.py`.
3. **Tests must import via `zeroth.core.*`, not `zeroth.*`.** The codemod already handled this; keep importlinter rules in CI to catch regressions.
4. **`mkdocstrings` API reference generation** needs the module path `zeroth.core` — confirm this works with namespace packages (it does, but mkdocstrings's `paths: [src]` config is required).
5. **`py.typed` placement.** Put it at `src/zeroth/core/py.typed`, never at `src/zeroth/py.typed`.

**Future `zeroth.studio.*` Python package:** yes, it can ship cleanly. A later `zeroth-studio-client` wheel could own `zeroth.studio.*` (e.g. generated OpenAPI client stubs) and coexist with `zeroth.core.*` because neither wheel ships the top-level `zeroth/__init__.py`. This is exactly the use case PEP 420 was designed for. The only rule: every wheel under the `zeroth` namespace must be namespace-compatible (no `zeroth/__init__.py`).

### 3. Where docs live

**Recommendation: option (a) — docs live inside `zeroth-core/docs/`, built by zeroth-core CI, published to GH Pages from the zeroth-core repo.**

Reasoning:

- The docs are overwhelmingly about the Python library (getting started, subsystem guides, API reference for `zeroth.core.*`, HTTP API reference from the OpenAPI spec). Every one of those is driven by source code that lives in `zeroth-core`. Co-locating docs with source is the standard and eliminates cross-repo sync pain.
- `mkdocstrings` requires the Python package to be importable at build time. That only works cleanly when docs build inside the repo that owns the source.
- Studio's docs are minimal (README + a short contributing guide + a link out to `zeroth-core`'s deployment docs) and can live in `zeroth-studio/README.md` without needing a full docs site.
- Option (b) `zeroth-docs` as a third repo adds a sync step and gives us nothing — it would still need to clone the two source repos at build time to run mkdocstrings, which is strictly worse than building in place.
- Option (c) two Pages sites gives users two bookmarks to remember and fragments the docs URL. Bad UX.

**Mechanics:**

- Use **MkDocs Material + mkdocstrings[python] + mike** (for versioned docs).
- `mkdocs.yml` at repo root with `docs/` as source.
- `mike` publishes versioned docs to the `gh-pages` branch: `latest` alias tracks the most recent tag, historical versions remain addressable at `/v0.1.0/`, `/v0.2.0/`, etc.
- Custom domain optional (e.g. `docs.zeroth.dev`) — decide later, default to `rrrozhd.github.io/zeroth-core/`.
- Studio repo gets a one-line README link: "Backend docs: https://rrrozhd.github.io/zeroth-core".

### 4. Docstring coverage gap — harden before, or ship as-is?

**Recommendation: ship a first pass of docs with known gaps, THEN add a docstring hardening sub-phase as explicit roadmap work.**

Rationale:

- Blocking the 0.1.0 docs release on 100% docstring coverage is a multi-week tangent. The codebase has 27 modules and ~22K LOC; hardening all of them before any docs ship means no docs until that's done.
- mkdocstrings degrades gracefully: modules without docstrings show a class/function signature and a "No description available" placeholder. That is **useful** — it tells the user the symbol exists and what its shape is, even without prose.
- A gap inventory is cheap to generate and far more valuable than a blanket pass. Use `interrogate` (coverage tool) in CI to produce a coverage percentage and a per-module report. Treat it as a metric to track, not a blocker.

**Proposed flow:**

1. Phase N (docs foundation): stand up MkDocs + mkdocstrings with the current codebase. Publish `0.1.0-docs` with whatever coverage exists. Add `interrogate` to CI with a **soft** threshold (e.g. fail only if coverage drops below the starting baseline).
2. Phase N+1 (docstring hardening): dedicated sub-phase, `interrogate` raises threshold progressively (baseline → 80% → 95%), with module-by-module tickets for the gaps. Written guides (getting started, per-subsystem guides, integration recipes) are written by hand during this phase.
3. Phase N+2: API reference auto-regenerates on every push; written guides evolve incrementally.

Interrogate ships numbers; mkdocstrings ships pages. Don't conflate them.

### 5. Build order and dependency chain

**Recommendation: publish regulus SDK to PyPI FIRST, then swap zeroth-core's dep, then publish zeroth-core. Do not ship zeroth-core 0.1.0 with a git URL dep.**

Why:

- PyPI permits git URL dependencies in **editable/source installs** but NOT in wheels uploaded to PyPI. A wheel whose METADATA contains `econ-instrumentation-sdk @ git+https://...` will be rejected by PyPI's validator (PEP 508 direct URL metadata policy).
- Hatchling writes direct references into METADATA by default when `allow-direct-references = true`. A wheel can still be **built** locally with direct references, it just can't be **uploaded** to PyPI if the reference survives into metadata.
- Therefore: publish regulus SDK first. Swap the dep to a version specifier. Then publish core.

**Correct ordering:**

```
1. Regulus preservation + public repo setup
2. Regulus sdk/python → TestPyPI → PyPI (econ-instrumentation-sdk 0.1.0)
3. zeroth-core: swap file:// dep to econ-instrumentation-sdk>=0.1.0 from PyPI
4. zeroth-core: lock file regeneration + CI green
5. zeroth-core → TestPyPI → PyPI (zeroth-core 0.1.0)
6. Docs site first publish (can happen in parallel with step 5)
7. zeroth-studio repo creation + Dockerfile move + link README to core
```

**Special case: governai — CRITICAL FLAG.** PROJECT.md constrains governai to stay pinned at git commit `7452de4` for v3.0. This is **incompatible with publishing zeroth-core to PyPI** for the same direct-reference reason. Three options for roadmapper to surface as a user-decision gate:

a. **Push the governai maintainer for a PyPI release.** Best, but out of our control and may block the milestone indefinitely.
b. **Publish governai (or a private fork) to PyPI / TestPyPI / a private index.** Unplanned extra work, license concerns.
c. **Vendor the needed governai modules into `zeroth.core.governai_shim`.** Maintenance burden, license concerns.
d. **Ship zeroth-core 0.1.0 as source-only install via git URL, not PyPI.** Users do `pip install zeroth-core @ git+https://github.com/rrrozhd/zeroth-core@v0.1.0`. This preserves the milestone's "pip-installable" goal in a weaker form; a full PyPI release waits for a governai resolution.

**Roadmapper MUST create an explicit user-decision gate phase for this before zeroth-core publish is attempted.** Recommend (d) as the pragmatic default for 0.1.0, with (a)/(b) as follow-on milestone work.

**`allow-direct-references = true`** stays in pyproject.toml as long as any direct reference exists (governai). Wheel CI must include a `twine check` + a custom check that fails if any direct reference survives into the uploaded wheel's METADATA (to prevent silent PyPI upload failures).

### 6. Studio repo contents

**Recommendation: `zeroth-studio` contains ONLY the Vue 3 frontend + its build tooling + its nginx Dockerfile. No Python, no FastAPI routes.**

What moves to `zeroth-studio/`:

- `apps/studio/` → repo root (flatten; `apps/` prefix no longer needed).
- `apps/studio/Dockerfile` → repo root Dockerfile (nginx static-file server).
- `docker/nginx/studio.conf` → `zeroth-studio/nginx/studio.conf`.
- `docker/nginx/certs/` → `zeroth-studio/nginx/certs/` (dev certs only; prod comes from env).
- Studio E2E tests, if any (grep `apps/studio/tests/`).
- Studio-specific docs (currently under `docs/studio/` — verify during extraction).
- Studio-specific `PROGRESS.md` tracking phases 24–26.
- `.github/workflows/ci.yml` (lint + build + E2E) and `docker-publish.yml` (ghcr image push on tag).

What stays in `zeroth-core/`:

- Everything in `src/zeroth/core/studio/` (the studio **backend helpers** — graph authoring API routes, validation, etc.). These are HTTP endpoints the frontend calls. They belong with the rest of the FastAPI service.
- Everything in `src/zeroth/core/service/` (FastAPI app, bootstrap).
- Alembic migrations.
- `docker-compose.yml` — stays in zeroth-core but **the nginx service is removed**. Zeroth-core's compose file starts only the backend stack (zeroth + postgres + redis + regulus + sandbox-sidecar). Zeroth-studio's repo ships its own deployment guide plus a reference compose that composes both published images.
- The core Dockerfile (unchanged, still CMD `python -m zeroth.core.service.entrypoint`).

**No thin Python FastAPI app in zeroth-studio.** It's pure static assets served by nginx, talking to the zeroth-core backend over HTTP. This is the cleanest boundary. The HTTP contract is the OpenAPI spec generated by zeroth-core, which Studio can consume as a TypeScript type-generation source (optional future enhancement).

**Deployment glue (integrated deployment of both):** document it in `zeroth-core/docs/deployment/integrated.md` with a reference `docker-compose.yml` that pulls both published images from ghcr.

### 7. Migration path for existing users — `__init__.py` shim or clean break?

**Recommendation: CLEAN BREAK. No shim. Ship a migration guide and a one-liner codemod script instead.**

Reasoning:

- A shim `src/zeroth/__init__.py` that re-exports from `zeroth.core.*` **breaks PEP 420**. The moment that file exists, `zeroth` is a regular package, and future siblings (`zeroth.studio`, hypothetical `zeroth.governai`) cannot be installed alongside without collision. This is explicitly contrary to the PEP 420 decision in PROJECT.md.
- The existing public user base for `zeroth` as an installable is **zero** (it has never been published to PyPI; version is still 0.1.0). The only consumers are the user's own projects — a migration guide plus a codemod script handles them.
- The mechanical migration is trivially automatable: `grep -rl 'from zeroth\.' | xargs sed -i '' -E 's/from zeroth\.([a-z_]+)/from zeroth.core.\1/g'`. The scratch build at `/tmp/zeroth-split/zeroth-core-build/` already proves this rewrite works cleanly.

**What to ship instead:**

- `docs/migration/from-monolith.md` — clear before/after examples, rationale, one-liner codemod.
- `zeroth-core/scripts/migrate-imports.py` — runnable codemod shipped in the repo (not in the wheel).
- CHANGELOG entry calling this out as a breaking rename for any 0.0.x → 0.1.0 users.

### 8. Publishing CI

**Recommendation: tag-triggered publish + TestPyPI smoke-test stage + PyPI Trusted Publishing (OIDC).**

Standard 2026 practice:

```yaml
on:
  push:
    tags: ["v*"]

jobs:
  build:
    - uv build
    - uv run twine check dist/*
    - custom: fail if any direct reference in METADATA
  publish-testpypi:
    needs: build
    environment: testpypi
    permissions: { id-token: write }
    - pypa/gh-action-pypi-publish@release/v1
      with:
        repository-url: https://test.pypi.org/legacy/
  smoke-test:
    needs: publish-testpypi
    - pip install -i https://test.pypi.org/simple/ zeroth-core==${{ github.ref_name }}
    - python -c "from zeroth.core.graph import Graph; print('ok')"
  publish-pypi:
    needs: smoke-test
    environment: pypi
    permissions: { id-token: write }
    - pypa/gh-action-pypi-publish@release/v1
```

**Key details:**

1. **Trusted Publishing (OIDC)** — configured once on PyPI per project, no API tokens. Use `environment: pypi` as the GitHub environment gate; can require manual approval before final publish.
2. **Version source.** Recommend `hatch-vcs`: tags are single source of truth, impossible to get out of sync with source.
3. **`allow-direct-references = true`** — keep it (needed for local dev with governai git pin). The CI pre-publish step MUST include a custom "no direct references in METADATA" check that inspects the built wheel and fails the build if any direct reference survived. This catches the governai problem automatically and prevents silent PyPI upload rejections.
4. **Tag-triggered, not manual version-bump PRs.** Version bump PR still happens in prep, but the publish is triggered by `git tag v0.1.0 && git push --tags` from main after merge. Simpler, more auditable, no ambient state.

**Versioning scheme:** SemVer. 0.x.y allows breaking changes freely during pre-1.0. Ship 0.1.0 first.

### 9. Docs CI — rebuild frequency and rollback

**Recommendation:**

- **Build on every push to `main`** → publishes to `/dev/`.
- **Publish a pinned versioned build on every release tag `v*`** → publishes to `/<version>/` and updates `latest` alias.
- **PR builds are non-publishing but strict** — `mkdocs build --strict` fails on broken links or missing mkdocstrings references, so bad docs never reach `main`.
- Use **`mike`** for version management — keeps `latest`, `dev`, and each tagged version addressable.

```yaml
on:
  push:
    branches: [main]     # publishes to /dev/
    tags: ["v*"]         # publishes to /<tag>/ and sets latest alias
  pull_request: {}       # build --strict, no publish

jobs:
  docs:
    - uv sync --group docs
    - uv run mkdocs build --strict
    - if: push to main or tag
      uv run mike deploy --push --update-aliases ${{ version }} latest
```

**`--strict` is non-negotiable.** Broken mkdocstrings references (e.g. a renamed module) must break the build, not silently ship a 404.

**Rollback story:**
- GH Pages serves from `gh-pages` branch.
- A bad build is recoverable by `git revert` on `gh-pages`, by `mike delete <version>`, or by re-running the previous tag's workflow.
- Because `mike` deploys atomically, a failed deploy leaves the existing site untouched — partial failures don't corrupt the live site.

### 10. Regulus repo structure

**Recommendation: push `rrrozhd/regulus` with the existing directory layout intact (backend/, dashboard/, demo/, docs/, infra/, sdk/). Do NOT flatten.**

Reasoning:

- Flattening to a `regulus-sdk` repo with only `sdk/python/` means losing the backend, dashboard, demo, and infra — all of which are legitimate parts of the project the user wants to keep around.
- The PyPI publish workflow can live under `.github/workflows/publish-sdk.yml` with `working-directory: sdk/python/`. Single-directory publishing from a multi-directory repo is standard practice (pydantic-core, astral-sh/uv).
- Tag filtering: use prefixed tags like `sdk-v0.1.0` to distinguish SDK releases from backend/dashboard releases. The publish workflow triggers only on `sdk-v*` tags.

**Pre-push clean-up checklist (for roadmapper to include as a task):**

1. `git log --all --full-history -- '*.env'` — any env file in history must be scrubbed (use `git filter-repo --path-glob '**/*.env' --invert-paths`) before the first push.
2. Scan for API keys / credentials / DB URLs: `rg -i 'sk-|api[_-]key|secret|password|bearer' --hidden`.
3. Check `infra/` for private Terraform state, `*.tfstate`, vendor credentials.
4. Check `dashboard/.env.local` and `backend/.env*` and similar.
5. Remove any `demo/` recordings or datasets with PII.
6. Add/update `.gitignore`.
7. Add a LICENSE (MIT or Apache-2.0 to match user's other public work).
8. Add a README explaining: (a) what's published to PyPI (the sdk/python subdir), (b) what's just open-source code to browse (backend/dashboard/demo/infra).

---

## Recommended repo-layer structures

### `zeroth-core/` structure

```
zeroth-core/
├── pyproject.toml                    # name = "zeroth-core", hatchling
├── uv.lock                           # locked dep tree
├── README.md                         # library intro + install + quickstart
├── LICENSE
├── CHANGELOG.md
├── CLAUDE.md                         # agent guidelines (scoped to core)
├── PROGRESS.md                       # core-scoped roadmap
├── Dockerfile                        # backend runtime image
├── docker-compose.yml                # backend stack (no nginx)
├── alembic.ini
├── mkdocs.yml                        # docs site config
├── .github/
│   └── workflows/
│       ├── ci.yml                    # lint + test + build on PR/push
│       ├── publish.yml               # PyPI publish on v* tags
│       └── docs.yml                  # mkdocs build + mike deploy
├── src/
│   └── zeroth/                       # PEP 420 namespace, NO __init__.py
│       └── core/                     # owned by this wheel
│           ├── __init__.py
│           ├── py.typed
│           ├── graph/
│           ├── orchestrator/
│           ├── contracts/
│           ├── runs/
│           ├── execution_units/
│           ├── agent_runtime/
│           ├── mappings/
│           ├── conditions/
│           ├── approvals/
│           ├── audit/
│           ├── policy/
│           ├── secrets/
│           ├── memory/
│           ├── guardrails/
│           ├── identity/
│           ├── config/
│           ├── deployments/
│           ├── dispatch/
│           ├── econ/
│           ├── migrations/           # alembic env.py + versions/
│           ├── observability/
│           ├── sandbox_sidecar/
│           ├── service/              # FastAPI app + entrypoint
│           ├── storage/
│           ├── studio/               # studio backend helpers (HTTP endpoints)
│           ├── webhooks/
│           └── demos/
├── tests/                            # pytest, imports zeroth.core.*
│   └── conftest.py
├── docs/                             # MkDocs source
│   ├── index.md
│   ├── getting-started/
│   ├── concepts/
│   ├── subsystems/                   # per-module guides
│   │   ├── graph.md
│   │   ├── orchestrator.md
│   │   └── ... (one per subsystem)
│   ├── api/                          # auto-generated by mkdocstrings
│   │   └── index.md
│   ├── http-api/                     # generated from OpenAPI
│   ├── recipes/
│   ├── deployment/
│   │   └── integrated.md             # core + studio together
│   └── migration/
│       └── from-monolith.md
├── scripts/
│   └── migrate-imports.py            # codemod for pre-split users
└── phases/                           # gsd phase artifacts (core-scoped)
```

### `zeroth-studio/` structure

```
zeroth-studio/
├── README.md                         # links to zeroth-core docs
├── LICENSE
├── CHANGELOG.md
├── package.json                      # Vue 3 + Vite + Vue Flow + Pinia
├── pnpm-lock.yaml (or package-lock.json)
├── vite.config.ts
├── tsconfig.json
├── index.html
├── Dockerfile                        # nginx:alpine serving dist/
├── PROGRESS.md                       # Studio phases 24-26 roadmap
├── CLAUDE.md
├── .github/
│   └── workflows/
│       ├── ci.yml                    # lint + type-check + unit + build
│       └── docker-publish.yml        # ghcr push on tag
├── nginx/
│   ├── studio.conf
│   └── certs/                        # dev certs only
├── src/                              # Vue source (current apps/studio/src)
├── public/
├── tests/
│   ├── unit/
│   └── e2e/
└── phases/                           # gsd phase artifacts (studio-scoped)
```

### `regulus/` structure (pushed as-is to rrrozhd/regulus)

```
regulus/
├── README.md                         # explains multi-component layout
├── LICENSE
├── backend/
├── dashboard/
├── demo/
├── docs/
├── infra/
├── sdk/
│   └── python/
│       ├── pyproject.toml            # name = "econ-instrumentation-sdk"
│       ├── src/...
│       └── tests/
└── .github/
    └── workflows/
        ├── ci.yml
        └── publish-sdk.yml           # triggers on tags "sdk-v*"
```

---

## Data flow — docs pipeline

```
┌───────────────┐       ┌─────────────────┐       ┌──────────────┐
│ zeroth.core.* │──┐    │ zeroth-core CI  │       │ gh-pages     │
│ source +      │  │    │  (GH Actions)   │       │ branch       │
│ docstrings    │  ├───►│                 │──────►│              │
└───────────────┘  │    │ mkdocs build    │       └──────┬───────┘
                   │    │ --strict        │              │
┌───────────────┐  │    │                 │              │
│ docs/*.md     │  │    │ mkdocstrings:   │              │
│ hand-written  │──┤    │  import zeroth  │       ┌──────▼───────┐
│ guides        │  │    │  .core, render  │       │   GH Pages   │
└───────────────┘  │    │  symbol pages   │       │   (hosted)   │
                   │    │                 │       │              │
┌───────────────┐  │    │ mike deploy     │       │  /latest/    │
│ OpenAPI spec  │──┘    │  version+alias  │       │  /dev/       │
│ (FastAPI gen) │       │                 │       │  /0.1.0/     │
└───────────────┘       └─────────────────┘       │  /0.2.0/     │
                                                  └──────────────┘
```

**Trigger matrix:**

| Event | Action | Target |
|---|---|---|
| Push to `main` | Build + publish | `/dev/` |
| Release tag `v0.1.0` | Build + publish + update `latest` alias | `/0.1.0/` + `/latest/` |
| Pull request | Build strict (no publish), fail on broken links | PR status check |

---

## Cross-repo release coordination

**Versioning decoupling:**
- `zeroth-core` uses SemVer on the Python lib surface (0.1.0, 0.1.1, 0.2.0…).
- `zeroth-studio` uses its own SemVer track independent of core (the frontend releases when the frontend ships).
- `econ-instrumentation-sdk` has its own track tagged `sdk-v*` in the regulus repo.

**Compatibility matrix:**
- zeroth-core documents the *minimum* zeroth-studio image tag compatible with each core release.
- zeroth-studio documents the *minimum* zeroth-core version its HTTP calls target.
- Both shipped images pin their counterpart by digest or tag in the reference `docker-compose.yml`.

**Breaking change protocol:**
- Any HTTP API change in core that affects Studio → opens a companion PR in `zeroth-studio` before the core release ships.
- Core releases that break studio are called out in `zeroth-core/CHANGELOG.md` with a pointer to the required studio version.
- Integration smoke test: a scheduled nightly GH Action in `zeroth-studio` pulls `zeroth-core:latest` image and runs a minimal E2E against it. Failure opens an issue.

---

## Architectural patterns to follow

### Pattern 1: namespace package as distribution boundary

**What:** Ship multiple wheels under a shared `zeroth` top-level namespace by never defining `zeroth/__init__.py` in any wheel.
**When:** Whenever you plan to split related Python packages across repos but want a unified import path.
**Trade-offs:** (+) clean separation of release cadence, clean ownership; (−) one wheel accidentally shipping `zeroth/__init__.py` silently breaks all siblings — requires CI enforcement.

### Pattern 2: documentation as code, colocated with source

**What:** `mkdocstrings` reads Python source at docs build time; guides live in `docs/` of the source repo.
**When:** Whenever the library is the primary consumer artifact.
**Trade-offs:** (+) docs always match code; (+) one place to edit both; (−) docs build depends on a working Python environment.

### Pattern 3: trusted publishing over API tokens

**What:** GH Actions authenticates to PyPI via OIDC, no stored tokens.
**When:** Any PyPI-publishing workflow post-2023.
**Trade-offs:** (+) no token rotation burden, no leak risk; (−) initial setup requires a PyPI account with Trusted Publishing configured per project, and works only from GH Actions.

### Pattern 4: tag-driven versioning via hatch-vcs

**What:** Wheel version derived from the latest git tag by hatch-vcs; no version string committed in source.
**When:** Any published Python package.
**Trade-offs:** (+) impossible to ship a mislabeled version, tags are immutable; (−) dev builds have non-canonical local versions like `0.1.0.dev3+g1234abc`.

### Pattern 5: HTTP contract as the only coupling point

**What:** Two repos share no source code, only an HTTP API (OpenAPI-documented). Optional TypeScript client is generated from the OpenAPI JSON.
**When:** Split frontend and backend live in separate repos.
**Trade-offs:** (+) independent release cadence, clear boundary; (−) requires discipline around compat matrix and integration smoke tests.

---

## Anti-patterns to avoid

### Anti-pattern 1: shim `__init__.py` at `src/zeroth/__init__.py`

**What people do:** Add `from zeroth.core import *` in `src/zeroth/__init__.py` to preserve backward-compatible imports.
**Why wrong:** Breaks PEP 420 namespace, blocks any future sibling package, and re-introduces the transitive import problem that motivated the split in the first place.
**Instead:** Clean break + migration guide + codemod script.

### Anti-pattern 2: monorepo uv workspace when the packages share no source

**What people do:** Put Vue frontend and Python library in one repo because "monorepos are cool."
**Why wrong:** The coupling point is an HTTP contract, not shared source. Monorepo overhead (path filters, per-package tagging, two language toolchains in one CI) is pure cost.
**Instead:** Two repos, coordinated via OpenAPI + compatibility matrix in docs.

### Anti-pattern 3: publishing a wheel with git URL deps

**What people do:** `pip install zeroth-core` only to find it transitively needs `governai @ git+...` and fails on air-gapped systems or on PyPI's direct-reference policy.
**Why wrong:** PyPI rejects direct references at upload time. Wheels built locally work, but nothing uploads.
**Instead:** Either publish the dep to PyPI first, or don't publish the consumer to PyPI.

### Anti-pattern 4: building docs on every push AND every tag with different configs

**What people do:** Two separate mkdocs configs for "dev" and "release" that drift.
**Why wrong:** Drift → dev works, release breaks (or vice versa), on release day when rollback is hardest.
**Instead:** One `mkdocs.yml`; `mike` handles versioning.

### Anti-pattern 5: flattening the regulus repo to only `sdk/python/`

**What people do:** Rewrite git history or cherry-pick only sdk/ to a new repo named `regulus-sdk`.
**Why wrong:** Throws away the rest of the project permanently; future open-sourcing of backend/dashboard becomes a new migration.
**Instead:** Push the multi-component repo as-is, publish only `sdk/python/` from CI.

### Anti-pattern 6: blocking docs release on 100% docstring coverage

**What people do:** Refuse to ship API reference until every module has a full docstring.
**Why wrong:** Multi-week tangent; mkdocstrings degrades gracefully so users get value from partial coverage.
**Instead:** Ship with gaps, track with `interrogate`, harden iteratively.

---

## Integration points

### External services / platforms

| Service | Integration | Notes |
|---|---|---|
| PyPI (`zeroth-core`) | Trusted Publishing OIDC from zeroth-core `publish.yml` | needs PyPI project pre-registered + TrustedPublisher config entry; **blocked by governai git-dep until resolved** |
| PyPI (`econ-instrumentation-sdk`) | Trusted Publishing from regulus `publish-sdk.yml` (tag filter `sdk-v*`) | must publish before zeroth-core can drop file:// URL dep |
| GH Pages (docs) | `gh-pages` branch on zeroth-core, managed by `mike` | enable Pages in repo settings, set source to `gh-pages` |
| GHCR (studio image) | `docker/build-push-action` from zeroth-studio on tag | public images for reference docker-compose |
| GHCR (core image, optional) | `docker/build-push-action` from zeroth-core on tag | optional — users can also build locally |
| TestPyPI | Pre-prod smoke test before real PyPI | both core and regulus CI |

### Internal boundaries

| Boundary | Communication | Notes |
|---|---|---|
| `zeroth-core` Python lib ↔ `zeroth-studio` Vue app | HTTP `/v1/*` API | OpenAPI spec is the contract; consider generating TS client from it |
| `zeroth-core` runtime ↔ `econ-instrumentation-sdk` (Regulus SDK) | In-process Python import | fail-open pattern already proven in v1.1 |
| `zeroth-core` runtime ↔ Regulus backend service | HTTP (companion container) | existing, unchanged |
| `zeroth-core` docs ↔ `zeroth-core` source | mkdocstrings at build time | requires `uv sync` of core deps in docs CI |
| `zeroth-studio` ↔ `zeroth-core` HTTP API version compat | Compatibility matrix in both READMEs + nightly smoke test | convention-based enforcement |

---

## Suggested build order (for gsd-roadmapper)

This is the **phase sequencing** the roadmap should honor. Items marked (DONE ad-hoc) reflect what PROJECT.md and the scratch `/tmp/zeroth-split/zeroth-core-build/` say is already partially complete.

| # | Phase | Depends on | Why this order |
|---|---|---|---|
| 0 | **Preservation / archive layers** (tarball + bare mirror + GH archive of `rrrozhd/zeroth-archive`) **(mostly DONE ad-hoc)** | — | Non-destructive safety net before anything else |
| 1 | **Core extraction / codemod** — rename `src/zeroth/*` → `src/zeroth/core/*`, rewrite imports, fix Dockerfile CMD, fix alembic.ini **(DONE ad-hoc at `/tmp/zeroth-split/zeroth-core-build/`; 661/669 tests collect, wheel builds, 130 packages resolve)** | 0 | Mechanical rename is a precondition for everything downstream |
| 2 | **Core repo creation + scratch-build promotion** — create `rrrozhd/zeroth-core`, push cleaned tree from the scratch build, investigate the 8 test collection failures, set up baseline CI (lint + test), add importlinter guard rail against any import of `zeroth.*` that isn't `zeroth.core.*` | 1 | Need a real GitHub repo before CI can run |
| 3 | **Regulus repo publication** — secret scan, cleanup, push `rrrozhd/regulus` as-is, stand up `publish-sdk.yml` workflow | 0 | Can run in parallel with 1–2; blocks 5 |
| 4 | **Regulus SDK → PyPI** — TestPyPI → smoke → PyPI (`econ-instrumentation-sdk 0.1.0`) under Trusted Publishing | 3 | Blocks core's PyPI publish |
| 5 | **Core dep swap** — zeroth-core pyproject.toml: `econ-instrumentation-sdk @ file://...` → `econ-instrumentation-sdk>=0.1.0` from PyPI, lock, CI green | 4, 2 | Clean PyPI dep required before core can publish to PyPI |
| 5a | **Governai decision gate (BLOCKING for PyPI publish)** — user-decision phase: (a) wait for upstream PyPI release, (b) publish a fork, (c) vendor modules, or (d) ship 0.1.0 as git-URL install only (not PyPI). See question 5. | 5 | **HARD BLOCK** on any PyPI publish of zeroth-core; the git-dep on governai is incompatible with PyPI upload |
| 6 | **Core publish (modality decided in 5a)** — either TestPyPI→PyPI (`zeroth-core 0.1.0`) with Trusted Publishing, or git-tag-based install documentation | 5a | Library is installable |
| 7 | **Docs foundation** — MkDocs Material + mkdocstrings + mike, `docs/` scaffold, API reference auto-gen, first 4–5 hand-written guides (getting started, concepts, graph, orchestrator, service), `docs.yml` workflow, publish `/dev/` + `/latest/` | 2 (written guides can parallel with 3–6; API reference needs 1) | Ship docs even with gaps |
| 8 | **Studio repo extraction** — create `rrrozhd/zeroth-studio`, move `apps/studio/` + nginx configs, set up CI, remove nginx service from zeroth-core's `docker-compose.yml`, write zeroth-studio README that links to zeroth-core docs | 1 | Can run in parallel with 3–7 once 1 is done |
| 9 | **Deployment glue** — reference `docker-compose.yml` for integrated deploy (pulls both images), deployment guide in `zeroth-core/docs/deployment/integrated.md`, optional core ghcr image | 6, 8 | Needs both images to exist |
| 10 | **Docstring hardening sub-phase** — interrogate baseline → raise threshold (80% → 95%) progressively, module-by-module gap fills | 7 | Iterative polish; not blocking |
| 11 | **In-depth subsystem guides** — full per-module documentation pass (all 27 subsystems), integration recipes, HTTP API reference auto-generated from OpenAPI, migration guide from monolithic | 7, 10 | Bulk of the "documentation" deliverable |
| 12 | **Integration smoke test + compatibility matrix** — nightly cross-repo test in zeroth-studio that pulls zeroth-core:latest and runs a minimal E2E; documented compatibility matrix in both READMEs | 6, 8, 9 | Prevents silent breakage as both repos evolve |

**Critical path:** 0 → 1 → 2 → 3 → 4 → 5 → 5a → 6 → 7 → 11. Phases 8, 9, 10, 12 run in parallel branches once their dependencies unblock.

**Already done ad-hoc (do NOT re-do, just promote):**
- Phase 0 preservation layers (per PROJECT.md: "Phase 0 (preservation) … have been completed ad-hoc")
- Phase 1 codemod at `/tmp/zeroth-split/zeroth-core-build/` — 661/669 tests collect, wheel builds, `uv sync` resolves 130 packages, string-literal monkeypatches and Dockerfile CMD and alembic.ini all updated. Roadmap task is to **promote** this scratch build into the real repo, not redo the codemod.

**Flags for roadmapper (must be surfaced as explicit roadmap items):**

1. **Phase 5a (governai) is a known unknown.** The current git-commit pin is incompatible with PyPI publishing. Roadmap MUST create an explicit user-decision gate with options (a)–(d) before any core publish attempt. Recommended default: option (d) ship 0.1.0 as git-install only, unblock PyPI later.
2. **Phase 2 promotion vs. redo.** The scratch build's 8 test collection failures (669 - 661) need investigation before promotion. Small task, not a new phase, but a required checkbox in the "promote scratch build" task.
3. **Docstring hardening is unbounded.** Roadmap should timebox it (e.g. target the current baseline for 0.1.0 docs; 80% interrogate for 0.2.0 docs; 95% for 1.0) rather than block on absolute coverage.
4. **Studio phases 24–26 are explicitly out of scope** for the v3.0 milestone per PROJECT.md — they move to zeroth-studio but as a **separate** roadmap. v3.0 for zeroth-studio ends at "repo exists, CI green, Dockerfile builds, README published, deployment guide in place."

---

## Sources

- `/Users/dondoe/coding/zeroth/.planning/PROJECT.md` (authoritative for milestone decisions)
- `/Users/dondoe/coding/zeroth/docs/superpowers/specs/2026-04-10-zeroth-core-platform-split-design.md` (historical; partially superseded by PROJECT.md pivot away from core/platform file split)
- `/Users/dondoe/coding/zeroth/docs/superpowers/plans/2026-04-10-zeroth-core-platform-split-plan.md` (preservation and filter-repo mechanics still valid; naming superseded)
- `/Users/dondoe/coding/zeroth/pyproject.toml` (existing dep graph, hatchling config, `allow-direct-references`)
- `/Users/dondoe/coding/zeroth/Dockerfile` (CMD + wheel path already updated in scratch build)
- `/Users/dondoe/coding/zeroth/docker-compose.yml` (nginx service to extract; backend stack to keep)
- Scratch build at `/tmp/zeroth-split/zeroth-core-build/` (proof-of-work: wheel builds, 661/669 tests collect, lock resolves 130 packages)
- PEP 420 (implicit namespace packages) — authoritative Python packaging spec
- PyPI Trusted Publishing: https://docs.pypi.org/trusted-publishers/
- mkdocstrings-python: https://mkdocstrings.github.io/python/
- mike (MkDocs versioning): https://github.com/jimporter/mike
- hatch-vcs: https://github.com/ofek/hatch-vcs
- interrogate (docstring coverage): https://interrogate.readthedocs.io/

---
*Architecture research for: v3.0 Core Library Extraction, Studio Split & Documentation*
*Researched: 2026-04-10*
