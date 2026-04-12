# Stack Research — v3.0 Core Library Extraction, Studio Split & Documentation

**Domain:** Python library packaging + technical documentation site + cross-repo frontend consumer
**Researched:** 2026-04-10
**Confidence:** HIGH (packaging + docs toolchain); MEDIUM (OpenAPI-in-docs integration pattern); HIGH (hosting)

> Scope note: this document ONLY lists what v3.0 *adds* or *changes*. The existing runtime stack (FastAPI, SQLAlchemy, Alembic, LiteLLM, Pydantic, ARQ, Redis, pgvector, Chroma, Elasticsearch, governai, econ-instrumentation-sdk, ruff, pytest, pytest-asyncio, uv, hatchling, Python 3.12+) is considered settled and NOT re-evaluated here. Supersedes the v2.0 Studio stack research.

---

## TL;DR — Additions for v3.0

1. **PyPI publishing:** `uv build` + `pypa/gh-action-pypi-publish@release/v1` via GitHub Actions with **OIDC trusted publishing**; TestPyPI first, PyPI on tag. No API tokens in secrets.
2. **Docs site:** **MkDocs Material** + **mkdocstrings[python]** (Griffe backend) + **mike** for versioning, published to **GitHub Pages**. Markdown-first, Python API reference auto-generated from docstrings via static AST parsing (no runtime imports needed).
3. **OpenAPI-in-docs:** **neoteroi-mkdocs** OpenAPI Docs plugin renders FastAPI's exported `openapi.json` into native MkDocs Material pages, plus a **ReDoc** static HTML page for the "fat reference" view.
4. **Docstring style:** **Google-style docstrings** (mkdocstrings-python default, best rendering, works with Griffe type extraction).
5. **Cross-repo consumer (`zeroth-studio`):** consume core via a release-asset-pinned `openapi.json` committed to `zeroth-studio`, feeding `openapi-typescript` (TS types) + `@hey-api/openapi-ts` (typed client). HTTP-only boundary, no npm SDK.
6. **Hatchling namespace config:** replace `packages = ["src/zeroth"]` with `sources = ["src"]` + `only-include = ["src/zeroth/core"]` to ship ONLY `zeroth/core/*` and keep the `zeroth` namespace PEP 420 implicit.
7. **Cleanup:** dedupe `psycopg` (keep `>=3.3`) and `litellm` (keep `>=1.83,<2.0`) in `pyproject.toml`.

---

## Recommended Stack

### Core Additions

