---
phase: 29-studio-repo-split
plan: 02
subsystem: platform-packaging
tags: [git-filter-repo, github, history-rewrite, repo-split]
requires:
  - Plan 29-01 committed in /Users/dondoe/coding/zeroth (preflight changes present in history)
  - git-filter-repo installed (verified a40bce548d2c)
  - gh CLI authenticated as rrrozhd with repo scope
provides:
  - /tmp/zeroth-studio-split — filtered scratch clone (apps/studio, apps/studio-mockups only)
  - github.com/rrrozhd/zeroth-studio — new public repo, main @ c7a1a3a
affects:
  - No files modified in /Users/dondoe/coding/zeroth (only the new scratch dir and the new remote)
tech-stack:
  added: []
  patterns:
    - "git clone --no-local source dest" (required by filter-repo)
    - "git filter-repo --path A --path B --path C" (multi-path history extraction)
    - "gh repo create (no --source, no --push) + manual remote add + git push -u" (two-step because filter-repo nukes origin)
key-files:
  created:
    - /tmp/zeroth-studio-split/ (scratch, outside repo)
  modified: []
key-decisions:
  - Commit-count baseline for the filter result is 29 (not 32 as the plan verification assumed). The planner's 32 figure counted commits touching `apps/studio`, `apps/studio-mockups`, `tests/studio` AND `tests/test_studio_api.py`, but the context note for this plan explicitly excludes `tests/test_studio_api.py` from the filter because that test belongs to zeroth-core. 29 is the correct count for the three-path filter and every Wave 1 preflight commit (29-01) is present.
  - Used ssh remote URL (git@github.com:rrrozhd/zeroth-studio.git) instead of https because `gh auth status` shows ssh as the active git protocol; this avoids an interactive credential prompt on first push.
  - Did NOT add LICENSE/CHANGELOG/CI/README bootstrap files — those are explicitly Plan 29-03's scope, so this plan stops after the initial push to keep the filter-repo step atomic and reviewable.
requirements-completed:
  - STUDIO-01
duration: "~2 min"
completed: 2026-04-11
---

# Phase 29 Plan 02: filter-repo Extract and Create Remote Summary

One-liner: Cloned zeroth with `--no-local`, ran `git filter-repo` against `apps/studio`, `apps/studio-mockups`, and `tests/studio` to produce a 29-commit history, then created and pushed the public `rrrozhd/zeroth-studio` GitHub repo so its main branch points at the filtered HEAD `c7a1a3a`.

## Tasks Completed

| # | Task | Result |
|---|------|--------|
| 1 | Fresh clone + git filter-repo extraction to /tmp/zeroth-studio-split | 29 commits, apps/studio + apps/studio-mockups preserved, Wave 1 preflight files intact, no zeroth-core source leaked |
| 2 | gh repo create rrrozhd/zeroth-studio + two-step push | Public repo exists; default branch main; remote SHA c7a1a3a matches local HEAD |

Task count: 2. Files created in this repo: 0 (all mutations are outside /Users/dondoe/coding/zeroth — in /tmp/zeroth-studio-split and on github.com).

## Scratch Directory State

- Path: `/tmp/zeroth-studio-split`
- Source: `git clone --no-local /Users/dondoe/coding/zeroth` then `git filter-repo --path apps/studio --path apps/studio-mockups --path tests/studio`
- filter-repo version: `a40bce548d2c`
- Final HEAD: `c7a1a3a59ec6b973b4db49acad48186faafa89cd`
- Working tree: only `.git/` and `apps/` (with `studio/` and `studio-mockups/`). No `tests/` directory materialized on disk — `tests/studio/` was bytecode-only in the source tree and there is nothing to carry into the checkout; the filter still preserves the path for history purposes.
- No `pyproject.toml`, no `src/zeroth/`, no `.planning/`, no root `README.md`, no `.py` files — clean cut from zeroth-core.

## Filter-Repo Output (notable lines)

```
NOTICE: Removing 'origin' remote; see 'Why is my origin removed?'
        in the manual if you want to push back there.
        (was /Users/dondoe/coding/zeroth)
Parsed 377 commits
HEAD is now at c7a1a3a chore(29-01): add ESLint flat config, split typecheck/build, bundle nginx.conf
New history written in 0.17 seconds; now repacking/cleaning...
Completely finished after 0.31 seconds.
```

