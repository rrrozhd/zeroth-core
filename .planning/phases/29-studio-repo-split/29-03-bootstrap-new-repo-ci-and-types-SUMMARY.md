---
phase: 29-studio-repo-split
plan: 03
subsystem: platform-packaging
tags: [bootstrap, github-actions, openapi-typescript, drift-gate, license, changelog]
requires:
  - Plan 29-02 complete (scratch clone + remote at c7a1a3a)
  - Node 22 + npm available locally
  - gh CLI authenticated as rrrozhd (ssh)
provides:
  - zeroth-studio public repo bootstrapped on main at b981943
  - Green GitHub Actions CI pipeline (lint/typecheck/build/test/drift-check)
  - Committed OpenAPI snapshot at openapi/zeroth-core-openapi.json
  - Generated TypeScript types at apps/studio/src/api/types.gen.ts (2798 lines, 86 KB)
  - Apache-2.0 LICENSE, keepachangelog CHANGELOG, CONTRIBUTING.md, README with compat matrix
affects:
  - No files in /Users/dondoe/coding/zeroth except this SUMMARY + state metadata
tech-stack:
  added: []
  patterns:
    - "CI drift gate: npm run generate:api + git diff --exit-code src/api/types.gen.ts"
    - "Single-job GH Actions matrix on Node 22 with working-directory: apps/studio"
    - "openapi-typescript relative path ../../openapi/... from apps/studio cwd"
key-files:
  created:
    - /tmp/zeroth-studio-split/LICENSE
    - /tmp/zeroth-studio-split/CHANGELOG.md
    - /tmp/zeroth-studio-split/CONTRIBUTING.md
    - /tmp/zeroth-studio-split/README.md
    - /tmp/zeroth-studio-split/.gitignore
    - /tmp/zeroth-studio-split/.github/workflows/ci.yml
    - /tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json
    - /tmp/zeroth-studio-split/apps/studio/src/api/types.gen.ts
  modified:
    - /tmp/zeroth-studio-split/apps/studio/package.json (fixed generate:api path, added --passWithNoTests)
key-decisions:
  - "CI pipeline is a single job on Node 22 (matches apps/studio/Dockerfile FROM node:22-alpine). No multi-version matrix — the runtime target is one."
  - "Drift gate is the LAST CI step so earlier failures (lint/typecheck/build/test) surface first; drift only fires if everything else is clean."
  - "Added --passWithNoTests to vitest run — apps/studio currently has no test files (tests/studio from zeroth-core was bytecode-only and did not materialize, see 29-02 SUMMARY), and vitest's default exit=1 on empty suite would make CI red. Rule 3 blocking fix; re-enabled once real tests land."
  - "generate:api relative path switched from 'openapi/zeroth-core-openapi.json' to '../../openapi/zeroth-core-openapi.json'. The script runs with cwd=apps/studio, so the openapi/ dir at the repo root is two levels up. The planner anticipated this fix."
  - "apps/studio-mockups is intentionally NOT wired into CI — design reference only per 29-CONTEXT D-01 clarification."
  - "LICENSE copied verbatim from zeroth-core to keep license text byte-identical."
requirements-completed:
  - STUDIO-02
  - STUDIO-04
  - STUDIO-05
duration: "~5 min"
completed: 2026-04-11
---

# Phase 29 Plan 03: Bootstrap new repo CI and types Summary

One-liner: Bootstrapped zeroth-studio with Apache-2.0 LICENSE, keepachangelog CHANGELOG, CONTRIBUTING.md, README with compat matrix (0.1.0 ↔ 0.1.1) + cross-link to zeroth-core, GitHub Actions CI (lint → typecheck → build → test → generate:api → drift-check), committed OpenAPI snapshot + generated types.gen.ts, pushed as a single bootstrap commit to main, and verified CI green on first push.

## Tasks Completed

| # | Task | Result |
|---|------|--------|
| 1 | Copy OpenAPI snapshot, generate types, add LICENSE/CHANGELOG/CONTRIBUTING/README/.gitignore | 6 root files + openapi/ + types.gen.ts (2798 lines). Local lint/typecheck/build/test all green. |
| 2 | Add .github/workflows/ci.yml, commit bootstrap, push, verify CI green | Commit b981943 pushed to main. Run 24281557404 conclusion=success on first push. |

Task count: 2. New files created in zeroth-studio: 8 (LICENSE, CHANGELOG.md, CONTRIBUTING.md, README.md, .gitignore, .github/workflows/ci.yml, openapi/zeroth-core-openapi.json, apps/studio/src/api/types.gen.ts). Modified: 1 (apps/studio/package.json). New files in zeroth-core: 1 (this SUMMARY).

