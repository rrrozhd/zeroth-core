# Phase 28: PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`) - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 28 ships `zeroth-core` as a real, pip-installable library on PyPI with:

1. **Optional-dependency extras** carved out of today's monolithic `dependencies` list (`memory-pg`, `memory-chroma`, `memory-es`, `dispatch`, `sandbox`, `all`).
2. **OSS-grade repo metadata** at the repo root: `LICENSE` (Apache-2.0), `CHANGELOG.md` (keepachangelog), `CONTRIBUTING.md`.
3. **Trusted-publisher (OIDC) GitHub Actions release workflow** for `zeroth-core` only — no long-lived API tokens, TestPyPI staging job before production publish, Sigstore attestations enabled.
4. **End-to-end acceptance fixture** (`examples/hello.py`) that proves a clean-venv `pip install zeroth-core` produces a working program.
5. **Packaging hardening** — fix the hatchling wheel target so the published wheel actually reflects the Phase 27 namespace layout.

`econ-instrumentation-sdk` is treated as **already published** (PyPI 0.1.1, pinned in `pyproject.toml` since commit 78c2076). Any further econ-sdk publishing work belongs in the Regulus repo, not here. STATE.md blockers about a missing Regulus remote are stale and will be cleaned up as part of this phase.

This phase does NOT deliver:
- Docs site, Getting Started narrative, Cookbook (Phases 30–32)
- `zeroth-studio` repo split (Phase 29)
- Code-of-conduct, governance model, RFC process, issue templates (deferred — heavyweight OSS treatment)
- SBOM generation, dependency-graph publishing
- New runtime features

</domain>

<decisions>
## Implementation Decisions

### Optional Extras Split (PKG-03)

- **D-01:** Base `dependencies` (installed by bare `pip install zeroth-core`) is **minimal core only**:
  `fastapi`, `httpx`, `pydantic`, `pydantic-settings`, `sqlalchemy`, `aiosqlite`, `alembic`, `PyJWT[crypto]`, `PyYAML`, `python-dotenv`, `governai`, `econ-instrumentation-sdk`, `tenacity`, `cachetools`, `litellm`, `langchain-litellm`, `uvicorn`, `mcp`.
  Backend-specific packages (`psycopg`, `pgvector`, `chromadb-client`, `elasticsearch`, `redis`, `arq`) move out of base into extras.
- **D-02:** Extra names are **locked verbatim per PKG-03**: `[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]`. No renaming, no scheme refinement — matches REQUIREMENTS.md exactly so verification is unambiguous.
- **D-03:** Extra contents:
  - `[memory-pg]` → `psycopg[binary]>=3.3`, `psycopg-pool>=3.2`, `pgvector>=0.4.2`
  - `[memory-chroma]` → `chromadb-client>=1.5.6`
  - `[memory-es]` → `elasticsearch[async]>=8.0,<9`
  - `[dispatch]` → `redis>=5.0.0`, `arq>=0.27` (both — arq requires redis; one extra enables the entire distributed-worker path)
  - `[sandbox]` → planner discovers exact deps (likely the container/sidecar Python clients used by `zeroth.core.sandbox_sidecar`)
  - `[all]` → `[memory-pg] + [memory-chroma] + [memory-es] + [dispatch] + [sandbox]` (every memory backend installable side-by-side; runtime config picks which one is active)
- **D-04:** **Verification gate per extra:** for each extra, planner must produce a CI job (or matrix entry) that creates a clean venv, runs `pip install "zeroth-core[<extra>]"`, and imports the modules that depend on it. PKG-03 says "each extra resolves and installs cleanly" — this is how we prove it.

### Versioning & Release Strategy

- **D-05:** First PyPI version: **`0.1.0`** (matches current `pyproject.toml`, no bump). Signals "usable but pre-1.0 API may evolve." Aligns with `econ-instrumentation-sdk` 0.1.1 cadence.
- **D-06:** Versioning scheme: **SemVer**. Pre-1.0 the minor bump indicates breaking change, patch is additive.
- **D-07:** Version source of truth: **`pyproject.toml [project].version` (static)**. Hand-edited at release time. Release workflow asserts the git tag matches `[project].version` before publishing — refuses to release on mismatch.
- **D-08:** Release trigger: **GitHub Release published event**. Operator (or `gh release create v0.1.0 --generate-notes …`) creates a GitHub Release; the `release: { types: [published] }` event fires the publish workflow. Gives free release notes UI and an explicit human-action gate.
- **D-09:** **TestPyPI dry-run is mandatory** before each production publish. The publish workflow has staged jobs (build → test → publish-testpypi → smoke-install-from-testpypi → publish-pypi). Both publishes use trusted publishers — no API tokens.

### Trusted-Publisher CI Design (PKG-05)