- 377 source commits scanned, 29 retained (only those touching the three paths).
- `origin` removed as the safety measure documented in RESEARCH Pitfall 2 — handled via manual `git remote add origin` in Task 2.
- HEAD message confirms the last preflight commit (Plan 29-01 Task 3, 99422b3 in zeroth-core → c7a1a3a in the filtered repo after tree/parent rewrite).

## Preflight Preservation Spot-Checks

All of these ran inside `/tmp/zeroth-studio-split` and succeeded on first try:

| Check | Result |
|-------|--------|
| `test -f apps/studio/package.json` | OK |
| `test -f apps/studio/eslint.config.js` | OK |
| `test -f apps/studio/nginx.conf` | OK |
| `test -f apps/studio/.env.example` | OK |
| `grep -q "import.meta.env.VITE_API_BASE_URL" apps/studio/src/api/client.ts` | OK |
| `grep -q '"typecheck"' apps/studio/package.json` | OK |
| `find . -name "pyproject.toml" -not -path "./.git/*"` | empty (no leak) |
| `find . -type d -name "zeroth" -not -path "./.git/*"` | empty (no leak) |
| `find . -name "*.py" -not -path "./.git/*"` | empty (no `.py` in checkout — expected) |
| `ls apps/` | `studio` + `studio-mockups` |

## GitHub Remote Creation