## CI Run Details

- **Repo:** rrrozhd/zeroth-studio
- **Run URL:** https://github.com/rrrozhd/zeroth-studio/actions/runs/24281557404
- **Run ID:** 24281557404
- **headSha:** b981943fe74b128827fc0cb1b469db6fd07fe639
- **Status:** completed
- **Conclusion:** success
- **Duration:** ~20 seconds
- **Jobs/Steps (all ✓):** Set up job → checkout@v4 → Setup Node 22 → npm ci → lint → typecheck → build → test → generate:api → git diff --exit-code src/api/types.gen.ts → Complete job
- **Annotations:** Non-blocking deprecation warning — `actions/checkout@v4` and `actions/setup-node@v4` run on Node.js 20 internally; GitHub is moving runners to Node 24 on 2026-06-02. No action needed for green CI; can be addressed in a future maintenance plan by bumping action SHAs or setting `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true`. Tracked but not fixed in this plan (out of scope).

## types.gen.ts Shape

- **Path:** `apps/studio/src/api/types.gen.ts`
- **Size:** 86,185 bytes (2798 lines)
- **Top-level exports:**
  - `export interface paths` — REST routes (list/create/get/update for runs, approvals, studio, admin, etc.)
  - `export type webhooks = Record<string, never>` — empty
  - `export interface components` — schemas (request/response bodies)
  - `export type $defs = Record<string, never>` — empty
  - `export interface operations` — named operation IDs (e.g., `list_node_types_api_studio_v1_node_types_get`)
- **Source spec:** `openapi/zeroth-core-openapi.json` (109,261 bytes, the Plan 29-01 snapshot copied verbatim)
- **Generator:** openapi-typescript 7.13.0 via `../../openapi/zeroth-core-openapi.json` relative path (cwd=apps/studio)
- **Drift gate verified:** CI runs `npm run generate:api` then `git diff --exit-code` on the generated file — passed in run 24281557404.

## Bootstrap Commit

- **SHA:** b981943fe74b128827fc0cb1b469db6fd07fe639 (short: b981943)
- **Parent:** c7a1a3a (last commit from Plan 29-02 = filter-rewritten Plan 29-01 preflight)
- **Author:** rrrozhd <rrrozhd@users.noreply.github.com>
- **Message:** `chore(bootstrap): add LICENSE, CI, README with compat matrix, OpenAPI snapshot, generated types` (+ body explaining STUDIO-02/04/05 delivery and the two auto-fixes)
- **Files changed:** 9 files, +7296 / -2 lines

## Verification Results

- [x] LICENSE at repo root (Apache-2.0, byte-identical to zeroth-core LICENSE)
- [x] CHANGELOG.md keepachangelog format with `## [0.1.0] — 2026-04-11` initial entry
- [x] CONTRIBUTING.md at repo root
- [x] README.md at repo root with `## Compatibility` section, compat matrix row `0.1.0 | 0.1.1`, and link to `https://github.com/rrrozhd/zeroth`
- [x] .gitignore at repo root
- [x] .github/workflows/ci.yml committed on main
- [x] openapi/zeroth-core-openapi.json committed at repo root
- [x] apps/studio/src/api/types.gen.ts generated, committed, 2798 lines
- [x] apps/studio/package.json generate:api path = `../../openapi/zeroth-core-openapi.json` (verified via grep)
- [x] CI run 24281557404 on main conclusion=success
- [x] Drift gate (`git diff --exit-code src/api/types.gen.ts`) passed inside CI
- [x] Phase-level STUDIO-03 cross-check: `grep -rE "^from zeroth|^import zeroth" apps/` returns zero matches — no Python imports leaked
- [x] /Users/dondoe/coding/zeroth HEAD unchanged during plan execution (no code commits in the main repo; only this SUMMARY + metadata commit follows)

## Deviations from Plan

### [Rule 3 — Blocking] vitest fails with exit=1 on empty test suite

- **Found during:** Task 1, after running `npm run lint && npm run typecheck && npm run build && npm run test` locally.
- **Issue:** `apps/studio` has no `*.test.ts` / `*.spec.ts` files yet. vitest 3.2.4 defaults to exit code 1 when no test files match, which would turn CI red at the `npm run test` step and block the bootstrap from going green on first push.
- **Fix:** Added `--passWithNoTests` to the `test` script in `apps/studio/package.json`:
  ```diff
  -    "test": "vitest run",
  +    "test": "vitest run --passWithNoTests",
  ```
  Kept `test:watch` unchanged since it's a developer convenience, not a CI gate. Once real tests are added in future phases, the flag becomes a no-op (vitest only uses it when zero matches are found).
