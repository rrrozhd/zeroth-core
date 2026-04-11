---
phase: 29-studio-repo-split
plan: 04
subsystem: platform-packaging
tags: [cleanup, repo-split, readme, deletion]
requires:
  - Plan 29-03 complete (rrrozhd/zeroth-studio bootstrapped, CI green on run 24281557404)
  - Safety gate passing at execution time (gh verified remote + latest main run = success)
provides:
  - Single source of truth: studio source lives only in rrrozhd/zeroth-studio
  - zeroth-core README has a Studio cross-link section (closes STUDIO-04)
  - pyproject.toml / docker-compose.yml / .claude/launch.json free of apps/studio refs
  - tests/test_studio_api.py preserved and still green (10/10)
affects:
  - apps/ (deleted entirely)
  - tests/studio/ (deleted — bytecode stub only)
  - docker-compose.yml (nginx service removed; was building ./apps/studio)
  - .claude/launch.json (studio-mockups debug config removed)
  - pyproject.toml (apps/** removed from ruff per-file-ignores + interrogate exclude)
  - README.md (Studio section added)
tech-stack:
  added: []
  patterns:
    - "Safety-gated destructive cleanup: verify remote+CI green via gh CLI before any local deletion"
    - "Historical docs under .planning/ and docs/superpowers/ kept verbatim — they document past state, not active config"
key-files:
  created:
    - .planning/phases/29-studio-repo-split/29-04-cleanup-zeroth-core-SUMMARY.md
  modified:
    - README.md
    - docker-compose.yml
    - .claude/launch.json
    - pyproject.toml
  deleted:
    - apps/studio/ (entire tree, 52 tracked files + ignored dist/node_modules/tsbuildinfo)
    - apps/studio-mockups/ (entire tree, 9 tracked files + ignored dist/node_modules)
    - tests/studio/ (bytecode-only __pycache__ dir)
    - apps/ (empty parent dir removed after contents gone)
key-decisions:
  - "Removed the docker-compose nginx service instead of leaving it pointing at a deleted build context. Keeping it would silently break `docker compose build`; rewriting it to pull a prebuilt studio image is out of scope for this cleanup plan and belongs to a follow-up once zeroth-studio publishes container artifacts."
  - "Dropped the .claude/launch.json studio-mockups debug entry because its runtimeArgs pointed at apps/studio-mockups — a dead path after deletion. The zeroth-api entry is preserved."
  - "Stripped apps/** from pyproject.toml ruff per-file-ignores and interrogate exclude. These entries had no effect once apps/ was gone, but leaving dead config is a code smell and would confuse future readers searching for 'apps' references."
  - "Left historical mentions in PROGRESS.md and docs/superpowers/ untouched. Those files document past phase state (Phase 10 mockup build, repo split design spec) — rewriting history-of-record docs is out of scope for a cleanup plan."
  - "Preserved docker/nginx/studio.conf on disk — orphaned after nginx service removal, but removing it is a small follow-up and not a blocker. Flagged in deferred items below rather than silently sweeping it."
requirements-completed:
  - STUDIO-03
  - STUDIO-04
duration: "~8 min"
completed: 2026-04-11
---

# Phase 29 Plan 04: Cleanup zeroth-core Summary

One-liner: Deleted apps/studio/, apps/studio-mockups/, and tests/studio/ from zeroth-core after confirming rrrozhd/zeroth-studio CI run 24281557404 was green on main; scrubbed dead refs from docker-compose.yml, .claude/launch.json, and pyproth.toml; added a Studio cross-link section to README.md pointing at the new repo and its compatibility matrix.

## Tasks Completed

| # | Task | Commit | Result |
|---|------|--------|--------|
| 1 | Safety gate + delete apps/ + tests/studio/, fix stale config refs, run pytest + ruff | 9d52d4b | 62 files changed, 4 insertions(+), 11658 deletions(-). Full pytest 662 passed / 12 deselected / 0 failed. `ruff check src/` clean. `tests/test_studio_api.py` 10/10 pass. |
| 2 | Add Studio cross-link section to README.md | b137753 | 1 file changed, 10 insertions(+). Link `https://github.com/rrrozhd/zeroth-studio` present at README:211. |

## Safety Gate

- **Command:** `gh repo view rrrozhd/zeroth-studio --json name,isPrivate,defaultBranchRef` → `{"name":"zeroth-studio","isPrivate":false,"defaultBranchRef":{"name":"main"}}`
- **Command:** `gh run list --repo rrrozhd/zeroth-studio --branch main --limit 1 --json conclusion,headSha,databaseId` → `[{"conclusion":"success","databaseId":24281557404,"headSha":"b981943fe74b128827fc0cb1b469db6fd07fe639"}]`
- **Gate status:** PASSED. Proceeded with deletion.

## Files Deleted (counts)

- `apps/studio/` tracked files: **52** (src/, package.json, vite.config.ts, tsconfig*, eslint.config.js, index.html, Dockerfile, nginx.conf, .env.example, .gitignore, .dockerignore). Plus untracked build artifacts removed via `rm -rf`: `dist/`, `node_modules/`, `tsconfig.tsbuildinfo`.
- `apps/studio-mockups/` tracked files: **9** (index.html, package.json, package-lock.json, editor-mockup.png, src/App.vue, src/components/StudioEditorMockup.vue, src/main.ts, src/style.css, vite.config.ts, tsconfig.json — staged via `git add -u` after `rm -rf`). Plus untracked `dist/`, `node_modules/`.
- `tests/studio/`: no tracked files (bytecode-only). Removed via `rm -rf` (3 stale `.pyc` files in `__pycache__/`).
- `apps/`: empty after above, removed via `rmdir`.

**Total tracked files deleted:** 61. **Commit line counts:** `62 files changed, 4 insertions(+), 11658 deletions(-)` (62 = 61 deletions + 1 addition for the docker-compose comment block; the line count also covers the docker-compose.yml, launch.json, pyproject.toml edits).

## Files Preserved (plan-critical)

- `tests/test_studio_api.py` — ✓ present at tests/ root, unmodified, 10/10 tests passing. This is the canonical server-side HTTP contract for the studio_api router.
- `src/zeroth/core/service/studio_api.py` — ✓ unchanged (not in plan scope; server-side router stays).
- `scripts/dump_openapi.py` — ✓ unchanged.
- `openapi/zeroth-core-openapi.json` — ✓ unchanged.

## Config File Diffs

### docker-compose.yml

Removed the entire `nginx:` service block that was building from `./apps/studio` and mounting `./docker/nginx/studio.conf`. Replaced with a comment block explaining the split:

```diff
-  nginx:
-    build:
-      context: ./apps/studio
-      dockerfile: Dockerfile
-    ports:
-      - "443:443"
-      - "80:80"
-    volumes:
-      - ./docker/nginx/studio.conf:/etc/nginx/conf.d/default.conf:ro
-      - ./docker/nginx/certs:/etc/nginx/certs:ro
-    depends_on:
-      - zeroth
-    networks:
-      - zeroth-net
-    restart: unless-stopped
+  # NOTE: Studio frontend moved to https://github.com/rrrozhd/zeroth-studio
+  # in Phase 29. The nginx reverse proxy that served the built studio bundle
+  # is no longer defined here; run the studio separately against this stack.
```

### .claude/launch.json

Removed the `studio-mockups` debug launch configuration; kept `zeroth-api`:

```diff
-    {
-      "name": "studio-mockups",
-      "runtimeExecutable": "npm",
-      "runtimeArgs": ["--prefix", "apps/studio-mockups", "run", "dev"],
-      "port": 5173
-    },
     {
       "name": "zeroth-api",
```

### pyproject.toml

Two targeted deletions of now-dead config entries:

```diff
 [tool.ruff.lint.per-file-ignores]
 "tests/**/*.py" = ["D", "B006", "B017", "E501", "F401", "F841", "I001", "SIM117"]
 "scripts/*.py" = ["D"]
-"apps/**/*.py" = ["D"]
 "src/zeroth/core/migrations/**/*.py" = ["D"]
```

```diff
 [tool.interrogate]
 ...
-exclude = ["src/zeroth/core/migrations", "tests", "scripts", "apps"]
+exclude = ["src/zeroth/core/migrations", "tests", "scripts"]
```

No `testpaths` change needed — it was already `["tests"]`.

## README.md Insertion

Inserted a new `## Studio` section between the existing Project Structure / Executable Unit Modes / Design Principles block and the final `## License` section (README:210 area). New content:

```markdown
## Studio

Zeroth's canvas UI for authoring and inspecting workflows lives in a separate repo:

**[rrrozhd/zeroth-studio](https://github.com/rrrozhd/zeroth-studio)** — Vue 3 + Vue Flow frontend that speaks to `zeroth-core` over HTTP.

The studio was split out in v3.0 Phase 29 to let the two projects ship on independent release cadences. A cross-repo [compatibility matrix](https://github.com/rrrozhd/zeroth-studio#compatibility) documents which studio versions pair with which core versions.
```

Verification: `grep -n "github.com/rrrozhd/zeroth-studio" README.md` → matches at lines 211, 213.

## Stray-Reference Scan

Initial scan across `*.md *.toml *.yml *.yaml *.sh *.py` (excluding `.planning/` and `.git/`) found:

| File | Line | Ref | Disposition |
|---|---|---|---|
| `docker-compose.yml` | 66 | `context: ./apps/studio` | **Fixed** — nginx service removed (Task 1) |
| `.claude/launch.json` | 7 | `apps/studio-mockups` | **Fixed** — studio-mockups debug config removed (Task 1) |
| `docs/superpowers/specs/2026-04-10-zeroth-core-platform-split-design.md` | 94, 145, 262, 435, 679 | `apps/studio` | **Left as-is** — historical design spec documenting the split itself; rewriting it would erase record of the decision |
| `docs/superpowers/plans/2026-04-10-zeroth-core-platform-split-plan.md` | 34, 59, 112 | `apps/studio` | **Left as-is** — historical plan doc |
| `PROGRESS.md` | 590, 599, 600 | `apps/studio-mockups` | **Left as-is** — phase log describing Phase 10 work; PROGRESS.md is append-only history |

Post-fix re-scan against non-doc, non-historical files: **0 references**. The only remaining mentions sit inside documentation that describes past state, which is correct.

## Test Suite Results

**Baseline (post-Phase 27, pre-this-plan):** 662 passed / 12 deselected / 0 failed (confirmed from recent commit `a35a0ae fix: resolve 26 pre-existing test failures from baseline`).

**After deletion:** 662 passed / 12 deselected / 0 failed in 44.94s. **Delta: 0 new failures.**

Focused run on the preservation target:

```
tests/test_studio_api.py::TestCreateWorkflow::test_create_workflow PASSED                 [ 10%]
tests/test_studio_api.py::TestCreateWorkflow::test_create_workflow_empty_name_rejected PASSED [ 20%]
tests/test_studio_api.py::TestListWorkflows::test_list_workflows PASSED                   [ 30%]
tests/test_studio_api.py::TestGetWorkflow::test_get_workflow PASSED                       [ 40%]
tests/test_studio_api.py::TestGetWorkflow::test_get_workflow_not_found PASSED             [ 50%]
tests/test_studio_api.py::TestUpdateWorkflow::test_update_workflow_name PASSED            [ 60%]
tests/test_studio_api.py::TestUpdateWorkflow::test_update_workflow_not_found PASSED       [ 70%]
tests/test_studio_api.py::TestDeleteWorkflow::test_delete_workflow PASSED                 [ 80%]
tests/test_studio_api.py::TestDeleteWorkflow::test_delete_workflow_not_found PASSED       [ 90%]
tests/test_studio_api.py::TestListNodeTypes::test_list_node_types PASSED                  [100%]
============================== 10 passed in 0.75s ==============================
```

`uv run ruff check src/` → `All checks passed!`

## Verification vs. Plan must_haves

- [x] `apps/studio/`, `apps/studio-mockups/`, and `tests/studio/` no longer exist in zeroth-core
- [x] `tests/test_studio_api.py` is untouched and still passes (10/10)
- [x] `README.md` has a Studio section linking to https://github.com/rrrozhd/zeroth-studio
- [x] `uv run pytest` passes with 0 new failures (662 passed, matches baseline)
- [x] `uv run ruff check src/` passes
- [x] `pyproject.toml` / pytest config does not reference the deleted paths
- [x] Safety gate confirmed green before any deletion

## Deviations from Plan

### [Rule 3 — Blocking] docker-compose nginx service referenced deleted build context

- **Found during:** Task 1, broader repo scan for stray references.
- **Issue:** `docker-compose.yml` defined an `nginx` service with `build: { context: ./apps/studio, dockerfile: Dockerfile }`. Deleting `apps/studio/` would make `docker compose build` fail with "unable to prepare context: path ./apps/studio not found".
- **Fix:** Removed the entire `nginx:` service block. Replaced with a 3-line comment pointing to the new repo and explaining why the reverse-proxy service is no longer defined here. The `zeroth`, `postgres`, `redis`, `regulus`, and `sandbox-sidecar` services are untouched.
- **Files modified:** `docker-compose.yml`
- **Verification:** `grep "apps/studio" docker-compose.yml` → no matches; the only `nginx` reference is the replacement comment.
- **Commit:** 9d52d4b (bundled with Task 1).

### [Rule 3 — Blocking] .claude/launch.json studio-mockups debug config pointed at deleted path

- **Found during:** Task 1, broader repo scan.
- **Issue:** Debug configuration `studio-mockups` had `runtimeArgs: ["--prefix", "apps/studio-mockups", "run", "dev"]`. After deletion, invoking that launch config from an IDE would fail with ENOENT.
- **Fix:** Removed the `studio-mockups` configuration entry from the `configurations` array. Left the `zeroth-api` entry untouched.
- **Files modified:** `.claude/launch.json`
- **Verification:** JSON is still valid; `grep apps launch.json` returns nothing.
- **Commit:** 9d52d4b.

### [Rule 3 — Blocking] pyproject.toml had dead apps/** entries in ruff and interrogate configs

- **Found during:** Task 1, targeted pyproject.toml scan per plan step 4.
- **Issue:** `[tool.ruff.lint.per-file-ignores]` contained `"apps/**/*.py" = ["D"]` and `[tool.interrogate] exclude` contained `"apps"`. After apps/ deletion these entries are dead config — not runtime-breaking, but the plan step 4 explicitly calls for scrubbing stale ruff/interrogate includes tied to studio paths.
- **Fix:** Removed both entries with targeted edits (no broader reformatting).
- **Files modified:** `pyproject.toml`
- **Verification:** `uv run ruff check src/` → clean. `grep -n "apps" pyproject.toml` → no matches.
- **Commit:** 9d52d4b.

**Total deviations:** 3, all [Rule 3 — Blocking] auto-fixed in the same commit as Task 1. **Impact:** all three were unavoidable — deletion without fixing them would have either broken `docker compose build` (docker-compose.yml), broken IDE debug launches (launch.json), or left dead config behind (pyproject.toml). No architectural changes, no Rule 4. The plan anticipated all three in steps 4-5 ("pyproject.toml scan" and "broader repo scan").

## Authentication Gates

None. `gh` CLI already authenticated as `rrrozhd` (inherited from Plan 29-02 / 29-03 session). Safety gate ran on the first try.

## Deferred Items

- **docker/nginx/studio.conf and docker/nginx/certs/**: orphaned after nginx service removal from docker-compose.yml. Kept on disk because removing them is not a blocker for this plan and belongs in a general docker-infra cleanup pass. No active config references them now.
- **Historical doc mentions** (PROGRESS.md, docs/superpowers/): left verbatim because those files document past decisions, not current active config.

## Issues Encountered

None. All plan steps executed as written, safety gate passed on first check, and the three Rule 3 blocking fixes were exactly the stale references the plan expected the broader scan to surface.

## Success Criteria Status

1. Single-source-of-truth achieved: ✓ studio lives only at rrrozhd/zeroth-studio; zeroth-core has zero frontend source files
2. zeroth-core test suite green: ✓ 662 passed + ruff clean
3. tests/test_studio_api.py preserved: ✓ untouched, 10/10 passing
4. README cross-link: ✓ STUDIO-04 loop closed (matrix in zeroth-studio README, pointer in zeroth-core README)
5. Safety gate prevented accidental deletion: ✓ ran first, confirmed run 24281557404 success before touching anything

## Phase 29 Overall Status

This is the final plan of Phase 29 (wave 4 of 4). With this plan complete:

- STUDIO-01 (rrrozhd/zeroth-studio with preserved history): done in Plan 29-02
- STUDIO-02 (zeroth-studio CI green on main): done in Plan 29-03
- STUDIO-03 (no Python imports under apps/ in zeroth-studio, zero apps/ in zeroth-core): **closed here**
- STUDIO-04 (compat matrix + cross-link on both sides): **closed here**
- STUDIO-05 (types.gen.ts + drift gate on zeroth-studio): done in Plan 29-03

Phase 29 ready for a phase-level rollup SUMMARY and verification pass.

## Self-Check: PASSED

- `apps/` directory: REMOVED (verified with `test ! -d apps`)
- `apps/studio/` directory: REMOVED
- `apps/studio-mockups/` directory: REMOVED
- `tests/studio/` directory: REMOVED
- `tests/test_studio_api.py`: PRESERVED and passing 10/10
- README.md contains `github.com/rrrozhd/zeroth-studio`: FOUND at lines 211, 213
- `docker-compose.yml` contains no `apps/studio` references: CONFIRMED
- `.claude/launch.json` contains no `apps/studio-mockups` references: CONFIRMED
- `pyproject.toml` contains no `apps` references: CONFIRMED
- `uv run pytest` full suite: 662 passed / 12 deselected / 0 failed (baseline match)
- `uv run ruff check src/`: All checks passed
- Commit 9d52d4b (Task 1): FOUND in `git log --oneline`
- Commit b137753 (Task 2): FOUND in `git log --oneline`