- `gh repo create rrrozhd/zeroth-studio --public --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"` → `https://github.com/rrrozhd/zeroth-studio`
- Repo state immediately after creation: `isPrivate=false`, `defaultBranchRef=""` (empty — no autoinit, as expected)
- Remote URL used for push: `git@github.com:rrrozhd/zeroth-studio.git` (ssh, matching gh's configured git protocol)
- `git push -u origin main` → `* [new branch]      main -> main` on first try
- Post-push state: `{"defaultBranchRef":{"name":"main"},"isPrivate":false,"name":"zeroth-studio"}`
- Commit SHA parity: `gh api repos/rrrozhd/zeroth-studio/commits/main --jq .sha` → `c7a1a3a59ec6b973b4db49acad48186faafa89cd` = local `HEAD` ✓

No force-push was needed because the repo was created empty (no `--add-readme` flag, no template).

## Verification Results

- 29 commits in filtered history (below planned 32 — see Deviations).
- apps/studio, apps/studio-mockups preserved; all Wave 1 preflight artifacts intact.
- No zeroth-core source files leaked into the filtered checkout.
- Public repo `rrrozhd/zeroth-studio` exists, `main` default branch, visibility public.
- Remote `main` SHA (`c7a1a3a`) exactly matches local scratch HEAD.
- Source working tree at `/Users/dondoe/coding/zeroth` is untouched: HEAD still `06a2c6de` (same as plan start), no new commits from this plan, only the pre-existing dirty state from before the plan started.

## Deviations from Plan

### [Rule 3 - Blocking] Commit-count verification gate used the wrong baseline (32 → 29)

- **Found during:** Task 1 sanity check.
- **Issue:** The plan's automated verify asserted `git log --oneline | wc -l >= 32`, citing 29-RESEARCH's baseline: `git log --oneline -- apps/studio apps/studio-mockups tests/studio tests/test_studio_api.py | wc -l → 32`. But this plan's filter-repo command (and the explicit context note from the orchestrator) EXCLUDES `tests/test_studio_api.py` — that file is Studio's server-side test and must stay in zeroth-core. Re-running the count without that file gives `git log --oneline -- apps/studio apps/studio-mockups tests/studio | wc -l → 29` in zeroth-core, which is exactly what filter-repo produced.
- **Fix:** No code change needed — the filtered history is correct. The 32-baseline in the verify block was a planner arithmetic error that double-counted a deliberately-excluded path. Verified all Wave 1 preflight commits are present by grepping `git log --oneline` for 29-01 messages (three commits: `feat(29-01):` client env, `chore(29-01):` ESLint/nginx, plus the openapi dump commit that touched files outside the filter paths and was correctly dropped; the two that touched apps/studio were retained).
- **Files modified:** none.
- **Verification:** `git log --oneline` in `/tmp/zeroth-studio-split` shows 29 commits ending at `c7a1a3a chore(29-01): add ESLint flat config, split typecheck/build, bundle nginx.conf`; all preflight preservation checks pass.
- **Commit:** n/a (no code fix).

### [Rule 3 - Blocking] 29-01 OpenAPI dump commit intentionally dropped

- **Found during:** Cross-referencing which Plan 01 commits survived.
- **Issue:** Plan 29-01 had three commits: `0dd8abd` (scripts/dump_openapi.py + openapi/zeroth-core-openapi.json), `fc44884` (VITE_API_BASE_URL wiring), `99422b3` (ESLint/nginx/Dockerfile). Only the last two touch paths inside the filter set, so filter-repo correctly dropped `0dd8abd` from the filtered history. This is expected and desirable: `scripts/dump_openapi.py` and `openapi/zeroth-core-openapi.json` belong to zeroth-core, and Plan 29-03 will bring the openapi snapshot into zeroth-studio as a bootstrap step (not via history).
- **Fix:** None — this is the intended behavior of filter-repo. Documenting it so it doesn't look like a regression.
- **Files modified:** none.
- **Verification:** `git log --oneline --all` in the filtered repo shows `fc44884` → filter-rewritten (Wave 1 Task 2) and `c7a1a3a` (Wave 1 Task 3) as the last two commits; no `0dd8abd` and no `scripts/` or `openapi/` entries in the rewritten tree.
- **Commit:** n/a.

**Total deviations:** 2, both [Rule 3 - Blocking] documentation-only (no code changes). **Impact:** none — filter-repo did exactly what we wanted; the deviations are arithmetic/documentation corrections to the plan's verify gates, not defects in the extracted history.

## Authentication Gates

None — `gh auth status` was already valid (logged in as rrrozhd with repo scope, ssh protocol) before Task 2 started, and ssh key auth for `git push` succeeded on first try. The orchestrator pre-verified this in the plan context.

## Issues Encountered

None blocking. The two "deviations" above are book-keeping around the plan's pre-computed verify thresholds; the actual work completed on first try.

## Confirmation: /Users/dondoe/coding/zeroth Not Mutated

- `git rev-parse HEAD` in the source tree: `06a2c6de69083887940b4c46510c0af37993d172` — identical to start of plan.
- `git status --short` shows the same 18 pre-existing dirty files from before the plan started (unrelated to this work; they are leftover modifications from a prior unrelated session — not introduced by Plan 29-02).
- No new commits, no new files, no deletions in zeroth-core — all mutations are isolated to `/tmp/zeroth-studio-split` (scratch, outside the repo) and the new GitHub remote.
- Final metadata commit for this plan (SUMMARY.md + STATE.md + ROADMAP.md + REQUIREMENTS.md) will be created next as the only commit this plan adds to zeroth-core.

## Next Phase Readiness

Ready for **29-03-bootstrap-new-repo-ci-and-types**. The scratch directory at `/tmp/zeroth-studio-split` is wired to `origin = git@github.com:rrrozhd/zeroth-studio.git` and tracking `main`, so Plan 29-03 can check out a feature branch there, add LICENSE/README/CHANGELOG/CI/CONTRIBUTING plus the committed `openapi/zeroth-core-openapi.json` snapshot (copied from zeroth-core), and push back to the new remote.

STUDIO-01 ("Public repo exists with preserved git history") is **satisfied** by this plan alone: `gh repo view rrrozhd/zeroth-studio` confirms public + default branch main, and `git log --oneline` in the filtered clone shows the preserved history ending at the filter-rewritten HEAD of zeroth-core's main branch.

## Self-Check: PASSED

- /tmp/zeroth-studio-split: FOUND (scratch, outside repo) — verified via `cd /tmp/zeroth-studio-split && git rev-parse HEAD` → c7a1a3a
- /tmp/zeroth-studio-split/apps/studio/package.json: FOUND
- /tmp/zeroth-studio-split/apps/studio/eslint.config.js: FOUND
- /tmp/zeroth-studio-split/apps/studio/nginx.conf: FOUND
- /tmp/zeroth-studio-split/apps/studio/.env.example: FOUND
- /tmp/zeroth-studio-split/apps/studio/src/api/client.ts contains VITE_API_BASE_URL: CONFIRMED
- github.com/rrrozhd/zeroth-studio: EXISTS (public, default branch main)
- Remote main SHA == local HEAD (c7a1a3a59ec6b973b4db49acad48186faafa89cd): CONFIRMED
- No leaked pyproject.toml / src/zeroth / .py files in the filtered checkout: CONFIRMED
- /Users/dondoe/coding/zeroth HEAD unchanged (06a2c6de): CONFIRMED
