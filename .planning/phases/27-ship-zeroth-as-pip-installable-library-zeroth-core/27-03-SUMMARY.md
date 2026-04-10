---
phase: 27-ship-zeroth-as-pip-installable-library-zeroth-core
plan: 03
subsystem: "packaging"
tags: ["namespace-package", "libcst", "rename", "pep420", "packaging"]
provides:
  - "Full `zeroth.*` to `zeroth.core.*` source-tree relocation under `src/zeroth/core/`"
  - "Repeatable LibCST codemod and Bash text-rewrite wrapper for the namespace rename"
  - "Smoke-test evidence proving the renamed layout imports as a PEP 420 namespace package"
affects: ["27-04", "packaging", "tests", "docker", "alembic"]
tech-stack:
  added: ["libcst"]
  patterns: ["git mv package relocation", "AST-preserving import rewrite", "namespace-package smoke verification"]
key-files:
  created:
    - "scripts/rename_to_zeroth_core.py"
    - "scripts/rewrite_zeroth_refs.sh"
    - "src/zeroth/core/__init__.py"
    - "tests/test_phase27_rename_scripts.py"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/rename-smoke-27-03.txt"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-27-03-targeted.txt"
  modified:
    - "pyproject.toml"
    - "uv.lock"
    - "Dockerfile"
    - "alembic.ini"
    - "README.md"
    - "docs/superpowers/specs/2026-04-10-zeroth-core-platform-split-design.md"
    - "docs/superpowers/plans/2026-04-10-zeroth-core-platform-split-plan.md"
    - "PROGRESS.md"
key-decisions:
  - "Kept the rename logic in `scripts/rename_to_zeroth_core.py` so the Python import rewrite and the metadata rewrite are both reproducible and directly testable."
  - "Preserved the old top-level storage re-exports in `src/zeroth/core/__init__.py`, but moved them under the new `zeroth.core` package so the public surface stays usable after the namespace flip."
  - "Removed the stale `src/zeroth/studio/` directory because it contained no tracked source files, only `__pycache__` output and empty folders; leaving it in place would violate the PEP 420 namespace-root requirement."
patterns-established:
  - "Future namespace/package renames should ship an AST-based codemod plus a separate explicit text-rewrite pass instead of relying on regex-only source edits."
requirements-completed: [RENAME-01, RENAME-02, RENAME-03]
duration: "11min"
completed: 2026-04-10
---

# Phase 27: ship-zeroth-as-pip-installable-library-zeroth-core Summary

**Namespace rename landed: all tracked Python code now lives under `zeroth.core.*`, packaging/runtime paths point at the new layout, and the namespace-package smoke checks pass**

## Performance

- **Duration:** 11 min
- **Started:** 2026-04-10T18:01:00Z
- **Completed:** 2026-04-10T18:12:00Z
- **Tasks:** 3
- **Files modified:** 200+

## Accomplishments

- Relocated the tracked Python source tree from `src/zeroth/*` to `src/zeroth/core/*` and enabled PEP 420 by deleting `src/zeroth/__init__.py`.
- Added a repeatable LibCST migration script plus a Bash wrapper so the import rewrite and non-Python metadata/path rewrite are reviewable and reproducible.
- Updated `pyproject.toml`, `Dockerfile`, and `alembic.ini` to the `zeroth-core` package identity and captured smoke-test evidence for the renamed layout.

## Task Commits

1. **Task 1: Add and validate rename automation** - pending commit in this session
2. **Task 2: Relocate the package tree and rewrite imports/paths** - pending commit in this session
3. **Task 3: Capture namespace smoke verification** - pending commit in this session

**Plan metadata:** recorded in the summary/state progress commit that follows this file update

## Files Created/Modified

- `scripts/rename_to_zeroth_core.py` - LibCST codemod and text-rewrite helper for the `zeroth.*` to `zeroth.core.*` migration.
- `scripts/rewrite_zeroth_refs.sh` - Checked-in Bash wrapper for the non-Python metadata/path rewrite pass.
- `src/zeroth/core/__init__.py` - New `zeroth.core` package root carrying the preserved storage re-exports.
- `pyproject.toml` - Renamed the distribution to `zeroth-core`, pointed wheel packaging at `src/zeroth/core`, and added `libcst` to the dev dependency group.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/rename-smoke-27-03.txt` - Captures the required namespace smoke commands and their outputs.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-27-03-targeted.txt` - Captures a small post-rename pytest slice that exercises real imports on the renamed layout.

## Decisions Made

- Kept the codemod and the text rewrite logic in the same Python module so the test suite could drive both behaviors before the actual tree move.
- Updated the new `zeroth.core` package root to preserve the old storage re-export surface instead of leaving it empty, which avoids unnecessary breakage for the renamed public package.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Repo reality] `src/zeroth/studio/` had no tracked Python sources**
- **Found during:** Task 1 (tree relocation)
- **Issue:** The plan assumed `src/zeroth/studio/` was a normal tracked subpackage, but the live repo only contained `__pycache__` output and empty folders there, so `git mv` had nothing real to move.
- **Fix:** Removed the stale directory so `src/zeroth/` contains only `core/`, which preserves the intended PEP 420 namespace root invariant.
- **Files modified:** filesystem cleanup only; no tracked Studio source files existed
- **Verification:** `find src/zeroth -maxdepth 1 -mindepth 1 | sort` now returns only `src/zeroth/core`
- **Committed in:** part of the task-completion commit

---

**Total deviations:** 1 auto-fixed (repo reality mismatch)
**Impact on plan:** No functional scope change. The live repo simply had no tracked `studio` package content to relocate.

## Issues Encountered

- The initial bulk `git mv` surfaced the stale `studio/` tree because Git cannot move an empty directory. The fix was to remove the pycache-only directory and continue with the namespace-root cleanup.

## Next Phase Readiness

- Plan 27-04 can now focus on docstring coverage, CI/docstring gates, and the post-rename full-suite regression comparison instead of structural package work.
- The namespace smoke checks and the stale-import grep are now in place as reusable verification commands for the final post-rename pass.

## Self-Check: PASSED
