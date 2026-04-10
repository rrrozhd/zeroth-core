---
phase: 27-ship-zeroth-as-pip-installable-library-zeroth-core
plan: 02
subsystem: "infra"
tags: ["github", "archive", "git", "mirror", "gh"]
provides:
  - "Public GitHub archive repo for the monolith mirror"
  - "README archive banner and matching repository description"
  - "Remote recovery evidence for normal, stash, and detached-worktree refs"
affects: ["27-03", "27-04", "archive", "migration"]
tech-stack:
  added: []
  patterns: ["remote archive publication", "temporary unarchive for mirror sync", "remote recoverability verification"]
key-files:
  created:
    - "scripts/publish_monolith_archive.sh"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-github-publish.txt"
    - ".planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-recovery-test.txt"
  modified:
    - "PROGRESS.md"
key-decisions:
  - "Treated the pre-existing `rrrozhd/zeroth-archive` repo as the target archive and updated it in place rather than creating a duplicate."
  - "Temporarily unarchived and made the existing repo public before mirror sync, then re-archived it after the README notice commit."
patterns-established:
  - "Archive publication scripts must tolerate an already-existing remote archive and repair its state before pushing."
requirements-completed: [ARCHIVE-01, ARCHIVE-02, ARCHIVE-03]
duration: "12min"
completed: 2026-04-10
---

# Phase 27: ship-zeroth-as-pip-installable-library-zeroth-core Summary

**GitHub archive repo published with mirror refs, visible archived notice, public visibility, and remote recovery proof**

## Performance

- **Duration:** 12 min
- **Started:** 2026-04-10T17:57:00Z
- **Completed:** 2026-04-10T18:09:00Z
- **Tasks:** 2
- **Files modified:** 4

## Accomplishments

- Published the local bare mirror to `rrrozhd/zeroth-archive` and preserved the archive refs on GitHub.
- Added the required archived notice to the remote README and aligned the repo description with it.
- Verified the GitHub archive is public, archived, and remotely recoverable from `main`, stash, and detached-worktree refs.

## Task Commits

1. **Task 1: Add GitHub archive publication script** - `213b208`
2. **Task 2: Publish the GitHub archive and capture remote recovery evidence** - `307bb9f`

**Plan metadata:** recorded in the summary/state progress commit that follows this file update

## Files Created/Modified

- `scripts/publish_monolith_archive.sh` - Automates GitHub archive repo repair, mirror sync, README banner insertion, visibility alignment, and final archiving.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-github-publish.txt` - Captures the `gh` and `git` output for publication and archive steps.
- `.planning/phases/27-ship-zeroth-as-pip-installable-library-zeroth-core/artifacts/archive-recovery-test.txt` - Proves `main`, `archive/stash-0`, and an archived detached-worktree branch can be checked out from GitHub.

## Decisions Made

- Reused the existing `rrrozhd/zeroth-archive` repo instead of replacing it, because the account already had the intended archive target.
- Enforced public visibility explicitly so the remote archive matches the Phase 27 plan rather than inheriting the repo’s previous private state.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Existing archive repo was read-only and private**
- **Found during:** Task 2 (GitHub archive publication)
- **Issue:** The target repo already existed in archived/private state, so `git push --mirror` failed and the archive would not satisfy the planned public-archive shape.
- **Fix:** Updated `scripts/publish_monolith_archive.sh` to unarchive the repo temporarily, switch it to public visibility, complete the mirror push and README notice update, then archive it again.
- **Files modified:** `scripts/publish_monolith_archive.sh`
- **Verification:** `gh repo view rrrozhd/zeroth-archive --json visibility,isArchived,description` returned `PUBLIC`, `true`, and the expected archive description; remote recovery clone/checkouts succeeded.
- **Committed in:** part of the task-completion commit

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Required to complete the intended GitHub archive publication against the already-existing remote repo. No scope creep.

## Issues Encountered

- The first push attempt failed because the existing archive repo was already archived and read-only. This was resolved by temporarily unarchiving it and aligning visibility before retrying.

## Next Phase Readiness

- Plan 27-03 can now proceed with the namespace rename because all three archive layers exist and have recovery evidence.
- The remote archive contains the stash and detached-worktree refs that the rename phase must preserve historically.

## Self-Check: PASSED
