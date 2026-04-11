---
phase: 28
plan: 02
subsystem: packaging
tags:
  - oss-metadata
  - license
  - changelog
  - contributing
  - examples
  - state-reconciliation
requires:
  - Phase 27 namespace rename to zeroth.core.*
  - econ-instrumentation-sdk 0.1.1 already on PyPI (commit 78c2076)
provides:
  - Canonical Apache-2.0 LICENSE at repo root
  - keepachangelog 1.1.0 CHANGELOG.md with seeded 0.1.1 entry
  - CONTRIBUTING.md dev setup / PR conventions / issues / license guide
  - examples/hello.py PKG-06 acceptance fixture (ANTHROPIC_API_KEY-gated)
  - Reconciled .planning/STATE.md blockers (D-25)
affects:
  - Plan 28-03 release workflow (executes examples/hello.py as clean-venv smoke test)
tech-stack:
  added:
    - litellm direct-call pattern (already a base dep, no new deps introduced)
  patterns:
    - Env-gated example with graceful skip for fork-PR CI safety
key-files:
  created:
    - LICENSE
    - CHANGELOG.md
    - CONTRIBUTING.md
    - examples/hello.py
  modified:
    - .planning/STATE.md
key-decisions:
  - Used litellm fallback pattern for hello.py instead of a full orchestrator graph — keeps the fixture ~70 lines and avoids service bootstrap; Phase 30 will replace with a real graph walkthrough
  - Preserved the canonical Apache-2.0 APPENDIX placeholders verbatim (per D-15) rather than filling in copyright holder — NOTICE file belongs in a later OSS hardening phase
  - Seeded CHANGELOG as 0.1.1 (not 0.1.0) to match the already-pinned econ-sdk cadence and the version that Plan 28-01 will publish
requirements-completed:
  - PKG-04
  - PKG-06
duration: ~10 min
completed: 2026-04-11
---

# Phase 28 Plan 02: Repo Metadata & Hello Example Summary

OSS-grade metadata (LICENSE, CHANGELOG, CONTRIBUTING) and the PKG-06 clean-venv acceptance fixture (examples/hello.py) landed at the repo root; `.planning/STATE.md` blockers reconciled per D-25 to drop the stale Regulus remote entry and reword the trusted-publisher todo to `zeroth-core`-only covering both pypi.org and test.pypi.org.

## Execution Overview

- **Duration:** ~10 minutes
- **Tasks completed:** 2 / 2
- **Files created:** 4 (LICENSE, CHANGELOG.md, CONTRIBUTING.md, examples/hello.py)
- **Files modified:** 1 (.planning/STATE.md)
- **Commits:** 2 task commits (474afd6, cc833ee)

## What Was Built

### LICENSE (201 lines)

Full canonical Apache-2.0 text, verbatim from https://www.apache.org/licenses/LICENSE-2.0.txt. Includes TERMS AND CONDITIONS (sections 1–9), END OF TERMS AND CONDITIONS marker, and the APPENDIX with `[yyyy] [name of copyright owner]` placeholders preserved (per D-15 — NOTICE file and filled-in attribution are out of scope for Phase 28).

### CHANGELOG.md

keepachangelog 1.1.0 format with:

- Standard header referencing Keep a Changelog 1.1.0 and SemVer 2.0.0
- Empty `## [Unreleased]` section for future work
- Seeded `## [0.1.1] - 2026-04-11` entry with `### Added` (LICENSE, CHANGELOG, CONTRIBUTING, py.typed, [project.urls], classifiers, examples/hello.py, optional extras, release workflow + Sigstore) and `### Changed` (dep carve-out, hatchling>=1.27 for PEP 639, wheel target fix)
- Reference links for `[Unreleased]` and `[0.1.1]` pointing at the `rrrozhd/zeroth-core` GitHub repo

No retrospective history — first public changelog per D-16.

### CONTRIBUTING.md

~85 lines covering, in order:

1. **Development setup** — `git clone`, `uv sync --all-extras --all-groups`, `uv run pytest -v`, `uv run ruff check src tests`, `uv run ruff format src`
2. **Running the example** — `python examples/hello.py` with note about the `ANTHROPIC_API_KEY` gate
3. **Pull request conventions** — conventional commit format (`type(scope): subject`), branch naming, linked issues, test expectations, lint/format, Google-style docstrings
4. **Filing issues** — link to `https://github.com/rrrozhd/zeroth-core/issues`, required repro data
5. **License** — relative link `[LICENSE](LICENSE)`, one-sentence contributor licensing statement
6. **Code of conduct** — placeholder noting a formal CoC will land in a future phase

### examples/hello.py (~70 lines)

**Approach chosen: litellm direct-call fallback (documented in plan `<interfaces>` as the explicit fallback).**