| Technology | Version (Apr 2026) | Purpose | Why Recommended |
|---|---|---|---|
| **uv** | 0.11.x (already installed) | Build frontend / dep mgmt / `uv build` produces wheel + sdist | Already the project's package manager; `uv build` delegates to hatchling; no reason to switch. |
| **hatchling** | >=1.27 | PEP 517 build backend (already used) | Already the backend; supports PEP 420 namespace packages via `only-include`. Switching backends (e.g., to uv_build) is unnecessary churn and uv_build has known PEP 420 limitations (astral-sh/uv#14451). |
| **pypa/gh-action-pypi-publish** | `release/v1` (moving tag; pin by SHA for supply-chain hardening) | GitHub Actions step that uploads wheel+sdist to (Test)PyPI via OIDC | The "blessed" PyPA action. Built-in trusted-publishing support via `id-token: write`, generates Sigstore attestations by default, no long-lived tokens. |
| **PyPI Trusted Publishers (OIDC)** | PyPI feature | Tokenless publishing from GitHub Actions | Eliminates API token theft risk; short-lived OIDC tokens minted per workflow run. PyPI's official recommendation. |
| **MkDocs** | 1.6.x (pulled by Material) | Static site generator (markdown → HTML) | De-facto Python-ecosystem doc SSG. Simpler than Sphinx for Markdown-first workflows. |
| **mkdocs-material** | 9.7.6 (Mar 2026) | Theme, search, nav, admonitions, code highlighting, dark mode | Ubiquitous, accessible, fast, first-class offline search (lunr), excellent code blocks, supported by mkdocstrings. Free tier is sufficient; Insiders not required. |
| **mkdocstrings** | >=0.27 | MkDocs plugin that renders Python API reference inline | Native Material integration — lets guides and API ref cross-link via `[Graph][zeroth.core.graph.models.Graph]`. |
| **mkdocstrings-python** | 2.0.3 | Python handler for mkdocstrings (Griffe-backed) | Uses Griffe to parse source **statically** (no import side-effects — critical since our code imports FastAPI, psycopg, Redis, etc.). Supports Google / NumPy / Sphinx docstring styles. Renders type annotations with auto cross-references. |
| **Griffe** | 1.x (transitive) | Static Python AST parser for API extraction | Static parsing means mkdocstrings does NOT execute `import zeroth.core.service` during docs build → docs CI needs no Postgres/Redis/etc. Major advantage over Sphinx autodoc. |
| **mike** | >=2.1 | Versioned docs deployment to `gh-pages` branch | Standard for MkDocs versioning; Material has first-class support via `extra.version.provider: mike`. Each tagged release gets its own `/vX.Y/` URL; `latest` alias points at newest. |
| **neoteroi-mkdocs** | >=1.1 | MkDocs plugin that converts OpenAPI 3 JSON → Material-styled Markdown endpoint reference | Produces documentation that matches the rest of the site (fonts, search, admonitions, nav). Unlike an iframe'd Swagger UI, its output is crawlable and searchable by the MkDocs lunr index. |
| **@redocly/cli** (ReDoc) | 1.x / 2.x | Single-file "fat reference" HTML for OpenAPI schema | Complementary to neoteroi — ReDoc is the best dense single-page API browser. Generate once per release, drop at `/reference/http/redoc/index.html`. |
| **GitHub Pages** | — | Docs hosting | Free, unlimited, custom domain (CNAME), HTTPS via Let's Encrypt, native `mike`+`gh-pages` workflow. |

### Supporting Libraries (docs build-time only)

| Library | Version | Purpose | When to Use |
|---|---|---|---|
| **pymdown-extensions** | >=10 | Markdown extensions (tabbed code, admonitions, mermaid, keys) | Pulled in by mkdocs-material recommended config; enables "Python / HTTP / curl" tabs in examples. |
| **mkdocs-gen-files** | >=0.5 | Programmatic page generation during build | Generates per-module API reference pages by walking `src/zeroth/core/` — avoids hand-maintaining nav for every module. Standard mkdocstrings recipe. |
| **mkdocs-literate-nav** | >=0.6 | Sidebar nav defined in `SUMMARY.md` instead of `mkdocs.yml` | Lets `mkdocs-gen-files` emit `SUMMARY.md` with the generated module tree for a fully dynamic API nav. |
| **mkdocs-section-index** | >=0.3 | Attach `index.md` to section headers | Cleaner nav UX (a section header is also a landing page). |
| **mkdocs-macros-plugin** | >=1.3 (optional) | Jinja templating in Markdown | For injecting version numbers, env matrices, shared snippets into guides. |
| **import-linter** | >=2 | Enforce that `zeroth.core.*` does not import forbidden modules | Carried over from the split design; runs in CI as a lint step. Polices future drift even though v3.0 is a pure rename. |

### Cross-Repo Consumer (`zeroth-studio`) Tooling

| Library | Version | Purpose | Why |
|---|---|---|---|
| **openapi-typescript** | 7.x | Converts `openapi.json` → TypeScript `types.ts` (no runtime) | Zero-runtime type generation, actively maintained, clean `components['schemas']` types. Decouples Studio from a heavy SDK generator. |
| **@hey-api/openapi-ts** | 0.60.x | Generates a typed fetch client from OpenAPI | Modern successor to `openapi-typescript-codegen`. Tree-shakable, TS-first. Alternative: **orval** (more opinionated around TanStack Query). |
| **GitHub Release asset pattern** | — | `zeroth-core` CI publishes `openapi.json` as a release asset on each tag; `zeroth-studio` CI downloads it into `src/generated/` | Deterministic, offline-capable, no need to boot a live core during Studio CI. |

### Development Tools

| Tool | Purpose | Notes |
|---|---|---|
| **uv build** | `uv build` → `dist/*.whl` + `dist/*.tar.gz` | Replaces `python -m build`. Respects hatchling `[tool.hatch.build.targets.wheel]`. |
| **uv publish** | Optional manual publish | Prefer the GitHub Action for real publishes; `uv publish` is handy for TestPyPI smoke tests from a dev machine. |
| **twine check** | Validates README rendering on PyPI | `uvx twine check dist/*` before first publish. Catches broken RST/MD in `long_description`. |
| **pip install --index-url https://test.pypi.org/simple/** | TestPyPI verification | Verify the package installs cleanly from TestPyPI before cutting the real PyPI release. |

---

## Installation (new deps only)

```bash
# Docs toolchain — as a uv dep group, NOT runtime deps
uv add --group docs \
  "mkdocs-material>=9.7.6" \
  "mkdocstrings[python]>=0.27" \
  "mike>=2.1" \
  "mkdocs-gen-files>=0.5" \
  "mkdocs-literate-nav>=0.6" \
  "mkdocs-section-index>=0.3" \
  "pymdown-extensions>=10" \
  "neoteroi-mkdocs>=1.1"

# Boundary lint
uv add --group dev "import-linter>=2"

# ReDoc static bundle (invoked in CI via npx; NOT a Python dep)
npx --yes @redocly/cli@latest build-docs openapi.json -o site/reference/http/redoc/index.html
```

Add to `pyproject.toml`:

```toml
[dependency-groups]
docs = [
    "mkdocs-material>=9.7.6",
    "mkdocstrings[python]>=0.27",
    "mike>=2.1",
    "mkdocs-gen-files>=0.5",
    "mkdocs-literate-nav>=0.6",
    "mkdocs-section-index>=0.3",
    "pymdown-extensions>=10",
    "neoteroi-mkdocs>=1.1",
]
```

---

## Namespace Package Mechanics (the load-bearing section)

The project renames `src/zeroth/*` → `src/zeroth/core/*` with **no** top-level `src/zeroth/__init__.py`. This is a PEP 420 implicit namespace package. Hatchling supports this, but the current `packages = ["src/zeroth"]` line is **wrong** for this layout because it treats `zeroth` as a regular package and expects `__init__.py`.

### Correct hatchling config

```toml
[build-system]
requires = ["hatchling>=1.27"]
build-backend = "hatchling.build"

[project]
name = "zeroth-core"
version = "3.0.0"
# ... (other fields)

[tool.hatch.build.targets.wheel]
# DO NOT use packages = ["src/zeroth"] — that requires zeroth/__init__.py
sources = ["src"]
only-include = ["src/zeroth/core"]

[tool.hatch.build.targets.sdist]
only-include = ["src/zeroth/core", "README.md", "LICENSE", "pyproject.toml"]

[tool.hatch.metadata]
allow-direct-references = true  # keep ONLY during dev; must be removed before first publish
```

**Why this exact incantation:**

- `sources = ["src"]` tells hatchling to strip `src/` from wheel layout so the wheel contains `zeroth/core/...`, not `src/zeroth/core/...`.
- `only-include = ["src/zeroth/core"]` means the wheel contains EXACTLY `zeroth/core/*` and nothing else — no accidental `zeroth/__init__.py`, no leakage of `tests/`, `.planning/`, etc.
- Omitting any `packages = [...]` key avoids hatchling's implicit package detection (which looks for `__init__.py` and would fail).
- The resulting wheel ships directories as namespace portions. A future `zeroth-studio-py` or `zeroth-sdk` wheel can coexist in the same `zeroth/` namespace without collision.

### Editable install gotcha

`uv sync` / `pip install -e .` with hatchling uses `.pth`-based editable installs that work with PEP 420. No `__editable__` shims needed. However:

- **If a dev accidentally creates `src/zeroth/__init__.py`** (empty or otherwise), Python shadows the namespace package globally and breaks any sibling `zeroth.*` package.
- **Mitigation:** add a unit test that asserts `importlib.util.find_spec('zeroth').submodule_search_locations is not None and not hasattr(__import__('zeroth'), '__file__')`. Namespace packages have no `__file__`. Flag in PITFALLS.md.
- `ruff` and `pytest` work fine with `src/` as the source root. Add `pythonpath = ["src"]` under `[tool.pytest.ini_options]` only if test imports fail (shouldn't be needed with uv's editable install).

### Distribution name vs import name

- Distribution (PyPI) name: **`zeroth-core`** (hyphen, reserved per split design)
- Import name: **`zeroth.core`** (dot)
- `pip install zeroth-core` → `from zeroth.core import ...`

Normal (cf. `google-cloud-storage` → `google.cloud.storage`). Document loudly in README.

---

## PyPI Publishing Flow (Trusted Publishing / OIDC)

### One-time setup on (Test)PyPI

1. On **TestPyPI** (https://test.pypi.org/manage/account/publishing/) add a "pending publisher":
   - PyPI Project Name: `zeroth-core`
   - Owner: `rrrozhd`
   - Repository: `zeroth-core` (post-split) or current repo if publishing before split
   - Workflow filename: `publish.yml`
   - Environment name: `testpypi`
2. On **PyPI** (https://pypi.org/manage/account/publishing/) repeat with environment `pypi`.
3. In the GitHub repo, create two protected environments under Settings → Environments: `testpypi` (auto-deploy) and `pypi` (require manual approval).

No API tokens stored anywhere. OIDC mints short-lived tokens per workflow run.

### `.github/workflows/publish.yml`

```yaml
name: publish
on:
  push:
    tags: ["v*"]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv python install 3.12
      - run: uv build             # produces dist/*.whl and dist/*.tar.gz
      - run: uvx twine check dist/*
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist/ }

  testpypi:
    needs: build
    runs-on: ubuntu-latest
    environment: testpypi
    permissions:
      id-token: write            # OIDC — the magic permission
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/

  pypi:
    needs: testpypi
    runs-on: ubuntu-latest
    environment: pypi            # requires manual approval
    permissions:
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: pypa/gh-action-pypi-publish@release/v1
        # default repository-url = pypi.org; no username/password = OIDC
```

### Gotchas (CRITICAL)

- **`allow-direct-references = true`** lets `governai @ git+https://...` stay in `dependencies` LOCALLY. **PyPI REJECTS direct-reference URLs** (`git+`, `file:`) in uploaded distribution metadata. Before first publish, EITHER:
  - Publish governai upstream to PyPI (out of scope), OR
  - Move `governai` to `[project.optional-dependencies].governai` and document a manual install step, OR
  - Remove from `dependencies` entirely and require users to pip-install it themselves from git.

  **This is the single biggest publishing blocker for `zeroth-core`. Must be resolved in an early phase.**

- **`econ-instrumentation-sdk @ file:///...`** — same problem. Solution is this milestone's other deliverable: publish `econ-instrumentation-sdk` to PyPI first (TestPyPI → PyPI), then switch to a version spec `econ-instrumentation-sdk>=X.Y`. The Regulus SDK uses setuptools; its own publish workflow is nearly identical (different `build-backend` but same OIDC flow).

- **Duplicate deps in current `pyproject.toml`:**
  - `litellm>=1.83,<2.0` (line 12) vs `litellm>=1.83.0` (line 29) → **keep the bounded one (`>=1.83,<2.0`)**, delete the duplicate.
  - `psycopg[binary]>=3.3` (line 16) vs `psycopg[binary]>=3.1` (line 30) → **keep `>=3.3`**, delete the duplicate.
  - `uv` currently resolves these (bounded wins), but strict resolvers and `twine check` may warn. Clean up unconditionally as part of Phase 1.

- **Version bumping:** `version = "0.1.0"` today → `"3.0.0"` at release. Use a single source of truth: either keep it in `pyproject.toml` and bump manually, or use `hatch-vcs` to derive from git tags (optional; adds a plugin dep).

---

## Documentation Site Architecture

```
docs/
├── index.md                          # landing
├── getting-started/
│   ├── install.md
│   ├── first-graph.md
│   └── running-in-process.md
├── concepts/
│   ├── graphs.md
│   ├── contracts.md
│   ├── orchestration.md
│   ├── governance.md
│   └── economics.md
├── guides/
│   ├── graph-authoring.md
│   ├── memory-backends.md
│   ├── approvals.md
│   ├── secrets.md
│   ├── dispatch.md
│   └── ...                           # one per subsystem
├── integrations/
│   ├── litellm-providers.md
│   ├── governai.md
│   └── regulus.md
├── deployment/
│   ├── docker.md
│   ├── postgres.md
│   └── redis-arq.md
├── migration/
│   └── from-pre-v3.md                # the zeroth → zeroth.core rename guide
├── reference/
│   ├── python/                       # GENERATED by mkdocs-gen-files
│   │   └── ...                       # one page per zeroth.core.* module
│   └── http/
│       ├── index.md                  # neoteroi-rendered endpoints
│       └── redoc.md                  # link to /reference/http/redoc/
└── gen_ref_pages.py                  # mkdocs-gen-files script
```

### `mkdocs.yml` skeleton

```yaml
site_name: Zeroth Core
site_url: https://rrrozhd.github.io/zeroth-core/
repo_url: https://github.com/rrrozhd/zeroth-core
theme:
  name: material
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.indexes
    - content.code.copy
    - content.code.annotate
    - search.highlight
    - search.share
    - toc.follow
  palette:
    - media: "(prefers-color-scheme: light)"
      scheme: default
      toggle: { icon: material/brightness-7, name: Dark mode }
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      toggle: { icon: material/brightness-4, name: Light mode }

plugins:
  - search
  - gen-files:
      scripts: [docs/gen_ref_pages.py]
  - literate-nav:
      nav_file: SUMMARY.md
  - section-index
  - mkdocstrings:
      default_handler: python
      handlers:
        python:
          paths: [src]
          options:
            docstring_style: google
            show_source: true
            show_root_heading: true
            show_signature_annotations: true
            separate_signature: true
            merge_init_into_class: true
            docstring_section_style: table
  - neoteroi.mkdocsoad:
      use_pymdownx: true
  - mike:
      alias_type: symlink
      canonical_version: latest

markdown_extensions:
  - admonition
  - pymdownx.details
  - pymdownx.superfences
  - pymdownx.tabbed: { alternate_style: true }
  - pymdownx.highlight: { anchor_linenums: true }
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - attr_list
  - md_in_html
  - toc: { permalink: true }

extra:
  version:
    provider: mike
    default: latest
```

### Docstring style decision: **Google**

- Default for mkdocstrings-python; renders most richly of the three supported styles.
- Easier to read in source than NumPy's underlined sections.
- Griffe extracts type annotations from function signatures separately, so docstrings don't duplicate types.
- Enforce with `ruff`'s `D` (pydocstyle) rules: `[tool.ruff.lint.pydocstyle] convention = "google"` — but roll out gradually per module to avoid a 280-test, 22K-LOC docstring storm.

### Per-module API reference auto-gen

`docs/gen_ref_pages.py` walks `src/zeroth/core/` and emits one `reference/python/<module>.md` per public module containing just:

```
::: zeroth.core.graph.models
```

mkdocstrings + Griffe do the rest. This is the canonical mkdocstrings recipe — see https://mkdocstrings.github.io/recipes/.

### OpenAPI integration: two-track strategy

FastAPI already exposes `/openapi.json` at runtime. We bake it into the docs pipeline:

1. **CI step:** during docs build, a short Python script imports the FastAPI app (with a dev config — SQLite backend, no Redis) and writes `docs/_generated/openapi.json`. This IS an `import fastapi` but that's fine — only the docs build needs it, not mkdocstrings' Python handler (which stays static).
2. **neoteroi-mkdocs** renders that JSON into `docs/reference/http/index.md` as Material-styled endpoint docs (searchable, themeable, deep-linkable).
3. **ReDoc** is built as a separate static HTML page via `npx @redocly/cli build-docs` and dropped at `site/reference/http/redoc/index.html` as an "all endpoints on one page" reference.

**Why both:** neoteroi output integrates with site search & theme; ReDoc is denser and better for quick endpoint lookup. One extra CI step — cheap.

**Why not Swagger UI?** Interactive "try it" is useless for a library reference site (no live backend), adds a heavy JS bundle, and its output is not searchable by lunr.

---

## Docs Hosting Decision: **GitHub Pages**

| Criterion | GitHub Pages | Read the Docs | Netlify/Vercel/Cloudflare Pages |
|---|---|---|---|
| Cost | Free | Free (Community) / $50+/mo (Business) | Free tier; build minutes capped |
| Custom domain + HTTPS | ✅ (CNAME + LE auto) | ✅ | ✅ |
| Versioning | ✅ via `mike` | ✅ native | Manual only |
| Build environment control | Full (GitHub Actions) | Constrained (RTD-managed) | Full |
| Search | lunr (client) | Algolia-backed (better) | lunr (client) |
| Works with `mike` | ✅ (designed for it) | ⚠️ awkward | ✅ but duplicates host's versioning |
| PR preview deploys | GH Pages preview environments (basic) | ✅ | ✅ (best feature) |
| Analytics | Via plugin | Built-in | Built-in |

**Choice: GitHub Pages.** `mike` is purpose-built for gh-pages, GitHub Actions gives full build control (can run Python scripts to export OpenAPI), custom domain is trivial, free forever for public repos. RTD's superior search is the only meaningful tradeoff and Material's built-in search is already very good.

**If** we later hit lunr's limits (>5K pages) → migrate search to **Algolia DocSearch** (free for OSS) without changing host.
**If** PR preview deploys become important → migrate to **Cloudflare Pages** (free preview per PR), keep `mike` for versioning.

### Deployment workflow (`.github/workflows/docs.yml`)

```yaml
name: docs
on:
  push:
    branches: [main]
    tags: ["v*"]
permissions:
  contents: write       # mike pushes to gh-pages
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }            # mike needs history
      - uses: astral-sh/setup-uv@v5
      - run: uv python install 3.12
      - run: uv sync --group docs
      - name: Export OpenAPI
        run: uv run python scripts/export_openapi.py > docs/_generated/openapi.json
      - name: Build ReDoc bundle
        run: |
          mkdir -p site-extras/reference/http/redoc
          npx --yes @redocly/cli@latest build-docs \
            docs/_generated/openapi.json \
            -o site-extras/reference/http/redoc/index.html
      - name: Deploy dev docs
        if: github.ref == 'refs/heads/main'
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          uv run mike deploy --push --update-aliases dev
      - name: Deploy tagged version
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          git config user.name github-actions
          git config user.email github-actions@github.com
          VERSION=${GITHUB_REF_NAME#v}
          uv run mike deploy --push --update-aliases "$VERSION" latest
          uv run mike set-default --push latest
```

---

## Cross-Repo Dependency: `zeroth-studio` → `zeroth-core`

Studio is a Vue 3 frontend; its only dep on core is the HTTP API surface. **Do NOT** try to publish an npm SDK of core — the duplication is not worth it.

### Recommended pattern: release-pinned OpenAPI schema

1. On every tagged `zeroth-core` release, the `publish.yml` workflow uploads `openapi.json` as a **GitHub Release asset** alongside the wheel.
2. `zeroth-studio` has `openapi.json` checked in at `src/generated/openapi.json` plus a version pin in `package.json`:
   ```json
   "zerothCoreVersion": "3.0.0"
   ```
3. A scheduled GitHub Action in Studio opens a "refresh openapi" PR when a new core release is detected:
   ```bash
   gh release download "v${ZEROTH_CORE_VERSION}" \
     -R rrrozhd/zeroth-core -p openapi.json -O src/generated/openapi.json
   npx openapi-typescript src/generated/openapi.json -o src/generated/api-types.ts
   npx @hey-api/openapi-ts -i src/generated/openapi.json -o src/generated/client
   ```
4. Studio's TypeScript build fails deterministically if core removes/renames an endpoint — caught in PR, not at runtime.

**Why not "fetch at runtime from a live core":** CI becomes non-reproducible, offline dev breaks, rollback coupling.

**Why not "generate from source via code-gen":** requires running Python in the Studio Node CI image. Release-asset pattern is simpler and versioned.

**Library choices:**

- `openapi-typescript` (types only, zero runtime) — **primary choice**
- `@hey-api/openapi-ts` — modern full client generator, TS-first
- `orval` — alternative if Studio commits to TanStack Query (more opinionated)
- `openapi-generator-cli` — **avoid** (Java dep, bloated output, slow)

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|---|---|---|
| MkDocs Material + mkdocstrings | **Sphinx + autodoc + Furo** | You need reStructuredText, LaTeX/PDF output, or academic cross-referencing (numpydoc, napoleon). Sphinx is still king for scientific Python (NumPy, SciPy, Astropy). For an application library with narrative guides, MkDocs Material is lighter and more readable. Critically, Sphinx autodoc **imports** code to extract docs → docs CI would need Postgres, Redis, etc. Griffe's static parsing is a major win. |
| MkDocs Material | **Docusaurus** | You want React/MDX components in docs, or you're a JavaScript shop. For a Python library, Docusaurus means Node tooling on top of Python tooling and no native Python API extraction. Skip. |
| MkDocs Material | **pdoc** | You want the absolute minimum: one command → HTML API ref, no guides. pdoc is great for tiny libraries but has no guides, search, versioning, or theming. We need all four. |
| hatchling | **uv_build** (astral-sh/uv native backend) | Once uv_build fully supports PEP 420 implicit namespace packages (tracked in astral-sh/uv#14451) AND is battle-tested, migrate for a single-tool story. Today it's premature. |
| hatchling | **setuptools** | You're publishing a project that predates pyproject.toml with complex `MANIFEST.in`. Not our case. Setuptools' namespace package story is also more awkward. |
| pypa/gh-action-pypi-publish (OIDC) | **API token in secrets + twine upload** | You're publishing from a CI that doesn't support OIDC to PyPI (GitLab, self-hosted Jenkins). On GitHub Actions, OIDC is strictly better. |
| GitHub Pages + mike | **Read the Docs** | You need RTD's Algolia search, their analytics, or your docs team already knows the RTD workflow. Costs $50+/mo for Business; build env is constrained. |
| GitHub Pages + mike | **Cloudflare Pages / Netlify / Vercel** | You need deploy previews per PR. Cloudflare is most "free forever" of the three. Revisit if we hit GH Pages bandwidth limits. |
| neoteroi-mkdocs | **mkdocs-render-swagger-plugin** | You specifically want interactive Swagger UI embedded. Swagger UI is a giant iframe that breaks Material search and theme — neoteroi produces native MkDocs pages. |
| Release-asset OpenAPI pinning | **Fetch from `/openapi.json` of a deployed staging core** | Studio needs real-time schema updates during rapid cross-repo dev. Exception, not norm. |
| openapi-typescript | **openapi-generator-cli (Java)** | Only if you need a generator targeting a non-JS language. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|---|---|---|
| **Long-lived PyPI API tokens in GitHub Secrets** | Token theft is the #1 PyPI supply-chain attack vector. If leaked, attackers publish malicious versions under your name. | **Trusted Publishing (OIDC).** Short-lived, scoped per workflow run, auto-revoked. |
| **`__init__.py` in `src/zeroth/`** | Breaks PEP 420 namespace semantics; future `zeroth.studio` / `zeroth.sdk` packages cannot coexist. | Leave `src/zeroth/` as a bare directory. Add a test that asserts the namespace nature of `zeroth`. |
| **`packages = ["src/zeroth"]` in hatchling** (current config) | Hatchling looks for `zeroth/__init__.py` and breaks when it's absent, OR ships unwanted files. | `sources = ["src"]` + `only-include = ["src/zeroth/core"]`. |
| **Direct references (`git+`, `file:`) in published `[project].dependencies`** | PyPI rejects them on upload. | Move to `[project.optional-dependencies]`, OR publish upstream to PyPI, OR require manual user install. |
| **Sphinx `autodoc`** (for this project) | Imports target modules during build. Our modules pull FastAPI, SQLAlchemy, psycopg, Redis — docs CI becomes a full runtime env. | **mkdocstrings + Griffe** — static AST parsing, no imports, no runtime deps needed during docs build. |
| **Swagger UI embedded via iframe** | Heavy JS bundle, breaks lunr search indexing, doesn't match site theme, slow on mobile. | **neoteroi-mkdocs** (searchable native pages) + **ReDoc** static HTML. |
| **Publishing `zeroth-studio` as an npm consumer of a hand-written `@zeroth/sdk` package** | Duplicates HTTP contracts in two languages; drift is guaranteed; maintenance burden. | **OpenAPI pinning** via release asset + type generation. Single source of truth. |
| **`openapi-generator-cli`** (Java OpenAPI generator) | Java dep in Node/TS CI, heavy output, slow, awkward TS idioms. | `openapi-typescript` (types) + `@hey-api/openapi-ts` (client). |
| **Duplicate dependency entries in `pyproject.toml`** (`litellm` ×2, `psycopg` ×2) | Confusing; trips strict resolvers and `twine check`. | Dedupe. Keep tighter bounds: `litellm>=1.83,<2.0`, `psycopg[binary]>=3.3`. |
| **`allow-direct-references = true`** AT PUBLISH TIME | PyPI will reject the upload. | Remove from `pyproject.toml` before first publish by eliminating all direct-reference deps. |

---

## Stack Patterns by Variant

**If `governai` cannot be published to PyPI in time for v3.0.0:**
- Move `governai @ git+...` from `[project].dependencies` to `[project.optional-dependencies].governai`.
- Document in install guide: `pip install zeroth-core && pip install 'git+https://github.com/rrrozhd/governai.git@...'`.
- Acceptable for v3.0.0; flag as tech debt.

**If docs need per-PR preview deploys:**
- Switch from GitHub Pages to **Cloudflare Pages** (free, preview URL per PR).
- Keep `mike` for semver versioning — CF Pages handles branch previews, `mike` handles releases, they compose.

**If lunr search becomes too slow:**
- Add **Algolia DocSearch** (free for OSS) as a Material search backend. No migration otherwise.

**If we ever need PDF or man-page output:**
- Add `mkdocs-with-pdf` plugin, or accept the migration cost to Sphinx. Defer indefinitely.

**If we want OpenAPI types in Python (core or another Python consumer):**
- Add `datamodel-code-generator` (Pydantic-v2 compatible) for `openapi.json` → pydantic models in a sibling repo.

---

## Version Compatibility

| Package A | Compatible With | Notes |
|---|---|---|
| `mkdocs-material>=9.7` | `mkdocs>=1.6,<2.0` | MkDocs 2.0 is in alpha (Feb 2026); Material will track it but pin to 1.6.x for now. |
| `mkdocstrings[python]>=0.27` | `mkdocs>=1.6`, `griffe>=1.0` | Griffe 1.x is the stable line. |
| `mike>=2.1` | `mkdocs>=1.4` | Works with `extra.version.provider: mike`. |
| `neoteroi-mkdocs>=1.1` | `mkdocs-material>=9.0`, `pymdown-extensions>=10` | Depends on Material's admonition & tabbed blocks. |
| `hatchling>=1.27` | `uv>=0.5`, Python `>=3.8` | `only-include` / PEP 420 support stable since ~1.17. |
| `pypa/gh-action-pypi-publish@release/v1` | PyPI Trusted Publishers | Pin with `release/v1` OR a SHA for supply-chain hardening. |
| `openapi-typescript@7` | TypeScript `>=5`, Node `>=18` | Handles OpenAPI 3.0 and 3.1 (FastAPI emits 3.1 since 0.110). |

---

## Confidence & Risk Register

| Area | Confidence | Risk / Unknown |
|---|---|---|
| PyPI OIDC trusted publishing | HIGH | Battle-tested. Main risk: `governai` / `econ-instrumentation-sdk` direct-reference blocker (known, flagged). |
| MkDocs Material + mkdocstrings | HIGH | Industry standard for Python library docs. |
| Hatchling `only-include` for PEP 420 | HIGH | Confirmed in hatch docs. Pitfall: existing `packages = ["src/zeroth"]` line must be replaced. |
| neoteroi-mkdocs for OpenAPI rendering | MEDIUM | Less popular than Swagger UI; if it breaks on FastAPI's OpenAPI 3.1 output, fallbacks are `mkdocs-render-swagger-plugin` (with search tradeoff) or a pure ReDoc-only strategy. Verify early in the docs phase. |
| OpenAPI pinning pattern for Studio | HIGH | Standard pattern; risk is forgetting to bump the pinned version, mitigated by scheduled GH Action. |
| GitHub Pages hosting | HIGH | No concerns. |
| Regulus SDK publishing (setuptools → PyPI) | MEDIUM | SDK is currently setuptools-based at `/Users/dondoe/coding/regulus/sdk/python/`; publish flow mirrors zeroth-core but uses setuptools build. Needs its own OIDC publisher config. |

---

## Open Questions for Roadmap / Later Phases

1. **Who owns the `zeroth-core` PyPI project registration?** Reserved per design doc but not yet claimed — must be claimed before configuring trusted publisher.
2. **Version scheme:** SemVer? CalVer? Recommend SemVer (`3.0.0`, `3.0.1`, ...) matching the milestone number.
3. **`governai` resolution timeline:** is upstream likely to publish v0.3.0 to PyPI before v3.0.0 ships, or do we accept the optional-dependency workaround?
4. **Docs domain:** `docs.zeroth.dev`? `zeroth-core.readthedocs.io`? `rrrozhd.github.io/zeroth-core`? Decide before CNAME config.
5. **Docstring backfill strategy:** 22K LOC exists without Google-style docstrings. Phase it: (a) enforce style on new code via ruff; (b) backfill public API modules first; (c) leave internals for later. Don't block v3.0.0 on 100% docstring coverage.

---

## Sources

- [pypa/gh-action-pypi-publish (GitHub)](https://github.com/pypa/gh-action-pypi-publish) — OIDC trusted publishing + Sigstore attestations
- [PyPI Trusted Publishers — Using a Publisher](https://docs.pypi.org/trusted-publishers/using-a-publisher/) — official setup flow
- [GitHub Docs — Configuring OpenID Connect in PyPI](https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-pypi) — `id-token: write` permission requirement
- [Hatch build configuration](https://hatch.pypa.io/1.13/config/build/) — `sources`, `only-include`, `force-include` semantics
- [pypa/hatch discussion #819 — namespace packages](https://github.com/pypa/hatch/discussions/819) — confirmed PEP 420 recipe
- [PEP 420 — Implicit Namespace Packages](https://peps.python.org/pep-0420/)
- [Python Packaging User Guide — namespace packages](https://packaging.python.org/en/latest/guides/packaging-namespace-packages/)
- [astral-sh/uv#14451 — uv_build PEP 420 limitation](https://github.com/astral-sh/uv/issues/14451) — rationale to stay on hatchling
- [mkdocs-material PyPI](https://pypi.org/project/mkdocs-material/) — 9.7.6 (Mar 2026) confirmed
- [mkdocstrings-python PyPI](https://pypi.org/project/mkdocstrings-python/) — 2.0.3 confirmed
- [mkdocstrings Python handler usage](https://mkdocstrings.github.io/python/usage/) — Google/NumPy/Sphinx docstring support, Griffe-backed static parsing
- [Material for MkDocs — Setting up versioning](https://squidfunk.github.io/mkdocs-material/setup/setting-up-versioning/) — mike integration
- [jimporter/mike (GitHub)](https://github.com/jimporter/mike) — versioning mechanics
- [Neoteroi MkDocs OpenAPI Docs plugin](https://www.neoteroi.dev/mkdocs-plugins/web/oad/) — FastAPI OpenAPI → Material pages
- [bharel/mkdocs-render-swagger-plugin (GitHub)](https://github.com/bharel/mkdocs-render-swagger-plugin) — fallback option
- [FastAPI Metadata and Docs URLs](https://fastapi.tiangolo.com/tutorial/metadata/) — OpenAPI export semantics
- [Redocly CLI — build-docs](https://redocly.com/docs/cli/commands/build-docs/) — static ReDoc bundle
- [openapi-typescript (GitHub)](https://github.com/drwpow/openapi-typescript) — TS type generation
- [@hey-api/openapi-ts](https://heyapi.dev/) — modern TS client generator

---

*Stack research for: v3.0 Core Library Extraction, Studio Split & Documentation*
*Researched: 2026-04-10*
*Supersedes: v2.0 Studio stack research (2026-04-09)*
