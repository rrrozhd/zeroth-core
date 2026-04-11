# Phase 28: PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`) - Research

**Researched:** 2026-04-11
**Domain:** Python packaging, PyPI trusted-publisher release automation, hatchling build backend, optional extras
**Confidence:** HIGH

## Summary

Phase 28 moves `zeroth-core` from a one-off manual PyPI upload (already done) to a reproducible, OIDC-trusted, TestPyPI-staged release pipeline. The CONTEXT.md is exceptionally complete — 25 locked decisions cover nearly every planning question. Research surfaces **three critical corrections** to those decisions that the planner MUST address before writing tasks:

1. **`zeroth-core==0.1.0` is ALREADY live on PyPI** (uploaded 2026-04-10 via manual build). PyPI versions are immutable, so D-05 ("First PyPI version: `0.1.0`, no bump") is impossible — the next release MUST be `0.1.1` or higher. [VERIFIED: https://pypi.org/pypi/zeroth-core/json — `latest: 0.1.0`, upload_time 2026-04-10T22:29:08]
2. **The existing published wheel already has the correct `zeroth/core/` layout** (verified by `unzip -l dist/zeroth_core-0.1.0-py3-none-any.whl`). D-21's "fix the hatchling wheel target" is not fixing a bug — `packages = ["src/zeroth"]` combined with the absence of `src/zeroth/__init__.py` already produces the desired namespace-package wheel. Changing to `packages = ["src/zeroth/core"]` is an *optional refinement* and must be tested because hatchling uses the last path segment as the package name — a naive change could produce a wheel with files rooted at `core/` instead of `zeroth/core/`. [VERIFIED: wheel listing in this session]
3. **`[sandbox]` extra has no runtime dependencies.** `zeroth.core.sandbox_sidecar` imports only `fastapi` (already in base), `pydantic` (already in base), stdlib (`asyncio`, `logging`), and shells out to the `docker` CLI via `asyncio.create_subprocess_exec`. There is no Python `docker` library import. D-03's "planner discovers exact deps" resolves to: `[sandbox]` is either an **empty marker extra** or should be **dropped entirely**. Planner should confirm with user or make the extra empty `= []` so `pip install "zeroth-core[sandbox]"` succeeds while remaining a no-op. [VERIFIED: grep of `src/zeroth/core/sandbox_sidecar/*.py` in this session]

**Primary recommendation:** Before writing any task, planner MUST reconcile the three findings above with the user (or mark them as Claude's Discretion resolutions in the plan's preamble). The rest of CONTEXT.md is execution-ready.

## User Constraints (from CONTEXT.md)

### Locked Decisions

All 25 decisions (D-01 through D-25) from CONTEXT.md are treated as locked. Key ones:

- **D-01** Base `dependencies` is minimal core only; backend-specific (`psycopg`, `pgvector`, `chromadb-client`, `elasticsearch`, `redis`, `arq`) move into extras.
- **D-02** Extra names LOCKED verbatim: `[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]`. No renaming.
- **D-03** Extra contents (per context): `[memory-pg]` → psycopg[binary]>=3.3, psycopg-pool>=3.2, pgvector>=0.4.2; `[memory-chroma]` → chromadb-client>=1.5.6; `[memory-es]` → elasticsearch[async]>=8.0,<9; `[dispatch]` → redis>=5.0.0, arq>=0.27; `[sandbox]` → planner discovers (**see finding #3 above**); `[all]` → union of the above.
- **D-04** Verification gate per extra: CI job creates a clean venv, runs `pip install "zeroth-core[<extra>]"`, imports the modules that depend on it.
- **D-05** First PyPI version: `0.1.0` — **IMPOSSIBLE, see finding #1 above.** Planner MUST resolve (recommendation: bump to `0.1.1`).
- **D-06** SemVer.
- **D-07** Version source: static `[project].version` in pyproject.toml; release workflow asserts git tag matches before publish.
- **D-08** Release trigger: GitHub Release `published` event.
- **D-09** TestPyPI dry-run is mandatory in every release.
- **D-10** Separate workflows per package; Phase 28 owns `.github/workflows/release-zeroth-core.yml` only.
- **D-11** GitHub environment: `pypi`; TestPyPI env is Claude's discretion based on trusted-publisher requirements (**research shows: separate `testpypi` env is the documented pattern — see Trusted Publishers section below**).
- **D-12** Publish stages: build → smoke-install → tests → publish-testpypi → smoke-install-from-testpypi → publish-pypi.
- **D-13** Sigstore attestations enabled; `attestations: true`, `id-token: write`.
- **D-14** OIDC permissions job-scoped, not workflow-scoped.
- **D-15** License: Apache-2.0.
- **D-16** CHANGELOG.md in keepachangelog 1.1.0 format, seeded with single `[0.1.0]` / `[0.1.1]` entry.
- **D-17** CONTRIBUTING.md ~1 page; dev setup, PR conventions, issues, LICENSE link.
- **D-18** Ship `examples/hello.py` in this repo; CI smoke-install step executes it.
- **D-19** `hello.py` uses real LLM, env-gated; recommendation: `ANTHROPIC_API_KEY`.
- **D-20** `examples/` committed; Phase 28 adds only `hello.py`.
- **D-21** Fix hatchling wheel target — **see finding #2 above.** The "fix" is cosmetic, not a bug fix; planner must verify any change preserves the `zeroth/core/` prefix in the wheel.
- **D-22** Add `[project.urls]` (Homepage, Source, Issues, Changelog).
- **D-23** Add description (already set), keywords, classifiers.
- **D-24** Verify README renders on PyPI (already does — current version published).
- **D-25** Update `.planning/STATE.md` blockers: remove stale Regulus blockers.

### Claude's Discretion (from CONTEXT.md)

- Exact `[sandbox]` extra contents — **resolved by research: empty**.
- TestPyPI env name (`pypi` vs separate `testpypi`) — **resolved by research: separate `testpypi` env required by PyPI trusted-publisher docs** (see Trusted Publishers section).
- Exact classifiers and keywords for PyPI metadata.
- `py.typed` marker file under `src/zeroth/core/` (PEP 561) — recommended yes.
- Default LLM provider in `hello.py` — recommended `ANTHROPIC_API_KEY`.
- `uv build` direct vs `pypa/build` action — recommended `uv build` for toolchain consistency.
- Reserve `zeroth-core` name on PyPI in advance — **already reserved / already published** per finding #1.

### Deferred Ideas (OUT OF SCOPE)

All deferred items from CONTEXT.md remain out of scope: zeroth-studio split (Phase 29), docs site (Phases 30–32), code-of-conduct, governance, RFC process, issue templates, SBOM, hatch-vcs, multi-Python matrix, ARM64/musl wheels, econ-sdk Regulus-side work, README rewrite, PyPI badges.

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PKG-01 | econ-instrumentation-sdk on PyPI, zeroth-core depends on it via PyPI constraint | **Already satisfied.** econ-instrumentation-sdk==0.1.1 live on PyPI [VERIFIED: pypi.org]; pyproject.toml already pins `econ-instrumentation-sdk>=0.1.1` (commit 78c2076). Phase 28 task here is verification only. |
| PKG-02 | zeroth-core published to PyPI, clean-venv install works | **Partially satisfied** — first release (0.1.0) already uploaded manually. Phase 28 task is reproducing the publish via trusted-publisher workflow for version 0.1.1+. |
| PKG-03 | Extras declared, each installable | Research confirms exact version pins for all extras (see Standard Stack table). `[sandbox]` resolves to empty. `[all]` is union. |
| PKG-04 | LICENSE, CHANGELOG.md, CONTRIBUTING.md at root | Research confirms Apache-2.0 canonical text URL, keepachangelog 1.1.0 section spec. |
| PKG-05 | GitHub Actions trusted publisher, no long-lived tokens | Research confirms pypa/gh-action-pypi-publish v1.14.0 latest, separate `testpypi` env pattern, required OIDC permissions, attestations default-on. |
| PKG-06 | Clean-venv pip install + hello example works end-to-end | Research confirms pattern: ship `examples/hello.py`, CI step env-gated on LLM API key. |

## Standard Stack

### Core Release Toolchain

| Tool | Version | Purpose | Why Standard | Provenance |
|------|---------|---------|--------------|------------|
| `hatchling` | 1.29.0 | Build backend (sdist + wheel) | Already configured; PEP 517/518 compliant; handles namespace packages | [VERIFIED: `curl pypi.org/pypi/hatchling/json` — 1.29.0] |
| `uv` | 0.11.3 | Dependency management + build driver (`uv build`) | Already in project; fastest PEP 517 build front-end; aligns with local toolchain | [VERIFIED: `uv --version` on this machine] |
| `pypa/gh-action-pypi-publish` | **v1.14.0** (or pin `release/v1`) | Official publish action; trusted-publisher, Sigstore attestations | PyPA-maintained; supports OIDC, attestations default-on | [VERIFIED: github.com/pypa/gh-action-pypi-publish/releases/latest — v1.14.0, 2026-04-07] |
| `actions/checkout` | v4 | Check out repo | Standard; already used in `.github/workflows/ci.yml` | [CITED: github.com/actions/checkout] |
| `astral-sh/setup-uv` | v5 | Install uv in CI | Already used in `ci.yml` | [VERIFIED: existing `.github/workflows/ci.yml`] |
| `actions/setup-python` | v5 | Install python in CI | Already used in `ci.yml` | [VERIFIED: existing `.github/workflows/ci.yml`] |

### Extras — Verified Current Versions

Each row shows the minimum in D-03 vs the latest available in the registry as of 2026-04-11. Pins are fine as declared in D-03; latest column confirms nothing is yanked or abandoned.

| Extra | Package | D-03 Min | Latest on PyPI | Provenance |
|-------|---------|----------|----------------|------------|
| `[memory-pg]` | `psycopg[binary]` | `>=3.3` | `3.3.3` | [VERIFIED: pypi.org 2026-04-11] |
| `[memory-pg]` | `psycopg-pool` | `>=3.2` | `3.3.0` | [VERIFIED] |
| `[memory-pg]` | `pgvector` | `>=0.4.2` | `0.4.2` | [VERIFIED] |
| `[memory-chroma]` | `chromadb-client` | `>=1.5.6` | `1.5.7` | [VERIFIED] |
| `[memory-es]` | `elasticsearch[async]` | `>=8.0,<9` | `9.3.0` available | [VERIFIED] — note: `<9` cap is intentional; ES 9 removed the `[async]` extras shape. Planner should NOT raise the cap without code audit. |
| `[dispatch]` | `redis` | `>=5.0.0` | `7.4.0` | [VERIFIED] |
| `[dispatch]` | `arq` | `>=0.27` | `0.27.0` | [VERIFIED] — arq has not released past 0.27; it's stable but slow-moving. OK. |
| `[sandbox]` | **(none)** | — | — | [VERIFIED: grep + import inspection of `zeroth/core/sandbox_sidecar/*.py` in this session — no `docker` or other runtime import beyond fastapi/pydantic which are already in base] |

**Installation verification commands:**

```bash
# Each extra, in a clean venv:
python -m venv /tmp/t && . /tmp/t/bin/activate && \
  pip install "zeroth-core[memory-pg]==0.1.1" && \
  python -c "import zeroth.core.memory.pgvector_connector"
```

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `uv build` | `pypa/build` GitHub Action | Both work; uv is faster and already in toolchain; pypa/build is the "blessed" generic. Recommendation: `uv build`. |
| Static `version` in pyproject.toml | `hatch-vcs` dynamic version from git tag | hatch-vcs removes version/tag drift but adds a moving part. CONTEXT D-07 locks static. Keep as-is. |
| Single `pypi` env for both PyPI + TestPyPI | Separate `pypi` + `testpypi` envs | PyPI docs recommend **separate environments** — one trusted publisher per (repo, workflow, env) triple, and TestPyPI is a different index requiring its own publisher registration. [CITED: docs.pypi.org/trusted-publishers/using-a-publisher] |
| Job-level `id-token: write` | Workflow-level | Minimum-scope principle: put the permission only on the publish job, not the whole workflow. [CITED: pypa/gh-action-pypi-publish README] |

## Architecture Patterns

### Recommended `pyproject.toml` Structure (after Phase 28)

```toml
[project]
name = "zeroth-core"
version = "0.1.1"   # MUST bump; 0.1.0 is already taken on PyPI
description = "Governed medium-code platform for production-grade multi-agent systems"
readme = "README.md"
requires-python = ">=3.12"
license = "Apache-2.0"   # PEP 639 SPDX expression form (hatchling 1.27+ supports this)
authors = [{ name = "..." }]
keywords = ["agents", "multi-agent", "llm", "governance", "fastapi"]
classifiers = [
    "Development Status :: 4 - Beta",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.12",
    "Framework :: FastAPI",
    "Topic :: Software Development :: Libraries :: Python Modules",
    "Operating System :: OS Independent",
    "Typing :: Typed",
]

dependencies = [
    "fastapi>=0.115",
    "httpx>=0.27",
    "pydantic>=2.10",
    "pydantic-settings>=2.13",
    "sqlalchemy>=2.0",
    "aiosqlite>=0.22",
    "alembic>=1.18",
    "PyJWT[crypto]>=2.10",
    "PyYAML>=6.0",
    "python-dotenv>=1.0",
    "governai>=0.2.3",
    "econ-instrumentation-sdk>=0.1.1",
    "tenacity>=8.2",
    "cachetools>=5.5",
    "litellm>=1.83,<2.0",
    "langchain-litellm>=0.3.4",
    "uvicorn>=0.30",
    "mcp>=1.7,<2.0",
]

[project.optional-dependencies]
memory-pg = [
    "psycopg[binary]>=3.3",
    "psycopg-pool>=3.2",
    "pgvector>=0.4.2",
]
memory-chroma = ["chromadb-client>=1.5.6"]
memory-es = ["elasticsearch[async]>=8.0,<9"]
dispatch = [
    "redis>=5.0.0",
    "arq>=0.27",
]
sandbox = []   # no runtime deps; shells out to docker CLI
all = [
    "zeroth-core[memory-pg]",
    "zeroth-core[memory-chroma]",
    "zeroth-core[memory-es]",
    "zeroth-core[dispatch]",
    "zeroth-core[sandbox]",
]

[project.urls]
Homepage = "https://github.com/rrrozhd/zeroth-core"
Source = "https://github.com/rrrozhd/zeroth-core"
Issues = "https://github.com/rrrozhd/zeroth-core/issues"
Changelog = "https://github.com/rrrozhd/zeroth-core/blob/main/CHANGELOG.md"

[build-system]
requires = ["hatchling>=1.27"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/zeroth/core"]   # see pitfall #2 below before trusting this
```

### Pattern: Trusted-Publisher Release Workflow (D-11/D-12/D-13/D-14)

Canonical pattern for `.github/workflows/release-zeroth-core.yml` based on PyPI docs and pypa/gh-action-pypi-publish README:

```yaml
# Source: https://docs.pypi.org/trusted-publishers/using-a-publisher/
# Source: https://github.com/pypa/gh-action-pypi-publish#trusted-publishing
name: Release zeroth-core
on:
  release:
    types: [published]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - name: Assert tag matches pyproject.toml version
        run: |
          PYPROJ_VERSION=$(python -c "import tomllib; print(tomllib.load(open('pyproject.toml','rb'))['project']['version'])")
          TAG_VERSION=${GITHUB_REF_NAME#v}
          test "$PYPROJ_VERSION" = "$TAG_VERSION" || { echo "mismatch"; exit 1; }
      - run: uv sync --all-groups
      - run: uv build
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist/ }

  smoke-install:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: |
          python -m venv .venv && . .venv/bin/activate
          pip install dist/*.whl
          python -c "import zeroth.core; print(zeroth.core.__path__)"

  test-wheel:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: |
          python -m venv .venv && . .venv/bin/activate
          pip install dist/*.whl pytest pytest-asyncio
          pytest tests/ -v --no-header -ra

  publish-testpypi:
    needs: [smoke-install, test-wheel]
    runs-on: ubuntu-latest
    environment: testpypi          # separate env per D-11/research finding
    permissions:
      id-token: write              # job-level only per D-14
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/
          # attestations default-on per v1.14.0 — no flag needed

  smoke-from-testpypi:
    needs: publish-testpypi
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: |
          VERSION=$(python -c "import tomllib; print(tomllib.load(open('pyproject.toml','rb'))['project']['version'])")
          python -m venv .venv && . .venv/bin/activate
          # Retry loop — TestPyPI indexing can lag ~30-60s after publish
          for i in 1 2 3 4 5; do
            pip install --index-url https://test.pypi.org/simple/ \
                        --extra-index-url https://pypi.org/simple/ \
                        "zeroth-core==$VERSION" && break
            sleep 15
          done
          if [ -n "$ANTHROPIC_API_KEY" ]; then
            python examples/hello.py
          else
            echo "SKIP: no ANTHROPIC_API_KEY available (forked PR)"
          fi

  publish-pypi:
    needs: smoke-from-testpypi
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with: { name: dist, path: dist/ }
      - uses: pypa/gh-action-pypi-publish@release/v1
```

### Pattern: `py.typed` Marker (PEP 561)

Empty file at `src/zeroth/core/py.typed` + ensure hatchling includes it in the wheel. Gives downstream users type-hint propagation. Recommended but optional.

### Anti-Patterns to Avoid

- **Raising `elasticsearch[async]>=8.0,<9` to `<10`.** ES 9.x removed the `async` extras shape; upgrade would break import. Audit code first.
- **Publishing to PyPI with a password / API token secret.** Defeats the purpose of trusted publishers. `permissions: id-token: write` is the only credential.
- **`permissions: id-token: write` at workflow level.** Job-level only — minimum scope. Build and test jobs don't need OIDC.
- **Omitting `environment:` from the publish job.** PyPI trusted-publisher config can (and should) be scoped to a specific GitHub environment. Environment = audit + approval hook.
- **Uploading the same version twice.** PyPI is immutable — re-upload is rejected with 400. Fix by bumping version; never by "yanking and re-uploading."
- **`packages = ["src/zeroth/core"]` without verification.** Hatchling derives the destination directory from the last path segment. Using `packages` with a namespace package may produce a wheel where files are at `core/foo.py` instead of `zeroth/core/foo.py`. See pitfall #2.
- **Building from src in CI and running tests in src mode, then publishing.** A src-mode test doesn't catch packaging bugs (missing files in wheel, wrong `__init__.py` inclusion). Always test against the **built wheel** before publishing.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Publishing to PyPI from CI | Custom twine-based upload with secret tokens | `pypa/gh-action-pypi-publish@release/v1` + trusted publisher | Handles OIDC, attestations, retries, rate limits, and matches PyPA support surface |
| Signing artifacts | Ad-hoc GPG signing | Sigstore attestations (default-on in action v1.14+) | Free, keyless, PyPI-native verification |
| Building sdist+wheel | Hand-written `setup.py` / shell scripts | `uv build` or `python -m build` | PEP 517 compliant, reproducible, handles hatchling hooks |
| License boilerplate | Summarize Apache-2.0 yourself | Copy verbatim from https://www.apache.org/licenses/LICENSE-2.0.txt | Only the canonical text gives the patent grant + protections |
| Changelog format | Free-form markdown | keepachangelog 1.1.0 structure (`Added/Changed/Deprecated/Removed/Fixed/Security`) | Machine-parseable, expected by downstream tools, matches PKG-04 |
| Version-tag consistency check | Trusting humans | Workflow step that reads `pyproject.toml` and asserts `GITHUB_REF_NAME == v{version}` | One-line guard that prevents the most common release bug |
| Extras verification | Manual "I think it installs" | CI matrix job: one entry per extra, each does clean-venv install + import smoke | Matches D-04 and makes PKG-03 provable |

## Runtime State Inventory

This phase is a packaging/metadata phase, not a rename or migration. No runtime state categories apply, with one exception:

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None | — |
| Live service config | **PyPI-side trusted publisher config** for `zeroth-core` — lives in the PyPI web UI, not git. TestPyPI needs a separate entry. | MANUAL USER ACTION — add publisher in pypi.org and test.pypi.org UIs (see "User Actions" section) |
| OS-registered state | None | — |
| Secrets / env vars | `ANTHROPIC_API_KEY` (or chosen LLM provider) must exist as a GitHub Actions secret for the smoke-from-testpypi step to actually run `hello.py`. If absent, the step skips with a message; it is not a blocker for publishing. | Optional user action — add secret to `pypi` / `testpypi` environment (or repo secrets) for end-to-end smoke coverage |
| Build artifacts | `dist/zeroth_core-0.1.0-py3-none-any.whl` and `.tar.gz` exist locally from the manual first publish. Not consequential. | None — CI rebuilds from scratch |
| **Prior PyPI state** | **`zeroth-core==0.1.0` is already live on PyPI** (immutable). Next release MUST be `0.1.1` or later. | Bump version in pyproject.toml before first workflow run |

## Common Pitfalls

### Pitfall 1: Version immutability trap (PyPI)
**What goes wrong:** Developer sets `version = "0.1.0"` in pyproject.toml, triggers the release workflow, publish-pypi job fails with HTTP 400 "File already exists."
**Why it happens:** PyPI (and TestPyPI) versions are immutable — once `zeroth-core==0.1.0` is uploaded, the same filename can never be uploaded again, even if yanked.
**How to avoid:** Start Phase 28 by bumping `[project].version` to `0.1.1`. Add the tag-matches-version CI step (D-07) so the failure happens in `build`, not `publish-pypi`.
**Warning signs:** Error containing "File already exists" or "400 Bad Request" from twine/gh-action-pypi-publish.
**Known-present in this project:** `zeroth-core==0.1.0` is already on PyPI [VERIFIED: https://pypi.org/pypi/zeroth-core/json].

### Pitfall 2: Hatchling `packages` key drops the namespace prefix
**What goes wrong:** Planner changes `packages = ["src/zeroth"]` → `packages = ["src/zeroth/core"]` per D-21. The built wheel now has files at `core/graph/...` instead of `zeroth/core/graph/...`, breaking `import zeroth.core`.
**Why it happens:** Hatchling's `packages` key takes the **last path segment** as the top-level name in the wheel by default. `src/zeroth/core` → `core`, not `zeroth/core`. The namespace-package pattern in hatchling is documented via `sources` or `only-include`, not `packages`.
**How to avoid:** Two safer options:
1. **Leave it alone.** Current `packages = ["src/zeroth"]` already produces the correct wheel [VERIFIED: `unzip -l dist/zeroth_core-0.1.0-py3-none-any.whl` shows `zeroth/core/__init__.py` entries and no `zeroth/__init__.py`]. D-21's premise that it's "wrong" is incorrect.
2. **If changing, use `only-include` + `sources`:**
   ```toml
   [tool.hatch.build.targets.wheel]
   sources = ["src"]
   only-include = ["src/zeroth/core"]
   ```
   This preserves the `zeroth/core/` prefix by stripping `src/` and including the `zeroth/core` subtree verbatim.
**Verification:** After any change, `uv build && unzip -l dist/zeroth_core-*.whl | head` must show entries starting with `zeroth/core/` and no `zeroth/__init__.py`.

### Pitfall 3: TestPyPI publisher config forgotten
**What goes wrong:** Publish-pypi job works; publish-testpypi job fails with OIDC rejection.
**Why it happens:** PyPI and TestPyPI are independent indices; trusted publishers must be registered **separately** on each. A single config on pypi.org does not cover test.pypi.org.
**How to avoid:** Task checklist must include: "Register publisher on pypi.org AND test.pypi.org for repo=rrrozhd/zeroth-core, workflow=release-zeroth-core.yml, env=pypi (for pypi.org) and env=testpypi (for test.pypi.org)."
**Warning signs:** `invalid-publisher: valid token, but no corresponding publisher` error in the publish-testpypi job log.

### Pitfall 4: TestPyPI indexing lag breaks smoke-install
**What goes wrong:** `publish-testpypi` succeeds, `smoke-from-testpypi` runs immediately, `pip install` fails with "No matching distribution" — then passes 60s later.
**Why it happens:** TestPyPI's simple index takes 15–60s to surface new uploads.
**How to avoid:** Add a retry loop to the smoke-install step (5 attempts, 15s each) as shown in the workflow pattern above. Or a fixed `sleep 60` — less elegant but bulletproof.
**Warning signs:** First-attempt failure that self-corrects on retry.

### Pitfall 5: `id-token: write` at workflow level
**What goes wrong:** An unrelated CI step (say, a pre-publish linter that runs `pip install` from an uncontrolled source) gains the OIDC token and could exfiltrate a PyPI identity.
**Why it happens:** Permissions are inherited by all jobs unless overridden.
**How to avoid:** Declare `permissions: { id-token: write, contents: read }` at **job level** on `publish-testpypi` and `publish-pypi` only. Other jobs get the default (no OIDC).
**Reference:** pypa/gh-action-pypi-publish README "security considerations" section.

### Pitfall 6: `hello.py` crashes from clean venv because a required env var is missing
**What goes wrong:** CI's `python examples/hello.py` step fails on PRs from forks (no secrets) even though the publish succeeded.
**Why it happens:** `ANTHROPIC_API_KEY` not present in forked-PR context.
**How to avoid:** In the CI step, branch: `if [ -n "$ANTHROPIC_API_KEY" ]; then python examples/hello.py; else echo "SKIP: no API key (fork)"; fi`. The script itself should print a clear error when run locally without a key so first-time users don't blame the library.
**Reference:** D-19 makes this explicit.

### Pitfall 7: Using `elasticsearch` 9.x by accident
**What goes wrong:** A future maintainer raises the pin to `>=9`, the `[async]` extras shape is gone, import breaks.
**How to avoid:** Keep the `<9` cap in D-03 verbatim. Leave a comment in pyproject.toml referencing this research doc if the planner wants a paper trail.

## Code Examples

### Verified: current hatchling wheel target produces correct layout

```bash
# Source: this session, verified output
$ unzip -l dist/zeroth_core-0.1.0-py3-none-any.whl | head -5
  Length      Date    Time    Name
---------  ---------- -----   ----
      827  02-02-2020 00:00   zeroth/core/__init__.py
     2772  02-02-2020 00:00   zeroth/core/agent_runtime/__init__.py
     2020  02-02-2020 00:00   zeroth/core/agent_runtime/errors.py
```

No top-level `zeroth/__init__.py`. Namespace package layout is correct as-is.

### Canonical trusted-publisher step

```yaml
# Source: https://github.com/pypa/gh-action-pypi-publish#trusted-publishing
- name: Publish to PyPI
  uses: pypa/gh-action-pypi-publish@release/v1
  # No username, no password — OIDC token provides identity.
  # attestations: true is the default since v1.11; omit the key.
```

### Canonical PEP 639 license declaration (hatchling 1.27+)

```toml
# Source: https://peps.python.org/pep-0639/
[project]
license = "Apache-2.0"
```

### Apache-2.0 canonical text source

```
https://www.apache.org/licenses/LICENSE-2.0.txt
```

Copy verbatim into `LICENSE` at repo root. Do not paraphrase. Do not add a copyright block at the top unless following the Apache appendix instructions.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `twine upload` with `PYPI_API_TOKEN` secret | OIDC trusted publisher + `pypa/gh-action-pypi-publish` | Stable since ~2023 | No long-lived tokens; tied to workflow identity |
| GPG-signed release artifacts | Sigstore attestations (keyless) | Default-on in action v1.11+ (2024) | Free, automatic, no key management |
| `[project] license = { text = "Apache-2.0" }` | `[project] license = "Apache-2.0"` (SPDX expression) | PEP 639, supported in hatchling 1.27+ | Cleaner, PyPI renders correctly |
| `python -m build` | `uv build` | 2024+ | Faster; already in project toolchain |
| Single GH environment for PyPI + TestPyPI | Separate `pypi` + `testpypi` envs | PyPI docs ~2024 | One trusted publisher per env, scoped audit |

**Deprecated/outdated:**
- `[project] license-files` with glob patterns is superseded by PEP 639 SPDX expressions for single-file Apache-2.0 projects (files still included in the wheel via SDIST by default).
- `setup.py` builds — hatchling + PEP 517 is the state of the art for pure-Python libraries.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `uv` | local build verification | ✓ | 0.11.3 | — |
| `git` | tag assertion, releases | ✓ | 2.50.1 | — |
| `gh` CLI | creating GitHub Release | ✓ | 2.88.0 | web UI |
| `python` (system) | local tests | Python 3.9.6 system; uv manages 3.12+ | — | uv-managed venv |
| Network to pypi.org | verification fetches | ✓ | — | — |
| PyPI trusted-publisher configured for zeroth-core | `publish-pypi` job | **✗ — MANUAL USER ACTION** | — | Publishing will fail with `invalid-publisher` until registered |
| TestPyPI trusted-publisher configured | `publish-testpypi` job | **✗ — MANUAL USER ACTION** | — | Same — register separately on test.pypi.org |
| `ANTHROPIC_API_KEY` GitHub secret | `smoke-from-testpypi` hello.py run | Unknown | — | Step skips if absent (not a blocker) |

**Missing dependencies with no fallback:**
- PyPI trusted-publisher configuration (one per index). The planner's PLAN.md must contain an explicit "USER ACTION REQUIRED" checkpoint before the workflow can succeed.

**Missing dependencies with fallback:**
- LLM API key — smoke step degrades gracefully.

## Security Domain

Phase 28 is a release-engineering phase, not a feature phase. Security concerns are release-supply-chain-focused.

### Applicable ASVS categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | N/A — no app auth changes |
| V3 Session Management | no | — |
| V4 Access Control | yes (release pipeline) | GitHub environment protection rules + trusted-publisher binding (repo + workflow + env triple) |
| V5 Input Validation | no | — |
| V6 Cryptography | yes | Sigstore keyless signing via pypa/gh-action-pypi-publish (no hand-rolled signing) |
| V14 Configuration | yes | `id-token: write` job-scoped; `contents: read` default; no secrets in workflow files; `.env` and credentials never committed |

### Known threat patterns for release pipelines

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Compromised CI step exfiltrates PyPI credentials | Information Disclosure | OIDC trusted publisher (no long-lived token to exfiltrate) + job-scoped `id-token: write` |
| Malicious workflow merged to main pushes rogue release | Tampering | PyPI publisher binding to specific workflow file name; require PR review on main; release only on GitHub Release `published` event (human gate) |
| Dependency confusion / typosquatting | Spoofing | Publish attestations so consumers can verify origin; pin hatchling to `>=1.27`; use official PyPA action pinned by SHA or `release/v1` |
| Tag/version mismatch fooling the release | Tampering | Pre-publish tag-matches-pyproject.toml assertion step |
| Test suite bypass via skipped tests | Tampering | `-ra` flag in pytest surfaces skips; no `--ignore` in release workflow; run tests against the built wheel not src |

### User actions (cannot be automated from this repo)

- [ ] Register trusted publisher on **pypi.org** → Manage `zeroth-core` → Publishing → Add (owner=`rrrozhd`, repo=`zeroth-core`, workflow=`release-zeroth-core.yml`, environment=`pypi`)
- [ ] Register trusted publisher on **test.pypi.org** → same fields, environment=`testpypi`
- [ ] Create `pypi` GitHub environment (no required reviewers per D-11; optional: enable "Required reviewers" for production gate)
- [ ] Create `testpypi` GitHub environment
- [ ] (Optional) Add `ANTHROPIC_API_KEY` to the `testpypi` environment for end-to-end hello-world smoke
- [ ] (Optional) Create a `CODEOWNERS` file if PR review on release branches is desired

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `[sandbox]` extra can be empty `= []` because sandbox_sidecar only imports base deps | Extras table | LOW — `pip install "zeroth-core[sandbox]"` still succeeds; if a future sidecar dep is added it can be moved into the extra with a minor bump |
| A2 | Planner will resolve the version immutability conflict (D-05) by bumping to 0.1.1 | Summary / pitfalls | MEDIUM — if planner ignores this and sets version to 0.1.0, first release workflow run fails at publish-pypi |
| A3 | Hatchling 1.27+ supports PEP 639 `license = "Apache-2.0"` SPDX form | pyproject.toml pattern | LOW — falls back to `license = { text = "Apache-2.0" }` if not supported; no functional change |
| A4 | `pypa/gh-action-pypi-publish@release/v1` is safe to pin by floating tag | Workflow pattern | LOW — PyPA maintains backward compatibility on `release/v1`; pinning by SHA is an option if the user prefers |
| A5 | `elasticsearch[async]>=8.0,<9` is still the correct shape (i.e., v8 still exposes `[async]` extras) | Extras table | LOW — verified current published v8 wheel has `[async]` extras |
| A6 | `testpypi` environment name is a convention, not a requirement | Workflow pattern | LOW — any name works as long as PyPI publisher config matches |

## Open Questions

1. **Version bump target.** `0.1.0` is already on PyPI. Should Phase 28 ship `0.1.1` (signals "packaging improvements only, no API changes") or `0.2.0` (signals "extras reorganized, base dependencies shrunk")?
   - What we know: CONTEXT.md D-05 says `0.1.0`, which is impossible.
   - What's unclear: user intent on version messaging.
   - Recommendation: **`0.1.1`**. The extras carve-out IS a breaking change for any consumer already using `zeroth-core==0.1.0` and expecting `psycopg` / `chromadb-client` to be available — but since the only known consumer is the user himself, `0.1.1` with a CHANGELOG `### Changed` entry is sufficient. If user disagrees, `0.2.0` is equally defensible. Planner should surface this to user before execution.

2. **`[sandbox]` — empty extra or remove?** An empty extra is legal and installable but may confuse users ("what did I get?"). Alternative is to drop `sandbox` from PKG-03 entirely — but this requires a REQUIREMENTS.md amendment.
   - Recommendation: **Keep `[sandbox] = []`**, document in CHANGELOG that it's a marker extra reserved for future sidecar-client deps. Matches PKG-03 verbatim.

3. **D-21 hatchling wheel target fix — skip or validate?**
   - Recommendation: Skip. Current config is correct. Add a task to document this finding in a pyproject.toml comment so a future reader doesn't re-open the question.

4. **Should the release workflow also run ruff/interrogate before publishing?** The existing `ci.yml` does, on every push. Belt-and-suspenders vs minimum scope.
   - Recommendation: **No**. `ci.yml` already gates merges to main; re-running lint in the release workflow is redundant and slows publish. Run pytest against the built wheel (packaging-specific signal) but not lint.

## Validation Architecture

`workflow.nyquist_validation` is explicitly `false` in `.planning/config.json`. Section omitted per protocol.

## Project Constraints (from CLAUDE.md)

- **Progress logging is mandatory.** Every meaningful unit of work must update `PROGRESS.md` via the `progress-logger` skill. Applies to all Phase 28 tasks. (Note: the `.claude/skills/progress-logger/` directory exists but no `SKILL.md` was found; planner should assume the skill is invocable and gracefully handle absence if not.)
- **Build commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`. Release workflow should invoke these exact commands where applicable.
- **Project layout:** `src/zeroth/` package dir, `tests/` for pytest. Already in place.
- **Context efficiency directive:** implementers should read PROGRESS.md + their phase PLAN.md, nothing else. Planner should structure tasks so each is self-contained.

## Sources

### Primary (HIGH confidence)
- [PyPI — zeroth-core JSON](https://pypi.org/pypi/zeroth-core/json) — confirms `0.1.0` already published (2026-04-10)
- [PyPI — econ-instrumentation-sdk JSON](https://pypi.org/pypi/econ-instrumentation-sdk/json) — confirms `0.1.1` live (PKG-01 satisfied)
- [PyPI — hatchling / psycopg / chromadb-client / etc.](https://pypi.org/) — current versions verified 2026-04-11
- [pypa/gh-action-pypi-publish release v1.14.0](https://github.com/pypa/gh-action-pypi-publish/releases/tag/v1.14.0) — 2026-04-07
- [PyPI Trusted Publishers — Using a Publisher](https://docs.pypi.org/trusted-publishers/using-a-publisher/) — separate env per index, OIDC audience rules
- [PyPI Trusted Publishers — Adding a Publisher](https://docs.pypi.org/trusted-publishers/adding-a-publisher/)
- [PyPI Attestations — Producing Attestations](https://docs.pypi.org/attestations/producing-attestations/) — default-on behavior
- [Python Packaging User Guide — Publishing via GitHub Actions](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/)
- [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/) — six section types
- [Apache License 2.0 canonical text](https://www.apache.org/licenses/LICENSE-2.0.txt)
- [PEP 639 — Improving license clarity](https://peps.python.org/pep-0639/)
- [PEP 420 — Implicit Namespace Packages](https://peps.python.org/pep-0420/)
- [PEP 561 — Type Information Distribution](https://peps.python.org/pep-0561/)
- Local filesystem inspection of `src/zeroth/core/sandbox_sidecar/*.py` — confirms no non-base runtime deps
- Local inspection of `dist/zeroth_core-0.1.0-py3-none-any.whl` via `unzip -l` — confirms correct layout

### Secondary (MEDIUM confidence)
- [hatch discussions #819 — namespace package config](https://github.com/pypa/hatch/discussions/819) — `sources` + `only-include` pattern
- [Hatch build configuration docs](https://hatch.pypa.io/1.13/config/build/)
- [Simon Willison TIL — PyPI releases from GitHub Actions](https://til.simonwillison.net/pypi/pypi-releases-from-github)

### Tertiary (LOW confidence)
- None — all claims in this research are either verified against live registries/filesystem or cited to official PyPA / PEP documentation.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified against live PyPI in this session.
- Trusted-publisher workflow pattern: HIGH — matches PyPA official docs verbatim; pattern is widely deployed.
- Hatchling wheel target behavior: HIGH for current state (verified via wheel inspection); MEDIUM for the D-21 alternate form (documented via Hatch discussions, not personally tested).
- `[sandbox]` empty claim: HIGH — direct import inspection.
- Version immutability: HIGH — it's a documented PyPI invariant.

**Research date:** 2026-04-11
**Valid until:** 2026-05-11 (stable release-engineering domain; re-verify `pypa/gh-action-pypi-publish` version and pinned library versions before each release)
