---
phase: 27-ship-zeroth-as-pip-installable-library-zeroth-core
plan: 04
subsystem: "verification"
tags: ["ci", "interrogate", "ruff", "pytest", "codemod", "regression"]
provides:
  - "Checked-in docstring tooling and the repo's first CI verification workflow"
  - "Post-rename proof that no new FAILED/ERROR/SKIPPED entries were introduced versus the pre-rename baseline"
  - "Hardened rename codemod coverage for `live_scenarios/`, source-path literals, and idempotent reruns"
affects: ["phase-28", "ci", "packaging", "tests", "live_scenarios"]
tech-stack:
  added: ["interrogate"]
  patterns:
    - "baseline-vs-postchange pytest diffing"
    - "idempotent codemod verification"
    - "ruff + interrogate + pytest CI gate"
key-files:
  created:
    - ".github/workflows/ci.yml"
    - "tests/test_phase27_docstring_tooling.py"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/27-04-SUMMARY.md"
  modified:
    - "pyproject.toml"
    - "scripts/rename_to_zeroth_core.py"
    - "tests/test_phase27_rename_scripts.py"
    - "live_scenarios/research_audit/bootstrap.py"
    - "live_scenarios/research_audit/run_server.py"
    - "tests/live_scenarios/test_research_audit.py"
    - "tests/test_smoke.py"
    - "PROGRESS.md"
    - ".planning/STATE.md"
    - ".planning/ROADMAP.md"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/interrogate-after.txt"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-after-rename.log"
key-decisions:
  - "Kept the docstring gate enforceable in `pyproject.toml` and CI rather than doing a one-off local measurement; `interrogate` now runs in the same command form locally and in GitHub Actions."
  - "Fixed the rename regressions through the checked-in codemod instead of hand-editing `live_scenarios/`, so reruns of the migration stay reproducible and test-driven."
  - "Accepted the increased collected-test count because the only delta was the five new Phase 27 verification tests; failure, error, and skip sets remained unchanged from baseline."
patterns-established:
  - "Namespace migrations must prove idempotence before being rerun over an already-renamed tree."
  - "Large structural refactors should preserve a baseline pytest log and compare exact `FAILED`/`ERROR`/`SKIPPED` sets after the change."
requirements-completed: [RENAME-04, RENAME-05]
duration: "20min"
completed: 2026-04-10
---

# Phase 27: ship-zeroth-as-pip-installable-library-zeroth-core Summary

**Verification gates are checked in, docstring coverage is enforced at 90.1%, and the renamed `zeroth.core.*` layout proved it introduced no new failures, errors, or skips relative to the pre-rename baseline**

## Performance

- **Duration:** 20 min
- **Started:** 2026-04-10T18:12:00Z
- **Completed:** 2026-04-10T18:32:00Z
- **Tasks:** 3
- **Verification runs:** interrogate, Ruff, targeted pytest, full pytest baseline comparison

## Accomplishments

- Added `interrogate` plus Google-style Ruff pydocstyle settings to `pyproject.toml` and created `.github/workflows/ci.yml` to run `uv sync --all-groups`, `ruff`, `pytest`, and `interrogate`.
- Captured the docstring baseline and final coverage artifact, with the public `zeroth.core.*` surface holding at **90.1%** without bulk filler docstrings.
- Fixed the two rename-induced regressions by hardening `scripts/rename_to_zeroth_core.py` to scan `live_scenarios/`, rewrite `src/zeroth/...` literals, and remain idempotent on rerun.
- Refreshed `pytest-after-rename.log` and diffed it against `pytest-before-rename.log`, confirming zero new `FAILED`, `ERROR`, or `SKIPPED` lines after the namespace migration.

## Verification Evidence

- `uv run interrogate -v src/zeroth/core` → `RESULT: PASSED (minimum: 90.0%, actual: 90.1%)`
- `uv run ruff check src tests` → passed
- `uv run python -c "from zeroth.core.orchestrator import RuntimeOrchestrator; print(RuntimeOrchestrator.__name__)"` → `RuntimeOrchestrator`
- `uv run pytest -v --no-header -ra` → `26 failed, 640 passed, 8 deselected, 1 warning, 4 errors`
- Baseline comparison:
  - Before: `26 failed, 635 passed, 8 deselected, 1 warning, 4 errors`
  - After: `26 failed, 640 passed, 8 deselected, 1 warning, 4 errors`
  - Difference in collected tests: +5, explained by the new Phase 27 verification tests
  - Difference in `FAILED`/`ERROR`/`SKIPPED` sets: none

## Deviations from Plan

### Auto-fixed Issues

**1. The rename codemod originally missed `live_scenarios/`**
- **Found during:** Task 3 regression repair
- **Issue:** The checked-in migration only scanned `src/`, `tests/`, `apps/`, and `scripts/`, leaving stale imports and repo-path literals under `live_scenarios/`.
- **Fix:** Expanded the codemod roots, added regression coverage, and reran the migration so the fix came from the same tool that performed the rename.

**2. The codemod was not idempotent on already-renamed imports**
- **Found during:** Task 3 regression repair
- **Issue:** A rerun rewrote `import zeroth.core` into `import zeroth.core.core`.
- **Fix:** Normalized duplicate `.core` segments, added red-green coverage, and excluded the codemod's own regression test file from future rewrites.

**3. The docstring baseline already cleared the 90% target**
- **Found during:** Task 2 baseline measurement
- **Issue:** The plan assumed a docstring-writing pass would be needed.
- **Fix:** Kept the scope on enforceable tooling and verification, because the real repo already measured at 90.1%.

## Next Phase Readiness

- Phase 27 is complete and leaves a stable `zeroth-core` namespace/package foundation for PyPI work.
- Phase 28 can start from a checked-in CI/docstring gate and a preserved post-rename baseline artifact set.
- Remaining blockers for Phase 28 are external publication prerequisites: the missing Regulus GitHub remote and PyPI trusted-publisher setup.

## Self-Check: PASSED
