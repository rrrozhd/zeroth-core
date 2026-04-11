---
phase: 28-pypi-publishing-econ-instrumentation-sdk-zeroth-core
verified: 2026-04-11T00:00:00Z
status: human_needed
score: 5/5 must-haves verified in-repo; SC4 + SC5 each have an irreducible human/operator step before they close end-to-end
overrides_applied: 0
human_verification:
  - test: "Register trusted publisher on pypi.org for zeroth-core"
    expected: "pypi.org/manage/account/publishing/ shows a publisher entry for owner=rrrozhd, repo=zeroth-core, workflow=release-zeroth-core.yml, environment=pypi"
    why_human: "PyPI web UI only — no CLI or API. Documented in 28-03 plan Task 3 and <known_deferred>."
  - test: "Register trusted publisher on test.pypi.org for zeroth-core"
    expected: "test.pypi.org/manage/account/publishing/ shows a publisher entry for owner=rrrozhd, repo=zeroth-core, workflow=release-zeroth-core.yml, environment=testpypi"
    why_human: "TestPyPI is a separate index with its own UI; same CLI-unavailability constraint."
  - test: "Create GitHub environments pypi and testpypi in rrrozhd/zeroth-core settings"
    expected: "github.com/rrrozhd/zeroth-core/settings/environments lists both envs (no required reviewers per D-11)"
    why_human: "GitHub repo settings UI — trivially scriptable with gh api but not part of this phase."
  - test: "Cut first GitHub Release v0.1.1 to trigger the release workflow end-to-end"
    expected: "gh release create v0.1.1 --title 'zeroth-core 0.1.1' --notes-file CHANGELOG.md fires release-zeroth-core.yml, all six jobs pass, and pypi.org/project/zeroth-core/0.1.1 becomes live with Sigstore attestations"
    why_human: "Depends on the three manual registrations above + operator intent. Success closes SC4 and SC5 end-to-end; documented as post-phase follow-up in 28-03-SUMMARY."
  - test: "(optional) Run examples/hello.py with ANTHROPIC_API_KEY set locally"
    expected: "Prints a one-sentence LLM greeting to stdout, exits 0"
    why_human: "Requires a real Anthropic API key; cost-bearing live LLM call not appropriate for automated verifier."
---

# Phase 28: PyPI Publishing (econ-instrumentation-sdk + zeroth-core) — Verification Report

**Phase Goal:** Both `econ-instrumentation-sdk` and `zeroth-core` are published to PyPI via GitHub Actions trusted publisher, a clean-venv install of `zeroth-core[all]` succeeds, and every declared optional extra is verified installable.

**Verified:** 2026-04-11
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

Phase 28 delivered every artifact the repository can produce on its own. The two success criteria that reach outside the repo (SC4 and SC5) require a handful of one-time operator actions on pypi.org, test.pypi.org, and the GitHub release UI — all explicitly called out as deferred in the phase plan and in the user-supplied `<known_deferred>` block.

### Observable Truths (per ROADMAP Success Criteria)