Rationale: the zeroth.core orchestrator requires service bootstrap (Settings, SecretsProvider, DI wiring) that is disproportionate to a 30-line example. litellm is already a base dep of `zeroth-core`, so the fixture:

1. Prints `SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM` to stderr and exits 0 when the key is absent (pitfall #6 — fork-PR CI safety)
2. `import zeroth.core` as the PKG-06 import-smoke guarantee
3. Calls `litellm.completion(model="anthropic/claude-3-haiku-20240307", ...)` with a minimal prompt
4. Prints the response content to stdout

A `# Phase 30 will replace this with a full graph walkthrough.` comment documents the fallback choice for future readers.

**Skip-path verification output:**

```
$ env -u ANTHROPIC_API_KEY python3 examples/hello.py
SKIP: set ANTHROPIC_API_KEY to run examples/hello.py against a real LLM
exit=0
```

### .planning/STATE.md Reconciliation (D-25)

**Before (### Blockers/Concerns):**

1. "Regulus has no GitHub remote yet — blocks publishing `econ-instrumentation-sdk`..."
2. "PyPI trusted publisher setup for `zeroth-core` and `econ-instrumentation-sdk` requires manual user action..."
3. "Local parent directory `/Users/dondoe/coding/zeroth/` needs to be renamed..."

**After:**

1. ~~(removed — stale; econ-sdk 0.1.1 has been on PyPI since commit 78c2076)~~
2. "PyPI trusted-publisher setup for `zeroth-core` requires manual user action — must register the publisher on pypi.org (environment `pypi`) AND test.pypi.org (environment `testpypi`) separately. econ-instrumentation-sdk publishing lives in the Regulus repo and is out of scope for Phase 28."
3. "Local parent directory `/Users/dondoe/coding/zeroth/` needs to be renamed..." (unchanged)

**Before (### Pending Todos):**

- "Plan and execute Phase 28 publication work..."
- "Configure the missing Regulus GitHub remote..."
- "Complete the manual PyPI trusted-publisher setup for both packages"

**After:**

- "Plan and execute Phase 28 publication work..." (unchanged)
- ~~(removed — out of scope)~~
- "Complete the manual PyPI trusted-publisher setup for zeroth-core on pypi.org AND test.pypi.org (two separate registrations)"

Also bumped `last_updated: "2026-04-11T00:00:00Z"`. No other sections touched.

## Verification Results

- Task 1 automated verify — PASS (canonical Apache-2.0 markers, 201 lines, keepachangelog elements, CONTRIBUTING sections all present)
- Task 2 automated verify — PASS (hello.py exists, contains required markers, skip path returns 0 with SKIP in stderr, STATE.md has stale Regulus line removed, test.pypi.org present, date bumped)
- Overall: `ls LICENSE CHANGELOG.md CONTRIBUTING.md examples/hello.py` — all four present
- `grep -c "Regulus has no GitHub remote" .planning/STATE.md` — 0 (as required)
- No `src/zeroth/` source code modified (metadata + fixture only, per success criteria)

## Deviations from Plan

None — plan executed exactly as written. The litellm fallback pattern was pre-approved in the plan's `<interfaces>` block as the explicit fallback when the orchestrator API is too heavy for a short example; choosing it was not a deviation.

## Authentication Gates

None encountered during execution. The `examples/hello.py` fixture deliberately treats the missing `ANTHROPIC_API_KEY` as a graceful skip (not an auth gate) so CI and fork PRs do not break.

## Commits

| Task | Hash    | Message |
|------|---------|---------|
| 1    | 474afd6 | docs(28-02): add LICENSE, CHANGELOG, CONTRIBUTING for PyPI release |
| 2    | cc833ee | feat(28-02): add examples/hello.py fixture and reconcile STATE.md blockers |

## Next

Ready for Plan 28-03 (release workflow + extras verification), which will:

- Wire `.github/workflows/release-zeroth-core.yml` with the staged build → smoke-install → tests → publish-testpypi → smoke-install-from-testpypi → publish-pypi pipeline
- Execute `examples/hello.py` as the clean-venv smoke test (consumes this plan's output)
- Verify every extra carved out by Plan 28-01 installs cleanly

Plan 28-02 is complete; Plans 28-01 and 28-02 were designed to run in parallel (Wave 1, no cross-dependencies).

## Self-Check

- [x] LICENSE exists at repo root (201 lines, canonical Apache-2.0)
- [x] CHANGELOG.md exists at repo root (keepachangelog 1.1.0, `[0.1.1] - 2026-04-11`)
- [x] CONTRIBUTING.md exists at repo root (dev setup, PR conventions, LICENSE link)
- [x] examples/hello.py exists and skip path returns exit 0
- [x] .planning/STATE.md Regulus blocker removed, trusted-publisher entry mentions test.pypi.org
- [x] Commits 474afd6 and cc833ee exist in `git log --oneline`

## Self-Check: PASSED
