---
phase: 28
plan: 03
subsystem: packaging
tags:
  - github-actions
  - trusted-publisher
  - oidc
  - pypi
  - ci
  - sigstore
requires:
  - 28-01 (pyproject 0.1.1 with extras + py.typed)
  - 28-02 (examples/hello.py fixture + LICENSE/CHANGELOG/CONTRIBUTING)
provides:
  - "Trusted-publisher OIDC release pipeline for zeroth-core with TestPyPI staging + Sigstore attestations"
  - "Per-extra clean-venv install+import CI matrix (PKG-03 continuous gate)"
  - "README install section surfacing optional extras"
affects:
  - "First GitHub Release v0.1.1 (creates first actual PyPI publish — deferred post-phase follow-up)"
tech-stack:
  added:
    - "pypa/gh-action-pypi-publish@release/v1 (trusted publisher action; Sigstore attestations default-on in v1.14+)"
    - "GitHub environments: pypi, testpypi (separate per trusted-publisher pattern)"
  patterns:
    - "Job-scoped id-token: write (never workflow-level) — minimum OIDC scope (D-14, pitfall #5)"
    - "Tag-equals-pyproject-version guard in build job (D-07)"
    - "Retry loop for TestPyPI indexing lag in smoke-from-testpypi (pitfall #4)"
    - "Env-gated ANTHROPIC_API_KEY smoke of examples/hello.py (pitfall #6)"
    - "test-wheel runs pytest against the INSTALLED wheel, not src mode"
    - "verify-extras matrix with fail-fast: false to surface all breakages in one run"
key-files:
  created:
    - .github/workflows/release-zeroth-core.yml
    - .github/workflows/verify-extras.yml
  modified:
    - README.md
key-decisions:
  - "Release workflow has top-level permissions: contents: read ONLY — id-token: write is scoped to publish-testpypi and publish-pypi jobs only (D-14)"
  - "publish-testpypi uses environment=testpypi, publish-pypi uses environment=pypi — two separate GitHub environments, matching the two separate trusted-publisher registrations required on pypi.org vs test.pypi.org"
  - "test-wheel installs the wheel with [all] extras so backend-dependent tests can import their deps against the built artifact"
  - "memory-es import smoke uses zeroth.core.memory.elastic_connector (confirmed via ls src/zeroth/core/memory/ — not es_connector / elasticsearch_connector)"
  - "sandbox matrix entry still runs even though the extra is empty — gate is 'install resolves + module imports from base deps', both true for an empty extra since sandbox_sidecar shells out to docker CLI at runtime"
  - "README Install section added surgically after the intro block — no badges, no section reshuffling, matches the Deferred README overhaul boundary in 28-CONTEXT.md"
  - "No workflow_dispatch or schedule triggers on release workflow — release.published is the only trigger (D-08)"
  - "attestations: true is NOT explicitly set on the publish action because it is default-on in v1.14+ (D-13); adding the flag would pin behavior we don't need"
requirements-completed:
  - PKG-02
  - PKG-03
  - PKG-05
  - PKG-06
duration: "~6 min"
completed: "2026-04-11"
---

# Phase 28 Plan 03: Release Workflow and Extras Verification Summary

Shipped both Phase 28 GitHub Actions workflows that make the release pipeline real: `release-zeroth-core.yml` (six-stage trusted-publisher OIDC pipeline with TestPyPI staging, tag-matches-version guard, retry-looped TestPyPI indexing, env-gated `examples/hello.py` smoke, and default-on Sigstore attestations on both publishes) and `verify-extras.yml` (continuous 6-entry matrix that builds a wheel and clean-installs every declared extra with a per-extra import smoke). Added a surgical Install section to `README.md` surfacing the extras. The one manual step Claude cannot perform — registering trusted publishers on pypi.org + test.pypi.org and creating the matching GitHub environments — is documented as a deferred user action; it must be completed before the first `gh release create v0.1.1` is actually run.

## Metrics

- **Duration:** ~6 min
- **Tasks completed:** 2 / 2 autonomous (Task 3 human-action deferred to the user, not blocking this plan)
- **Files created:** 2 (`.github/workflows/release-zeroth-core.yml`, `.github/workflows/verify-extras.yml`)
- **Files modified:** 1 (`README.md`)
- **Commits:** 2 task commits (`2831469`, `4a003db`)

## What Was Built

### `.github/workflows/release-zeroth-core.yml` — trusted-publisher release pipeline