- **D-10:** **Separate workflows per package.** Phase 28 owns `.github/workflows/release-zeroth-core.yml` only. The `econ-instrumentation-sdk` publish workflow lives in the Regulus repo and is out-of-scope here.
- **D-11:** GitHub environment name: **`pypi`** (single env, no required reviewers). The release is gated by who can push tags / create GitHub Releases — adding env approval would be duplicate friction. TestPyPI publish runs in the same `pypi` environment (or a sibling `testpypi` env — Claude's Discretion based on PyPI trusted-publisher config requirements).
- **D-12:** Pre-publish gates (workflow stages, in order):
  1. **build** — `uv build` produces sdist + wheel
  2. **smoke-install** — clean venv, `pip install dist/*.whl`, `python -c "import zeroth.core; print(zeroth.core.__path__)"` plus a few key public-API imports
  3. **tests** — `uv run pytest` against the built+installed wheel (not src) to catch packaging bugs that don't show up in dev mode
  4. **publish-testpypi** — pypa/gh-action-pypi-publish with `repository-url: https://test.pypi.org/legacy/`
  5. **smoke-install-from-testpypi** — clean venv, `pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ zeroth-core==<version>`, run `python examples/hello.py` (env-gated LLM, see D-19)
  6. **publish-pypi** — pypa/gh-action-pypi-publish to production
- **D-13:** **Sigstore attestations enabled.** Publish action gets `attestations: true` and `id-token: write` permission. Free Sigstore signing via PyPI's built-in support. SBOM generation deferred.
- **D-14:** OIDC permissions: workflow declares `permissions: { id-token: write, contents: read }` at job level (not workflow level — minimum scope).

### LICENSE / CHANGELOG / CONTRIBUTING (PKG-04)

- **D-15:** **License: Apache-2.0.** Permissive + explicit patent grant + contributor protection. Standard for governed/enterprise-adjacent libraries. `LICENSE` file at repo root contains the full canonical Apache-2.0 text. `pyproject.toml [project].license = { text = "Apache-2.0" }` (or SPDX expression form, planner picks per current PEP 639 support in hatchling).
- **D-16:** **CHANGELOG.md** in keepachangelog 1.1.0 format. Seeded with a single `[0.1.0] - 2026-04-XX` entry summarizing: "First public PyPI release. Namespace rename to `zeroth.core.*` (Phase 27). Optional extras introduced. Trusted-publisher OIDC release pipeline." No retrospective covering pre-rename history.
- **D-17:** **CONTRIBUTING.md**: ~1 page covering (a) dev setup (`uv sync`, `uv run pytest`, `uv run ruff check`), (b) PR conventions (commit format, branch naming), (c) where to file issues, (d) link to LICENSE and code-of-conduct (deferred — note "TBD" for now). No RFC process, no governance section, no issue templates.

### PKG-06 Acceptance Fixture

- **D-18:** **Ship `examples/hello.py` in this repo.** ~30-line script that builds a tiny graph with one agent node, runs it, prints output. The CI smoke-install step (D-12 step 5) executes this file from a clean venv as the PKG-06 acceptance test. Phase 30 will later wrap narrative around the same file — no chicken-and-egg, no duplication.
- **D-19:** **`hello.py` uses a real LLM, env-gated.** The example uses `litellm` with an env-var API key (e.g., `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`). CI smoke-install step skips with a clear message if no key is present, so PRs from forks don't break. Locally, the example works out of the box for any user with a key in their env.
  - Planner must decide: which provider as the default? Closest to "first impression" matters. Recommend `ANTHROPIC_API_KEY` since this is Claude-built.
- **D-20:** `examples/` directory is committed to the repo. Phase 28 adds only `hello.py`. Phase 31 will populate the rest.

### Packaging Hardening (cross-cutting)

- **D-21:** **Fix the hatchling wheel target.** Current `pyproject.toml` has `[tool.hatch.build.targets.wheel] packages = ["src/zeroth"]` — this is wrong for the Phase 27 PEP 420 namespace layout. Change to `packages = ["src/zeroth/core"]` so the built wheel contains only `zeroth/core/` and not a top-level `zeroth/__init__.py`. Verification: `unzip -l dist/zeroth_core-0.1.0-py3-none-any.whl | grep -v "^Archive"` must show `zeroth/core/...` entries and **no** `zeroth/__init__.py`.
- **D-22:** Add `[project.urls]` block: Homepage, Source, Issues, Changelog. Required by PyPI long-description rendering and improves project page quality.
- **D-23:** Add `[project] description`, `keywords`, `classifiers` (Development Status :: 4 - Beta, License :: OSI Approved :: Apache Software License, Programming Language :: Python :: 3.12, Topic :: Software Development :: Libraries, Framework :: FastAPI). Planner discovers the right minimal set.
- **D-24:** Ensure `README.md` renders cleanly on PyPI — verify `[project] readme = "README.md"` is present (already is) and that the README has no broken relative links that would 404 on PyPI.

### STATE.md Reconciliation

- **D-25:** **Update `.planning/STATE.md` blockers** as part of Phase 28 work. Remove the stale "Regulus has no GitHub remote" and "PyPI trusted-publisher setup blocked on Regulus" entries. The trusted-publisher setup for `zeroth-core` IS in scope here, but for `econ-instrumentation-sdk` it is Regulus's problem and not gating Phase 28. Replace blockers list with the actual remaining manual user actions (PyPI trusted-publisher config for `zeroth-core` only, GitHub Release creation procedure).

### Claude's Discretion

- Exact `[sandbox]` extra contents — depends on what `zeroth.core.sandbox_sidecar` actually imports. Planner discovers via grep + import inspection.
- Whether TestPyPI uses the same `pypi` GitHub environment or a separate `testpypi` env — depends on PyPI's trusted-publisher config requirements (one env per index).
- Exact set of `[project] classifiers` and `keywords` for PyPI metadata.
- Whether to add `py.typed` marker file under `src/zeroth/core/` (PEP 561) — recommended yes for typed-library distribution but not strictly required by PKG-*.
- Default LLM provider env var in `hello.py` (`ANTHROPIC_API_KEY` recommended).
- Whether the release workflow uses `uv build` directly or `pypa/build` action — both work; uv-native is more consistent with the rest of the toolchain.
- Whether to also publish to PyPI's `--repository` configured for the `zeroth-core` project name reservation in advance of v0.1.0 release. **Recommendation: yes** — reserve the name on PyPI before the first publish so it cannot be squatted between now and release day. This is a manual user action.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Scope Anchors
- `.planning/ROADMAP.md` §Phase 28 — goal + 5 success criteria are the hard scope boundary
- `.planning/REQUIREMENTS.md` §Packaging (`PKG-01` through `PKG-06`) — acceptance criteria for every Phase 28 deliverable
- `.planning/PROJECT.md` §Key Decisions — locked v3.0 milestone decisions (extract core as pip-installable, `zeroth.core.*` namespace)
- `.planning/STATE.md` §Pending Todos and §Blockers/Concerns — must be reconciled per D-25

### Prior Phase Context
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/27-CONTEXT.md` — packaging metadata decisions D-09 through D-13 are the upstream baseline; the wheel-target fix (D-21 here) corrects what Phase 27 left in an inconsistent state
- `.planning/phases/27-*/PLAN.md` — Phase 27 plan files that touched `pyproject.toml` and the hatchling config

### Current Repo State (to be transformed)
- `pyproject.toml` — current state: `name = "zeroth-core"`, `version = "0.1.0"`, monolithic `dependencies` list with backend deps not yet carved into extras, `[tool.hatch.build.targets.wheel] packages = ["src/zeroth"]` (incorrect — D-21 fixes), no `[project.optional-dependencies]`, no `[project.urls]`, no `classifiers`
- `src/zeroth/core/` — the actual installable package (post Phase 27 rename)
- `src/zeroth/` — namespace-package directory; must not contain `__init__.py` and the wheel must not include one
- `README.md` — repo root readme (PyPI long_description)
- `.github/workflows/` — existing CI workflows; planner discovers conventions before adding `release-zeroth-core.yml`
- `tests/` — existing test suite (~280+ tests); the publish workflow runs it against the built wheel, not src

### Recent Commits (relevant baseline)
- `78c2076` — `fix(27-04): switch econ-instrumentation-sdk to PyPI dep` (PKG-01 satisfied)
- `6bc707e` — `chore(deps): depend on governai>=0.2.3 from PyPI instead of git URL`

### External Specifications
- **PEP 621** — Project metadata in `pyproject.toml` (`[project]` table, optional-dependencies, urls, classifiers, license)
- **PEP 639** — Improving license clarity (SPDX license expressions in `[project] license`)
- **PEP 440** — Version identifiers (SemVer 0.1.0 is also valid PEP 440)
- **PEP 420** — Implicit Namespace Packages (relevant for the wheel-target fix in D-21)
- **PEP 561** — Distributing and packaging type information (`py.typed` marker)
- **PyPI Trusted Publishers docs** — https://docs.pypi.org/trusted-publishers/ — OIDC publisher config, GitHub environment scoping
- **pypa/gh-action-pypi-publish** — official GitHub Action, supports `attestations: true` for Sigstore signing
- **keepachangelog 1.1.0** — https://keepachangelog.com/en/1.1.0/ — CHANGELOG.md format
- **Apache License 2.0** — https://www.apache.org/licenses/LICENSE-2.0.txt — canonical license text
- **Hatchling docs** — `[tool.hatch.build.targets.wheel]` config for namespace packages
- **uv build docs** — `uv build` produces sdist+wheel, used in the publish workflow

### Iteration Log Convention
- `PROGRESS.md` (root) — CLAUDE.md mandates every meaningful unit of work updates this via the `progress-logger` skill. Phase 28 executors MUST follow.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `uv` + `hatchling` toolchain already configured — no new build tools needed
- `interrogate`, `libcst`, `ruff`, `pytest` already in dev deps from Phase 27
- Existing `tests/` suite gives the publish workflow a real "did the wheel break anything" signal

### Established Patterns
- src-layout: `src/zeroth/core/` post-rename — wheel target must point here (D-21)
- uv-based commands: `uv sync`, `uv run pytest`, `uv build` — release workflow uses these
- ruff lint + format with Google docstring convention — already enforced; release CI doesn't need to add new lint
- All deps in a single `[project] dependencies` list — Phase 28 splits this into base + 6 extras

### Integration Points
- `pyproject.toml` is the single touchpoint for: package metadata, dependency split, wheel target fix, project URLs, classifiers
- `.github/workflows/release-zeroth-core.yml` is the new file that owns all release CI
- `examples/hello.py` is a new file at repo root — first occupant of the `examples/` directory
- `LICENSE`, `CHANGELOG.md`, `CONTRIBUTING.md` are new files at repo root — none currently exist
- `README.md` may need a small "Install" section update referencing extras (`pip install "zeroth-core[memory-pg]"`)
- `.planning/STATE.md` blockers list needs reconciliation (D-25)

### Scale Reality Checks
- ~22K LOC under `src/zeroth/core/`
- ~280+ tests in `tests/`
- 6 extras × clean-venv install verification = 6 CI matrix entries (or one job that loops)
- Wheel size after extras carve-out should drop significantly (no chromadb-client / elasticsearch / pgvector pulled by default)

</code_context>

<specifics>
## Specific Ideas

- Extra names are **locked** verbatim per PKG-03. Any deviation requires REQUIREMENTS.md amendment first.
- License is **locked** at Apache-2.0. Not negotiable downstream.
- First version is **locked** at `0.1.0`. Matches current pyproject; no bump.
- TestPyPI dry-run is **mandatory** in every release. Skipping it is not an option even for "trivial" releases.
- `examples/hello.py` is the canonical PKG-06 fixture. Phase 30 will wrap narrative around this exact file — do not duplicate it inside the docs site.
- The `econ-instrumentation-sdk` PyPI publishing question is **resolved as out-of-scope** for Phase 28. PKG-01 is satisfied by the existing `>=0.1.1` PyPI dependency. Any further econ-sdk work happens in the Regulus repo on its own cadence.
- Trusted-publisher config on pypi.org (the manual half of D-10) is a **user action** — Claude cannot click the PyPI web UI buttons. Plan must include an explicit "USER ACTION REQUIRED" checkpoint for: (a) reserve `zeroth-core` name on PyPI, (b) configure trusted publisher for `rrrozhd/zeroth-core` repo + `release-zeroth-core.yml` workflow + `pypi` environment, (c) repeat for TestPyPI.

</specifics>

<deferred>
## Deferred Ideas

- **Phase 29:** `zeroth-studio` repo split via `git filter-repo`. No Phase 28 dependency.
- **Phase 30:** Getting Started narrative tutorial (will reuse `examples/hello.py` from Phase 28).
- **Phase 31:** `examples/` directory expansion with cookbook recipes for every subsystem.
- **Phase 32:** API reference, deployment guide, migration guide.
- **Deferred OSS heavyweight:** code-of-conduct, governance model, RFC process, issue templates, PR templates, security policy (`SECURITY.md`). Belongs in a later docs/community phase, not Phase 28.
- **Deferred:** SBOM (cyclonedx) generation in release workflow. Sigstore attestations are enough for v0.1.0.
- **Deferred:** `hatch-vcs` for git-tag-derived versioning. Static `[project].version` is fine for now; revisit if hand-bump bugs become a problem.
- **Deferred:** Multi-Python-version build matrix (3.13, 3.14). Phase 28 ships py3.12 only; expand later.
- **Deferred:** ARM64 / Linux musl wheels. Pure-Python package, no native extensions — sdist + universal wheel cover everything.
- **Deferred:** `econ-instrumentation-sdk` Regulus-side publishing improvements. Lives in Regulus repo, not here.
- **Deferred:** README "Install" section overhaul to showcase extras — small update is fine in Phase 28; full rewrite belongs in Phase 30.
- **Deferred:** PyPI badge collection (build status, coverage, downloads). Cosmetic, can add anytime.

</deferred>

---

*Phase: 28-pypi-publishing-econ-instrumentation-sdk-zeroth-core*
*Context gathered: 2026-04-11 via /gsd-discuss-phase 28*