- **Files modified:** `/tmp/zeroth-studio-split/apps/studio/package.json`
- **Verification:** `npm run test` locally now prints `No test files found, exiting with code 0`; CI step `Test` ✓ in run 24281557404.
- **Commit:** included in bootstrap commit b981943.

### [Rule 3 — Blocking] generate:api relative path was wrong for the repo layout

- **Found during:** Task 1, pre-flight path check from the plan.
- **Issue:** Plan 29-01 added the script as `openapi-typescript openapi/zeroth-core-openapi.json -o src/api/types.gen.ts`, which assumes cwd=repo-root. But npm runs scripts with cwd=package root, and the package lives at `apps/studio/`. So the bare path would try to read `apps/studio/openapi/...` — which does not exist. The snapshot lives at the repo root (two levels up).
- **Fix:** Updated the script to `openapi-typescript ../../openapi/zeroth-core-openapi.json -o src/api/types.gen.ts`. This was explicitly anticipated by the plan ("Fix the script to use ../../openapi/..."); including here for deviation-tracking completeness.
- **Files modified:** `/tmp/zeroth-studio-split/apps/studio/package.json`
- **Verification:** `npm run generate:api` succeeded and produced 2798 lines in `apps/studio/src/api/types.gen.ts`; CI drift gate passed on first run.
- **Commit:** included in bootstrap commit b981943.

**Total deviations:** 2, both [Rule 3 — Blocking] auto-fixed in the same bootstrap commit (no intermediate commits). **Impact:** none on plan scope or success criteria — both are local mechanical corrections. No architectural changes, no Rule 4.

## Authentication Gates

None. `gh auth status` showed valid rrrozhd login with ssh protocol (inherited from Plan 29-02). Both `git push origin main` and `gh run watch` / `gh run view` executed on first try without prompting.

## Issues Encountered

None blocking. One non-blocking annotation from GitHub:
- `actions/checkout@v4` and `actions/setup-node@v4` use Node.js 20 internally, which GitHub is deprecating by 2026-06-02. Future-dated, not failing the build. Suggest a maintenance bump to newer action SHAs in a future plan — out of scope for this bootstrap.

## Success Criteria Status

1. STUDIO-02 (independent CI green on main): ✓ run 24281557404 conclusion=success
2. STUDIO-04 (README cross-link + compat matrix): ✓ README.md has `Compatibility` section with `0.1.0 ↔ 0.1.1` row and link to `https://github.com/rrrozhd/zeroth`
3. STUDIO-05 (types generated from committed snapshot + drift gate): ✓ types.gen.ts committed, CI drift gate passed
4. STUDIO-03 (no Python imports under apps/): ✓ grep returns zero matches
5. Offline-reproducible (no runtime HTTP fetch for type gen): ✓ spec is a committed file, openapi-typescript reads from disk

## Next Phase Readiness

Ready for **29-04-remove-studio-from-zeroth-core-and-cross-link** (final wave of Phase 29). The new repo has an independent green CI and committed types, so zeroth-core can now safely delete `apps/studio/`, `apps/studio-mockups/`, and `tests/studio/` from its tree and add a "Studio" link in its own README pointing at `https://github.com/rrrozhd/zeroth-studio`.

## Self-Check: PASSED

- /tmp/zeroth-studio-split/LICENSE: FOUND
- /tmp/zeroth-studio-split/CHANGELOG.md: FOUND
- /tmp/zeroth-studio-split/CONTRIBUTING.md: FOUND
- /tmp/zeroth-studio-split/README.md: FOUND (contains "Compatibility" + "github.com/rrrozhd/zeroth")
- /tmp/zeroth-studio-split/.gitignore: FOUND
- /tmp/zeroth-studio-split/.github/workflows/ci.yml: FOUND (contains `generate:api` + `git diff --exit-code`)
- /tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json: FOUND (109,261 bytes)
- /tmp/zeroth-studio-split/apps/studio/src/api/types.gen.ts: FOUND (2798 lines)
- /tmp/zeroth-studio-split/apps/studio/package.json generate:api == ../../openapi/zeroth-core-openapi.json: CONFIRMED
- GitHub run 24281557404 conclusion=success on headSha b981943: CONFIRMED
- Commit b981943 present on github.com/rrrozhd/zeroth-studio main: CONFIRMED (push output `c7a1a3a..b981943  main -> main`)
- /Users/dondoe/coding/zeroth working tree untouched except for this new SUMMARY: CONFIRMED