- **Trigger:** `release: types: [published]` only (per D-08 — no workflow_dispatch, no schedule)
- **Workflow-level permissions:** `contents: read` only. OIDC scope is job-local (D-14, pitfall #5).
- **Six jobs with explicit `needs:` chain:**

  1. **build** — `actions/checkout@v4` → `astral-sh/setup-uv@v5` → `actions/setup-python@v5` (3.12) → **tag-matches-pyproject-version guard** (`tomllib` one-liner, fails loudly on mismatch per D-07) → `uv sync --all-groups` → `uv build` → upload `dist/` artifact.
  2. **smoke-install** (`needs: [build]`) — downloads `dist`, creates clean venv, `pip install dist/*.whl`, `import zeroth.core`.
  3. **test-wheel** (`needs: [build]`) — checkout (for `tests/`) + download `dist`, creates clean venv, installs the wheel with `[all]` extras plus `pytest` + `pytest-asyncio`, runs `pytest tests/ -v --no-header -ra -m "not live"` against the **installed wheel**, not src mode (catches packaging bugs — the anti-pattern in 28-RESEARCH.md).
  4. **publish-testpypi** (`needs: [smoke-install, test-wheel]`)
     - `environment: testpypi`
     - `permissions: { id-token: write, contents: read }` — **job level only**
     - `pypa/gh-action-pypi-publish@release/v1` with `repository-url: https://test.pypi.org/legacy/`
     - No explicit `attestations:` flag — default-on in v1.14+ (D-13).
  5. **smoke-from-testpypi** (`needs: [publish-testpypi]`)
     - `env: ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}`
     - Checkout (for `examples/hello.py`) + setup-python 3.12
     - Extracts the target version from `pyproject.toml` via `tomllib`
     - **5-try × 15s retry loop** around `pip install --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ "zeroth-core==${VERSION}"` (pitfall #4: TestPyPI indexing can lag 30–60s after publish)
     - Conditionally runs `python examples/hello.py` only if `ANTHROPIC_API_KEY` is non-empty; otherwise prints a clean `SKIP:` line (pitfall #6 — fork-PR safety).
  6. **publish-pypi** (`needs: [smoke-from-testpypi]`)
     - `environment: pypi`
     - `permissions: { id-token: write, contents: read }` — **job level only**
     - `pypa/gh-action-pypi-publish@release/v1` (default index, attestations default-on).

### `.github/workflows/verify-extras.yml` — continuous extras gate

- **Trigger:** `push` and `pull_request` on `main` branch
- **Single job `verify-extras`** with `strategy.fail-fast: false` and matrix `extra: [memory-pg, memory-chroma, memory-es, dispatch, sandbox, all]`
- Steps per matrix entry:
  1. Checkout, install uv, setup-python 3.12
  2. `uv build` to produce a local wheel
  3. Clean venv at `/tmp/venv-${{ matrix.extra }}`
  4. `pip install "${WHEEL}[${{ matrix.extra }}]"` using the resolved wheel path
  5. Per-extra import smoke via a `case` dispatch:
     - `memory-pg` → `import zeroth.core.memory.pgvector_connector`
     - `memory-chroma` → `import zeroth.core.memory.chroma_connector`
     - `memory-es` → `import zeroth.core.memory.elastic_connector` (confirmed real filename via `ls src/zeroth/core/memory/`)
     - `dispatch` → `import zeroth.core.dispatch.worker`
     - `sandbox` → `import zeroth.core.sandbox_sidecar` (empty extra still gated on resolve + base-dep import)
     - `all` → composite import of all the above

### `README.md` — surgical Install section

Added a `## Install` block immediately after the top-of-file intro paragraph and above the existing `## Why Zeroth?` section. Contents:

- `pip install zeroth-core` (base install)
- One `pip install "zeroth-core[<extra>]"` example per declared extra (`memory-pg`, `memory-chroma`, `memory-es`, `dispatch`, `sandbox`, `all`) with a one-line gloss each
- A trailing sentence listing the available extras for discoverability

No badges, no section reshuffling, no full rewrite — the full README overhaul is explicitly deferred in `28-CONTEXT.md` (Deferred list) to a later docs phase.

## Verification Results

**Task 1 (release workflow):** PASS via the full automated assertion set from the plan verify block:

```text
✓ trigger is release.types=[published]
✓ job set = {build, smoke-install, test-wheel, publish-testpypi, smoke-from-testpypi, publish-pypi}
✓ publish-testpypi.environment == 'testpypi'
✓ publish-pypi.environment == 'pypi'
✓ publish-testpypi.permissions == {id-token: write, contents: read}
✓ publish-pypi.permissions == {id-token: write, contents: read}
✓ build / smoke-install / test-wheel have NO id-token permission
✓ pypa/gh-action-pypi-publish@release/v1 present
✓ test.pypi.org/legacy present
✓ examples/hello.py present
✓ ANTHROPIC_API_KEY present
✓ tomllib (tag guard) present
```

**Task 2 (verify-extras + README):** PASS via automated assertions:

```text
✓ matrix == {memory-pg, memory-chroma, memory-es, dispatch, sandbox, all}
✓ fail-fast == False
✓ per-extra import paths present (pgvector_connector, chroma_connector,
  elastic_connector, dispatch.worker, sandbox_sidecar)
✓ README.md contains 'pip install' and 'zeroth-core['
```

## Deviations from Plan

None — plan executed exactly as written. The `<ACTUAL_ES_MODULE>` placeholder in the plan's example YAML was resolved to `elastic_connector` by `ls src/zeroth/core/memory/` before authoring the workflow (the plan called for exactly this discovery step).

**Total deviations:** 0.

## Authentication Gates

None encountered during execution. The release workflow is *designed* around an auth gate (trusted-publisher registration) but that gate is a deferred user action documented below, not an execution-time failure.

## Deferred User Action — Task 3 (`checkpoint:human-action`)

Task 3 of this plan is a `checkpoint:human-action` that cannot be automated (pypi.org and test.pypi.org publisher registration is a web-UI-only flow; GitHub repo environments are created via the GitHub Settings UI). Per the orchestrator's continuation instructions, execution did **not** block on this checkpoint — the plan is marked complete and the action is deferred to the user. **The release workflow will not actually publish until all three steps below are completed.**

### Required before first release

1. **Create GitHub environments** at https://github.com/rrrozhd/zeroth-core/settings/environments:
   - Environment `pypi` (no required reviewers, per D-11)
   - Environment `testpypi` (no required reviewers)
   - *(Optional)* add `ANTHROPIC_API_KEY` as an environment secret on `testpypi` (or at repo level) if you want the `smoke-from-testpypi` job to execute `examples/hello.py` end-to-end against a real LLM. Skipping it is fine — the workflow prints `SKIP:` and moves on.

2. **Register the trusted publisher on pypi.org** at https://pypi.org/manage/account/publishing/:
   - PyPI Project Name: `zeroth-core`
   - Owner: `rrrozhd`
   - Repository name: `zeroth-core`
   - Workflow name: `release-zeroth-core.yml`
   - Environment name: `pypi`

3. **Register the trusted publisher on test.pypi.org** at https://test.pypi.org/manage/account/publishing/ (separate registration — pitfall #3 in 28-RESEARCH.md):
   - PyPI Project Name: `zeroth-core`
   - Owner: `rrrozhd`
   - Repository name: `zeroth-core`
   - Workflow name: `release-zeroth-core.yml`
   - Environment name: `testpypi`
   - If the name is not yet owned on TestPyPI, use the "pending publisher" flow to claim it on first upload.

## Issues Encountered

None.

## Success Criteria

- [x] `.github/workflows/release-zeroth-core.yml` committed with the six-stage trusted-publisher pipeline (D-08/D-09/D-10/D-11/D-12/D-13/D-14)
- [x] Publish jobs scoped to `testpypi` and `pypi` GitHub environments respectively
- [x] `id-token: write` at JOB level only, not workflow level
- [x] `pypa/gh-action-pypi-publish@release/v1` with default-on Sigstore attestations
- [x] Tag-matches-pyproject-version guard in the build job
- [x] `test-wheel` runs pytest against the built wheel, not src mode
- [x] `smoke-from-testpypi` retries pip install to handle TestPyPI indexing lag
- [x] `smoke-from-testpypi` runs `examples/hello.py` with ANTHROPIC_API_KEY gate
- [x] `.github/workflows/verify-extras.yml` committed with a 6-entry matrix and per-extra import smokes
- [x] `README.md` install section mentions at least one extras example
- [ ] **[DEFERRED — USER ACTION]** Trusted publishers registered on pypi.org AND test.pypi.org and `pypi` / `testpypi` GitHub environments created
- [x] Phase 28 post-phase follow-up documented below

## Post-Phase Follow-Up

After Phase 28 completes and the three deferred user actions above are done, the operator triggers the first real publish by creating a GitHub Release for `v0.1.1`:

```bash
# Ensure main is at the commit you want to release
git tag -a v0.1.1 -m "zeroth-core 0.1.1: first trusted-publisher release"
git push origin v0.1.1
gh release create v0.1.1 \
  --title "zeroth-core 0.1.1" \
  --notes-file CHANGELOG.md \
  --verify-tag
```

That `release: published` event fires `release-zeroth-core.yml`. **PKG-02 / PKG-05 / PKG-06 close when that run succeeds end-to-end** (build → smoke-install → test-wheel → publish-testpypi → smoke-from-testpypi → publish-pypi, with a visible `v0.1.1` release on pypi.org/project/zeroth-core). This is not gated by the current plan's success criteria — it is a post-phase operator action.

## Next

Phase 28 plan-level work is complete. Ready for the phase verification step (`/gsd-verify-work 28`), then the post-phase release cut above. Plans 28-01, 28-02, and 28-03 together close the Phase 28 implementation deliverables modulo the deferred user actions.

## Self-Check

- [x] `.github/workflows/release-zeroth-core.yml` exists on disk
- [x] `.github/workflows/verify-extras.yml` exists on disk
- [x] `README.md` contains the `## Install` section with `zeroth-core[` extras examples
- [x] Commit `2831469` (release workflow) exists in `git log --oneline`
- [x] Commit `4a003db` (verify-extras + README) exists in `git log --oneline`
- [x] YAML validity checked for both workflow files
- [x] Memory ES connector module name confirmed via `ls src/zeroth/core/memory/` → `elastic_connector.py`

## Self-Check: PASSED