| #   | Truth                                                                                                                                    | Status              | Evidence                                                                                                                                                                                                                                                       |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SC1 | `econ-instrumentation-sdk` live on PyPI at a stable version; `zeroth-core` consumes it via a PyPI constraint, not a local file path     | ✓ VERIFIED          | `pyproject.toml` line 38 pins `econ-instrumentation-sdk>=0.1.1` in base `[project].dependencies`. No `file://` or path deps anywhere. Commit `78c2076` (pre-phase) swapped the local path to PyPI. `uv sync --all-extras --all-groups` resolves cleanly.     |
| SC2 | `pyproject.toml` declares the six extras `[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]` and each resolves | ✓ VERIFIED          | `pyproject.toml` `[project.optional-dependencies]` declares all six names verbatim. `sandbox = []` (empty marker per resolved Q2). `all` uses self-referencing `zeroth-core[...]` entries. `.github/workflows/verify-extras.yml` matrix covers all six. Local `uv run python -c "import zeroth.core.memory.pgvector_connector, zeroth.core.memory.chroma_connector, zeroth.core.memory.elastic_connector, zeroth.core.dispatch.worker, zeroth.core.sandbox_sidecar"` prints "all imports OK". |
| SC3 | Repo root has `CHANGELOG.md` (keepachangelog), `LICENSE`, `CONTRIBUTING.md`                                                             | ✓ VERIFIED          | `LICENSE` is 201 lines, canonical Apache-2.0, contains "Apache License / Version 2.0, January 2004 / END OF TERMS AND CONDITIONS / APPENDIX". `CHANGELOG.md` uses keepachangelog 1.1.0 format with seeded `[0.1.1] - 2026-04-11` entry and reference links. `CONTRIBUTING.md` has Development setup, Running the example, PR conventions, Filing issues, License (links to `LICENSE`), Code of conduct placeholder. |
| SC4 | PyPI releases published by GitHub Actions trusted publisher (OIDC); no long-lived tokens in repo or CI                                  | ⚠ HUMAN_NEEDED     | Workflow `.github/workflows/release-zeroth-core.yml` is fully correct: `release.published` trigger, six-stage pipeline, job-scoped `id-token: write` only on publish-testpypi / publish-pypi, separate `testpypi` and `pypi` environments, `pypa/gh-action-pypi-publish@release/v1` (Sigstore default-on), tomllib tag-version guard, TestPyPI retry loop, env-gated `examples/hello.py` smoke. Zero API tokens in the repo or workflow. **Actual trusted-publisher registration on pypi.org + test.pypi.org and creation of the `pypi` / `testpypi` GitHub environments are web-UI-only and deferred to the user** (28-03 Task 3 checkpoint; `<known_deferred>`). No first release has been cut yet. |
| SC5 | Clean-venv `pip install zeroth-core` + running Getting Started hello example produces working output end-to-end                          | ⚠ HUMAN_NEEDED     | Plan 28-01 Task 2 smoke-tested `pip install dist/zeroth_core-0.1.1-py3-none-any.whl` in a clean `/tmp/zc-smoke` venv — `import zeroth.core` succeeded and `py.typed` was found on disk. `[all]` extras installed and imported cleanly. `examples/hello.py` skip path verified: `env -u ANTHROPIC_API_KEY python3 examples/hello.py` prints "SKIP: ..." and exits 0. **End-to-end against the real PyPI-hosted `zeroth-core==0.1.1` waits for the first `gh release create v0.1.1`** (which itself waits for SC4's trusted-publisher registration). Once the release fires, the workflow's own `smoke-from-testpypi` step executes `examples/hello.py` against the published wheel in a clean venv — that is the operational end-to-end gate. |

**Score:** 5/5 in-repo deliverables verified. SC4 and SC5 each have one operator-only closure step (trusted-publisher registration + cutting `v0.1.1`), both explicitly flagged as deferred before verification began.

### Required Artifacts

| Artifact                                              | Expected                                                           | Status     | Details                                                                                                                                                                                                                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pyproject.toml`                                      | 0.1.1, Apache-2.0, 6 extras, urls, classifiers, py.typed-inclusive | ✓ VERIFIED | `version = "0.1.1"`, `license = "Apache-2.0"`, all six extras verbatim, `[project.urls]` has Homepage/Source/Issues/Changelog, 8 classifiers, `[build-system].requires = ["hatchling>=1.27"]`, wheel target `packages = ["src/zeroth"]` (intentional — pitfall #2). |
| `src/zeroth/core/py.typed`                            | PEP 561 marker                                                     | ✓ VERIFIED | Empty file exists on disk; present in `dist/zeroth_core-0.1.1-py3-none-any.whl` as `zeroth/core/py.typed`.                                                                                                                                                          |
| `dist/zeroth_core-0.1.1-py3-none-any.whl`             | Built wheel, correct namespace layout                              | ✓ VERIFIED | Wheel exists. `unzip -l` shows entries rooted at `zeroth/core/...`, no `zeroth/__init__.py` (namespace pkg), `zeroth/core/py.typed` present, no top-level `core/` entries.                                                                                         |
| `LICENSE`                                             | Canonical Apache-2.0, ~175+ lines                                  | ✓ VERIFIED | 201 lines, canonical text from apache.org, APPENDIX placeholders preserved.                                                                                                                                                                                         |
| `CHANGELOG.md`                                        | keepachangelog 1.1.0 with [0.1.1]                                  | ✓ VERIFIED | 43 lines, header references Keep a Changelog 1.1.0 + SemVer 2.0.0, `[Unreleased]` + `[0.1.1] - 2026-04-11` sections with Added/Changed subsections, reference links at bottom.                                                                                      |
| `CONTRIBUTING.md`                                     | Dev setup, PR conventions, issues, LICENSE link                    | ✓ VERIFIED | 78 lines, all six required sections present, `[LICENSE](LICENSE)` relative link present, `uv sync` and `examples/hello.py` referenced.                                                                                                                             |
| `examples/hello.py`                                   | PKG-06 fixture, env-gated ANTHROPIC_API_KEY                        | ✓ VERIFIED | 66 lines. Imports `zeroth.core`. Skip path: `env -u ANTHROPIC_API_KEY python3 examples/hello.py` → exit 0, stderr "SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM". Uses `litellm.completion(model="anthropic/claude-3-haiku-20240307", ...)` for the real-LLM path. |
| `.github/workflows/release-zeroth-core.yml`           | Six-stage trusted-publisher pipeline                               | ✓ VERIFIED | YAML valid, 6 jobs (build, smoke-install, test-wheel, publish-testpypi, smoke-from-testpypi, publish-pypi), correct `needs:` chain, top-level `permissions: contents: read`, job-level `id-token: write` only on the two publish jobs, separate `testpypi` / `pypi` environments, `pypa/gh-action-pypi-publish@release/v1`, tomllib tag-match guard, retry loop, `examples/hello.py` referenced with `ANTHROPIC_API_KEY` gate. |
| `.github/workflows/verify-extras.yml`                 | 6-entry matrix, per-extra import smokes                            | ✓ VERIFIED | YAML valid. Matrix `[memory-pg, memory-chroma, memory-es, dispatch, sandbox, all]`, `fail-fast: false`. Per-extra import steps reference real module paths (`pgvector_connector`, `chroma_connector`, `elastic_connector`, `dispatch.worker`, `sandbox_sidecar`) — all exist under `src/zeroth/core/`. |
| `README.md`                                           | Install section with at least one extras example                   | ✓ VERIFIED | Lines 12–23 contain `pip install zeroth-core` base install plus one `pip install "zeroth-core[<extra>]"` line for each of the six extras.                                                                                                                            |
| `.planning/STATE.md`                                  | Regulus blocker removed, trusted-publisher entry reworded          | ✓ VERIFIED | (per 28-02 summary; verifier accepts plan-level evidence — the reconciliation is a surgical edit, not a goal-critical artifact)                                                                                                                                     |

### Key Link Verification

| From                                              | To                                              | Via                                            | Status     | Details                                                                                                                                                             |
| ------------------------------------------------- | ----------------------------------------------- | ---------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pyproject.toml [optional-dependencies].all`      | memory-pg/memory-chroma/memory-es/dispatch/sandbox | self-referencing `zeroth-core[...]` entries    | ✓ WIRED    | All five self-refs present in the `all` array.                                                                                                                       |
| `pyproject.toml` hatch wheel target               | `zeroth/core/` layout in wheel                  | hatchling `packages = ["src/zeroth"]`          | ✓ WIRED    | Verified via `unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl`.                                                                                                    |
| `CONTRIBUTING.md`                                 | `LICENSE`                                       | markdown `[LICENSE](LICENSE)`                  | ✓ WIRED    | Present at line 69.                                                                                                                                                  |
| `examples/hello.py`                               | `zeroth.core`                                   | `import zeroth.core`                           | ✓ WIRED    | Line 40.                                                                                                                                                             |
| `CHANGELOG.md [0.1.1]`                            | `pyproject.toml [project].version`              | matching version number                       | ✓ WIRED    | Both are `0.1.1`.                                                                                                                                                    |
| `release-zeroth-core.yml publish-testpypi`        | test.pypi.org trusted publisher                 | OIDC `id-token: write` + `environment: testpypi` | ⚠ PENDING  | Workflow side is correct. PyPI side (trusted-publisher registration + GitHub environment `testpypi`) is deferred user action.                                       |
| `release-zeroth-core.yml publish-pypi`            | pypi.org trusted publisher                      | OIDC `id-token: write` + `environment: pypi`   | ⚠ PENDING  | Workflow side correct. PyPI side deferred user action.                                                                                                               |
| `verify-extras.yml matrix`                        | each declared extra                             | clean-venv pip install + module import         | ✓ WIRED    | Matrix keys exactly match `pyproject.toml [optional-dependencies]` keys. Import targets exist on disk.                                                               |
| `smoke-from-testpypi` job                         | `examples/hello.py`                             | subprocess + `ANTHROPIC_API_KEY` gate          | ✓ WIRED    | Job references `python examples/hello.py` under an `if [ -n "${ANTHROPIC_API_KEY:-}" ]` guard.                                                                       |

### Data-Flow Trace (Level 4)

Not applicable to this phase — deliverables are build/packaging/CI configuration and metadata files, not dynamic-data-rendering components.

### Behavioral Spot-Checks

| Behavior                                                                                     | Command                                                         | Result                                                                                                                                     | Status  |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------- |
| `examples/hello.py` skip path exits 0 with SKIP message when `ANTHROPIC_API_KEY` is unset    | `env -u ANTHROPIC_API_KEY python3 examples/hello.py`           | `SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM` → exit 0                                                         | ✓ PASS  |
| `pyproject.toml` parses and has required fields                                              | `uv run python` + `tomllib.load` + asserts                      | version=0.1.1, license=Apache-2.0, extras={all,dispatch,memory-chroma,memory-es,memory-pg,sandbox}, sandbox=[], all self-refs, urls x4, econ-sdk>=0.1.1 pinned, no psycopg/chromadb in base | ✓ PASS  |
| Release workflow YAML parses and has correct job structure                                   | `yaml.safe_load` + asserts on jobs/environments/permissions     | 6 jobs present; publish-testpypi env=testpypi; publish-pypi env=pypi; both publish jobs have `{id-token: write, contents: read}`; build has no id-token; top-level perms = `{contents: read}` | ✓ PASS  |
| verify-extras workflow YAML parses with 6-entry matrix and fail-fast disabled                | `yaml.safe_load`                                                | matrix=[all,dispatch,memory-chroma,memory-es,memory-pg,sandbox]; fail-fast=False                                                          | ✓ PASS  |
| Every per-extra import target actually exists under `src/zeroth/core/`                       | `uv run python -c "import ... all five modules"`                | "all imports OK"                                                                                                                            | ✓ PASS  |
| Wheel layout is correct (`zeroth/core/` rooted, py.typed present, no top-level __init__.py)  | `unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl`              | `zeroth/core/py.typed` present; 20+ entries under `zeroth/core/`; no `zeroth/__init__.py` line                                            | ✓ PASS  |
| LICENSE contains canonical Apache-2.0 markers                                                 | `grep` for "Apache License" / "Version 2.0, January 2004" / "END OF TERMS AND CONDITIONS" / "APPENDIX" | all four markers found at expected lines (1, 2, 176, 178)                                                                                  | ✓ PASS  |
| README.md install section mentions extras                                                    | `grep -n "zeroth-core\["` README.md                             | Six lines found (one per extra)                                                                                                            | ✓ PASS  |
| Actual trusted-publisher release to PyPI                                                      | `gh release create v0.1.1` + workflow run                       | not yet attempted — requires the deferred manual registrations first                                                                       | ? SKIP — routed to human_verification |

### Requirements Coverage

| Requirement | Source Plan(s)        | Description                                                                                  | Status        | Evidence                                                                                                                                                                         |
| ----------- | --------------------- | -------------------------------------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PKG-01      | 28-01                 | econ-instrumentation-sdk published to PyPI, replace local path dep                           | ✓ SATISFIED   | econ-sdk 0.1.1 has been on PyPI since 2026-04 (commit `78c2076` switched the dep). `pyproject.toml` line 38 pins `econ-instrumentation-sdk>=0.1.1` from PyPI (no file://).     |
| PKG-02      | 28-01, 28-03          | zeroth-core pip-installable from PyPI, clean-venv install works                              | ⚠ PARTIAL     | Local clean-venv install from built wheel succeeds. End-to-end "from real PyPI at 0.1.1" requires the first trusted-publisher release, which is gated on the manual PyPI steps. |
| PKG-03      | 28-01, 28-03          | Six extras declared and each verifiably installable                                          | ✓ SATISFIED   | `pyproject.toml` declares all six verbatim. `verify-extras.yml` is the continuous CI gate and will run on the next push/PR to main. Local imports verified.                     |
| PKG-04      | 28-02                 | LICENSE, CHANGELOG.md, CONTRIBUTING.md at repo root                                           | ✓ SATISFIED   | All three present, correct formats, canonical Apache-2.0 text.                                                                                                                   |
| PKG-05      | 28-03                 | Trusted-publisher OIDC release, no long-lived tokens                                          | ⚠ PARTIAL     | Workflow fully correct and token-free. Actual first trusted-publisher release pending the manual PyPI registration + first `gh release create v0.1.1`.                          |
| PKG-06      | 28-02, 28-03          | Clean-venv install + hello example works end-to-end                                           | ⚠ PARTIAL     | `examples/hello.py` shipped, skip path verified, clean-venv install verified locally. End-to-end "install from pypi.org then run example" is gated on PKG-05.                   |

All three PARTIAL requirements share the same root cause: the irreducible manual PyPI-UI + GitHub-Release operator step. This is documented as deferred before verification.

### Anti-Patterns Found

Focused scan over files modified in Phase 28:

| File                                              | Line | Pattern                                                                      | Severity  | Impact                                                                                                                                                                                             |
| ------------------------------------------------- | ---- | ---------------------------------------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CHANGELOG.md`                                    | 38–40 | `## Changed` entry claims "Hatchling wheel target corrected to `src/zeroth/core`" | ℹ Info    | Minor documentation inaccuracy — Plan 28-01's resolved Q3 explicitly **kept** `packages = ["src/zeroth"]` (it was already correct). The CHANGELOG entry misrepresents the actual change. Does not affect publishability or tooling. Recommend clarifying before cutting `v0.1.1` release notes from CHANGELOG. |
| `sandbox` extra                                   | —    | Empty extra `sandbox = []`                                                   | ℹ Info    | Intentional marker (resolved Q2). `sandbox_sidecar` shells out to docker CLI and has no Python runtime deps beyond base. Documented in CHANGELOG and plan. Not a stub.                             |
| `examples/hello.py`                               | 42–47 | litellm direct-call fallback instead of full orchestrator graph              | ℹ Info    | Intentional and pre-approved in 28-02 plan `<interfaces>`. Phase 30 will replace with a full walkthrough. Graph bootstrap would be disproportionate to a 30-line example.                          |

No blockers or warnings. One informational inconsistency in CHANGELOG wording recommended for cleanup.

### Human Verification Required

See `human_verification` frontmatter above. Summary:

1. **Register trusted publisher on pypi.org** for owner=rrrozhd, repo=zeroth-core, workflow=release-zeroth-core.yml, environment=pypi (web UI only).
2. **Register trusted publisher on test.pypi.org** with environment=testpypi (separate index, separate registration).
3. **Create GitHub environments `pypi` and `testpypi`** at github.com/rrrozhd/zeroth-core/settings/environments (no required reviewers).
4. **Cut `v0.1.1` GitHub Release** (`gh release create v0.1.1 --title 'zeroth-core 0.1.1' --notes-file CHANGELOG.md --verify-tag`) — this fires `release-zeroth-core.yml` and exercises the six-stage pipeline end-to-end. When that run is green, SC4 and SC5 (and PKG-02/PKG-05/PKG-06) close.
5. **(optional)** Run `examples/hello.py` locally with `ANTHROPIC_API_KEY` set to confirm the real-LLM path.

### Gaps Summary

**No in-repo gaps found.** Every deliverable Phase 28 owns inside the repository — `pyproject.toml` restructure, six extras, Apache-2.0 SPDX license, py.typed marker, LICENSE/CHANGELOG/CONTRIBUTING, `examples/hello.py`, both workflows, README install section, STATE.md reconciliation — is present, correct, and verified. The wheel builds, the skip path works, the YAML is valid, and every import referenced by the workflows resolves against real code on disk.

Two success criteria (SC4 trusted-publisher-via-GHA and SC5 end-to-end PyPI → hello example) cannot fully close until the operator performs the web-UI-only PyPI trusted-publisher registrations and cuts the first `v0.1.1` GitHub Release. These steps were explicitly called out as deferred before verification began (`<known_deferred>`) and are tracked in the `human_verification` list above rather than as code-level gaps.

One minor documentation inaccuracy in `CHANGELOG.md` (line 38–40 claims the hatchling wheel target was "corrected to `src/zeroth/core`" when Plan 28-01 in fact kept `["src/zeroth"]` after verifying it was already correct) is flagged as informational — recommend editing before the CHANGELOG is consumed as release notes for `v0.1.1`.

---

*Verified: 2026-04-11*
*Verifier: Claude (gsd-verifier)*
