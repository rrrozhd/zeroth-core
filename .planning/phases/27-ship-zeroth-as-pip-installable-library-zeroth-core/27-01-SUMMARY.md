---
phase: 27-ship-zeroth-as-pip-installable-library-zeroth-core
plan: 01
subsystem: "infra"
tags: ["git", "archive", "pytest", "worktree", "mirror"]
provides:
  - "Local tarball archive with self-contained git metadata snapshot"
  - "Local bare mirror with stash and detached-worktree refs preserved as archive branches"
  - "Pre-rename pytest baseline log for regression comparison"
affects: ["27-02", "27-03", "27-04", "archive", "rename", "verification"]
tech-stack:
  added: []
  patterns: ["reproducible archive scripts", "baseline-first verification", "dynamic detached-worktree preservation"]
key-files:
  created:
    - "scripts/archive_monolith.sh"
    - "scripts/verify_monolith_archive.sh"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-inventory.txt"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-preflight.txt"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-before-rename.log"
  modified:
    - ".planning/STATE.md"
    - "PROGRESS.md"
key-decisions:
  - "Built the tarball from a temporary clone-plus-overlay snapshot so the archive contains a real `.git/` directory even though the active checkout is a linked worktree."
  - "Recorded the repo's current pytest failures as the authoritative pre-rename baseline instead of trying to fix unrelated breakage inside the archive phase."
patterns-established:
  - "Archive scripts must enumerate detached worktrees dynamically rather than relying on a hard-coded count."
  - "Phase 27 verification compares post-rename results against the captured baseline, not against an assumed green suite."
requirements-completed: [ARCHIVE-01, ARCHIVE-02, RENAME-04]
duration: "7min"
completed: 2026-04-10
---

# Phase 27: ship-zeroth-as-pip-installable-library-zeroth-core Summary

**Local monolith archive layers and pre-rename regression baseline captured with dynamic worktree/stash preservation scripts**

## Performance

- **Duration:** 7 min
- **Started:** 2026-04-10T17:43:57Z
- **Completed:** 2026-04-10T17:50:00Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments

- Created repeatable archive-preflight and recovery-check scripts for the monolith snapshot.
- Preserved both stashes and both detached worktrees as named `archive/*` refs before cloning the bare mirror.
- Captured the tarball, local bare mirror, recoverability evidence, and the full pre-rename pytest baseline.

## Task Commits

1. **Task 1: Create archive-preflight automation** - `34170e0`
2. **Task 2: Capture local archive and pytest baseline** - `f02ce32`

**Plan metadata:** recorded in the summary/state progress commit that follows this file update

## Files Created/Modified

- `scripts/archive_monolith.sh` - Records live inventory, synthesizes archive refs, creates the tarball snapshot, clones the bare mirror, and writes preflight metadata.
- `scripts/verify_monolith_archive.sh` - Clones the local mirror and checks out normal, stash, and detached-worktree refs to prove recoverability.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-inventory.txt` - Captured live counts for worktrees, detached worktrees, branches, stashes, and Python subpackages.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-preflight.txt` - Recorded tarball metadata, mirror metadata, and recovery checkout logs.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/pytest-before-rename.log` - Recorded the exact pre-rename pytest baseline.

## Decisions Made

- Used a temporary clone plus rsync overlay for the tarball input so the archive contains actual git metadata rather than only the linked-worktree `.git` pointer file.
- Treated the current non-green pytest result as baseline evidence for `RENAME-04`; Phase 27 only needs to avoid introducing new regressions relative to that log.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

- The baseline pytest run is already non-green: 26 failures and 4 errors, concentrated in live scenario, service-run/thread, approval, secret-protection, and postgres-backend paths. This is a pre-existing repo state and must be compared against, not “fixed” inside the archive step.

## Next Phase Readiness

- Plan 27-02 can publish the already-created local mirror to GitHub without needing to rebuild the archive layers first.
- Plans 27-03 and 27-04 now have the required baseline artifact for post-rename comparison.

## Self-Check: PASSED
