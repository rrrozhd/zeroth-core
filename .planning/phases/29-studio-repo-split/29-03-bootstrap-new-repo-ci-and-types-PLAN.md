---
phase: 29-studio-repo-split
plan: 03
type: execute
wave: 3
depends_on: [29-02]
files_modified:
  - /tmp/zeroth-studio-split/LICENSE
  - /tmp/zeroth-studio-split/CHANGELOG.md
  - /tmp/zeroth-studio-split/CONTRIBUTING.md
  - /tmp/zeroth-studio-split/README.md
  - /tmp/zeroth-studio-split/.gitignore
  - /tmp/zeroth-studio-split/.github/workflows/ci.yml
  - /tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json
  - /tmp/zeroth-studio-split/apps/studio/src/api/types.gen.ts
  - github:rrrozhd/zeroth-studio main branch
autonomous: true
requirements:
  - STUDIO-02
  - STUDIO-04
  - STUDIO-05
user_setup: []

must_haves:
  truths:
    - "zeroth-studio has LICENSE (Apache-2.0), CHANGELOG.md (keepachangelog), CONTRIBUTING.md at repo root"
    - "zeroth-studio README cross-links to zeroth-core and documents the compat matrix (0.1.0 ↔ 0.1.1)"
    - "openapi/zeroth-core-openapi.json is committed at the zeroth-studio repo root (copied from the Plan 01 snapshot in zeroth-core)"
    - "apps/studio/package.json generate:api resolves correctly against the repo-root openapi/ path"
    - "apps/studio/src/api/types.gen.ts is generated, committed, and consistent with the spec"
    - ".github/workflows/ci.yml runs install → lint → typecheck → build → test → generate:api → drift-check on push/PR to main"
    - "GitHub Actions CI is green on main after the bootstrap push"
  artifacts:
    - path: "/tmp/zeroth-studio-split/LICENSE"
      provides: "Apache-2.0 license file"
      contains: "Apache License"
    - path: "/tmp/zeroth-studio-split/README.md"
      provides: "Project README with compat matrix + cross-link + dev quickstart"
      contains: "zeroth-studio"
    - path: "/tmp/zeroth-studio-split/CHANGELOG.md"
      provides: "keepachangelog-format changelog, initial 0.1.0 entry"
      contains: "## [0.1.0]"
    - path: "/tmp/zeroth-studio-split/.github/workflows/ci.yml"
      provides: "GitHub Actions CI pipeline"
      contains: "generate:api"
    - path: "/tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json"
      provides: "Committed OpenAPI snapshot at the new-repo root"
      contains: "openapi"
    - path: "/tmp/zeroth-studio-split/apps/studio/src/api/types.gen.ts"
      provides: "Generated TypeScript types consumed by the frontend"
      contains: "paths"
  key_links:
    - from: "/tmp/zeroth-studio-split/.github/workflows/ci.yml"
      to: "/tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json"
      via: "generate:api script + git diff --exit-code drift gate"
      pattern: "git diff --exit-code"
    - from: "/tmp/zeroth-studio-split/README.md"
      to: "https://github.com/rrrozhd/zeroth"
      via: "markdown link"
      pattern: "github\\.com/rrrozhd/zeroth"
    - from: "/tmp/zeroth-studio-split/apps/studio/package.json"
      to: "openapi/zeroth-core-openapi.json"
      via: "generate:api script path"
      pattern: "openapi/zeroth-core-openapi\\.json"
---

<objective>
Wave 3: Add all bootstrap files to /tmp/zeroth-studio-split, commit, push, and verify CI goes green. This delivers STUDIO-02 (independent CI), STUDIO-04 (cross-linking + compat matrix), and STUDIO-05 (types generated from OpenAPI via a drift-gated snapshot).

**CRITICAL WORKING DIRECTORY NOTE:** This plan mutates `/tmp/zeroth-studio-split/` and pushes to `rrrozhd/zeroth-studio`. No changes to /Users/dondoe/coding/zeroth.

Output: A fully-bootstrapped zeroth-studio repo with green CI on main.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/29-studio-repo-split/29-CONTEXT.md
@.planning/phases/29-studio-repo-split/29-RESEARCH.md
@.planning/phases/29-studio-repo-split/29-01-SUMMARY.md
@.planning/phases/29-studio-repo-split/29-02-SUMMARY.md

<interfaces>
<!-- Pinned versions (from 29-RESEARCH §Standard Stack): -->
- Node: 22 LTS (matches apps/studio/Dockerfile FROM node:22-alpine)
- openapi-typescript: ^7.13 (already a devDep)
- ESLint: 9.x flat config (added in Plan 01 Task 3)

<!-- CI template (verified in 29-RESEARCH §CI template): -->
Steps: checkout → setup-node (22) → npm ci → lint → typecheck → build → test → generate:api → git diff --exit-code src/api/types.gen.ts

<!-- The OpenAPI snapshot source: the same file committed in Plan 01 Task 1 at
     /Users/dondoe/coding/zeroth/openapi/zeroth-core-openapi.json. It was NOT
     carried by filter-repo (that path is not in the --path list), so we must
     copy it in manually. -->
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Copy OpenAPI snapshot, generate types, add repo-level bootstrap files (LICENSE, CHANGELOG, CONTRIBUTING, README, .gitignore)</name>
  <files>
    /tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json,
    /tmp/zeroth-studio-split/apps/studio/src/api/types.gen.ts,
    /tmp/zeroth-studio-split/LICENSE,
    /tmp/zeroth-studio-split/CHANGELOG.md,
    /tmp/zeroth-studio-split/CONTRIBUTING.md,
    /tmp/zeroth-studio-split/README.md,
    /tmp/zeroth-studio-split/.gitignore
  </files>
  <action>
**Working directory: /tmp/zeroth-studio-split**

1. **Copy the OpenAPI snapshot** from zeroth-core (Plan 01 Task 1) into the new repo root:
```bash
mkdir -p /tmp/zeroth-studio-split/openapi
cp /Users/dondoe/coding/zeroth/openapi/zeroth-core-openapi.json /tmp/zeroth-studio-split/openapi/zeroth-core-openapi.json
```

2. **Install deps and generate TypeScript types** from the snapshot:
```bash
cd /tmp/zeroth-studio-split/apps/studio
npm ci
# generate:api script was added in Plan 01 Task 3; it reads openapi/... at repo root (two levels up from apps/studio)
# Verify the relative path works:
node -e "const p=require('path').resolve('../../openapi/zeroth-core-openapi.json'); require('fs').accessSync(p); console.log('OK',p)"

# Because the script uses a bare relative path "openapi/zeroth-core-openapi.json",
# it's resolved from the script's cwd — which is apps/studio. That's wrong for
# this layout. Fix the script to use "../../openapi/zeroth-core-openapi.json":
python -c "
import json
p='package.json'
d=json.load(open(p))
d['scripts']['generate:api']='openapi-typescript ../../openapi/zeroth-core-openapi.json -o src/api/types.gen.ts'
json.dump(d,open(p,'w'),indent=2); open(p,'a').write('\n')
"
grep generate:api package.json

# Generate types
npm run generate:api
# Confirm the file was created with paths interface
test -f src/api/types.gen.ts && grep -q "paths" src/api/types.gen.ts

# Make sure lint passes on the generated file (29-RESEARCH Pitfall 6).
# The eslint.config.js from Plan 01 Task 3 already ignores src/api/types.gen.ts.
npm run lint
npm run typecheck
npm run build
npm run test
```

If any of these fail, STOP and fix in-place before committing — the bootstrap commit must produce a green working tree.

3. **Create LICENSE (Apache-2.0)**. Use the standard Apache 2.0 license text. Copy from zeroth-core if present (`/Users/dondoe/coding/zeroth/LICENSE`) to ensure consistency, or from the canonical https://www.apache.org/licenses/LICENSE-2.0.txt.
```bash
cp /Users/dondoe/coding/zeroth/LICENSE /tmp/zeroth-studio-split/LICENSE 2>/dev/null || {
  echo "zeroth-core LICENSE missing, download from apache.org"
  curl -sSL https://www.apache.org/licenses/LICENSE-2.0.txt > /tmp/zeroth-studio-split/LICENSE
}
```

4. **Create CHANGELOG.md** (keepachangelog format):
```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] — 2026-04-11

### Added
- Initial split from the [rrrozhd/zeroth](https://github.com/rrrozhd/zeroth) monorepo (Phase 29 of v3.0).
- Vue 3 + Vue Flow Studio frontend (`apps/studio/`) with full git history preserved via `git filter-repo`.
- Mockup design reference app (`apps/studio-mockups/`) — not build-verified in CI.
- Env-driven API base URL via `VITE_API_BASE_URL` (defaults to `/api/studio/v1` for legacy proxy setups).
- GitHub Actions CI: lint (ESLint 9 flat config), typecheck (vue-tsc), build (Vite), test (Vitest), and OpenAPI type drift gate.
- Committed OpenAPI snapshot at `openapi/zeroth-core-openapi.json` as source of truth for generated types (`apps/studio/src/api/types.gen.ts`).
- Standalone `nginx.conf` bundled into the Studio Dockerfile for self-contained deployment.
- Apache-2.0 LICENSE, CONTRIBUTING.md, and compatibility matrix in README.
```

5. **Create CONTRIBUTING.md** — short, practical. Model after zeroth-core's if present, otherwise a ~40-line minimum:
```markdown
# Contributing to zeroth-studio

Thanks for your interest in contributing! This repo is the Vue 3 + Vue Flow frontend for [zeroth-core](https://github.com/rrrozhd/zeroth).

## Development

Prereqs: Node 22 LTS, npm.

```bash
cd apps/studio
cp .env.example .env.local     # edit VITE_API_BASE_URL if your zeroth-core runs elsewhere
npm install
npm run dev
```

You need a running `zeroth-core` service. See the [zeroth-core README](https://github.com/rrrozhd/zeroth) for install and run instructions.

## Commits

We use [Conventional Commits](https://www.conventionalcommits.org/). Examples:
- `feat(canvas): add node duplication shortcut`
- `fix(inspector): prevent crash on empty selection`
- `chore(deps): bump vue-flow to 1.49`

## Before opening a PR

```bash
cd apps/studio
npm run lint
npm run typecheck
npm run build
npm run test
```

CI will re-run these plus an OpenAPI drift check. If the zeroth-core OpenAPI changes, regenerate types:

```bash
npm run generate:api
git add src/api/types.gen.ts
```

## Compatibility

See the [compatibility matrix in the README](./README.md#compatibility) for which zeroth-studio version pairs with which zeroth-core version.

## License

By contributing, you agree your contributions will be licensed under Apache-2.0 (see [LICENSE](./LICENSE)).
```

6. **Create README.md** at the repo root:
```markdown
# zeroth-studio

Vue 3 + Vue Flow frontend for [zeroth-core](https://github.com/rrrozhd/zeroth), the governed multi-agent runtime.

This is the canvas UI for authoring, inspecting, and (in future phases) operating zeroth workflows. It speaks to zeroth-core exclusively over HTTP — no shared Python source.

## Quickstart

```bash
# Prereqs: Node 22 LTS, npm, and a running zeroth-core service.
git clone https://github.com/rrrozhd/zeroth-studio.git
cd zeroth-studio/apps/studio
cp .env.example .env.local
# Edit .env.local if your zeroth-core is not at http://localhost:8000
npm install
npm run dev
```

## Repo layout

```
zeroth-studio/
├── apps/
│   ├── studio/           # Vue 3 + Vue Flow app (primary, CI-verified)
│   └── studio-mockups/   # Design reference, not build-verified
├── openapi/
│   └── zeroth-core-openapi.json   # Pinned spec, source of truth for generated types
├── .github/workflows/ci.yml
└── README.md, CHANGELOG.md, LICENSE, CONTRIBUTING.md
```

## OpenAPI type generation

Types in `apps/studio/src/api/types.gen.ts` are generated from `openapi/zeroth-core-openapi.json` via:

```bash
cd apps/studio
npm run generate:api
```

CI fails if the generated file drifts from the committed version. To update, regenerate the spec in zeroth-core (`uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json`), copy the file into this repo, and re-run `generate:api`.

## Compatibility

This repo is the frontend for [rrrozhd/zeroth](https://github.com/rrrozhd/zeroth) (the `zeroth-core` Python service). The matrix below shows verified pairings.

| zeroth-studio | zeroth-core | Notes                                    |
|---------------|-------------|------------------------------------------|
| 0.1.0         | 0.1.1       | Initial split — Phase 29 of v3.0         |

## License

[Apache-2.0](./LICENSE)
```

7. **Create repo-root .gitignore**:
```
node_modules
dist
*.tsbuildinfo
.env.local
.env.*.local
.DS_Store
```

Do NOT commit yet — Task 2 adds the CI workflow and then makes a single bootstrap commit.
  </action>
  <verify>
    <automated>cd /tmp/zeroth-studio-split && test -f LICENSE && test -f CHANGELOG.md && test -f CONTRIBUTING.md && test -f README.md && test -f .gitignore && test -f openapi/zeroth-core-openapi.json && test -f apps/studio/src/api/types.gen.ts && grep -q "0.1.0" CHANGELOG.md && grep -q "compat" README.md -i && grep -q "zeroth-core" README.md && grep -q "../../openapi/zeroth-core-openapi.json" apps/studio/package.json && cd apps/studio && npm run lint && npm run typecheck && npm run build && npm run test</automated>
  </verify>
  <done>All bootstrap files present at repo root, types.gen.ts generated from the committed snapshot, apps/studio passes lint+typecheck+build+test locally, package.json generate:api path is ../../openapi/zeroth-core-openapi.json.</done>
</task>

<task type="auto">
  <name>Task 2: Add GitHub Actions CI workflow, commit bootstrap, push, and verify CI green on main</name>
  <files>/tmp/zeroth-studio-split/.github/workflows/ci.yml, github:rrrozhd/zeroth-studio main</files>
  <action>
**Working directory: /tmp/zeroth-studio-split**

1. **Create `.github/workflows/ci.yml`** per 29-RESEARCH §CI template (adjusted for the correct relative paths and to split typecheck/build which Plan 01 Task 3 introduced):

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verify:
    name: Lint / Typecheck / Build / Test / Drift-check
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/studio
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node 22
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm
          cache-dependency-path: apps/studio/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Typecheck
        run: npm run typecheck

      - name: Build
        run: npm run build

      - name: Test
        run: npm run test

      - name: Regenerate OpenAPI types
        run: npm run generate:api

      - name: Verify generated types match committed
        run: git diff --exit-code src/api/types.gen.ts
```

Design notes:
- Single Node 22 job (29-RESEARCH §Node Version for CI — matches the Dockerfile)
- `working-directory: apps/studio` for all run steps so scripts execute in the package root
- `cache-dependency-path` points at the real lockfile (npm workspaces not used here)
- Drift gate is the LAST step so earlier failures surface first
- `apps/studio-mockups` is intentionally NOT built in CI per resolved Q1 ("design reference only")

2. **Commit and push the bootstrap**:
```bash
cd /tmp/zeroth-studio-split
git add LICENSE CHANGELOG.md CONTRIBUTING.md README.md .gitignore \
        .github/workflows/ci.yml \
        openapi/zeroth-core-openapi.json \
        apps/studio/src/api/types.gen.ts \
        apps/studio/package.json apps/studio/package-lock.json

git status   # sanity: no stray modifications outside intended files
git -c user.name="rrrozhd" -c user.email="rrrozhd@users.noreply.github.com" commit -m "chore(bootstrap): add LICENSE, CI, README with compat matrix, OpenAPI snapshot, generated types

Phase 29 of zeroth v3.0 — completes the repo split started in zeroth monorepo.
Delivers STUDIO-02 (independent CI), STUDIO-04 (cross-link + compat matrix),
STUDIO-05 (generated types from pinned OpenAPI spec with drift gate)."

git push origin main
```

3. **Wait for CI and verify green**:
```bash
# Poll the latest run on main until it finishes
sleep 10
RUN_ID=$(gh run list --repo rrrozhd/zeroth-studio --branch main --limit 1 --json databaseId --jq '.[0].databaseId')
echo "Watching run $RUN_ID"
gh run watch "$RUN_ID" --repo rrrozhd/zeroth-studio --exit-status
# --exit-status exits non-zero if the run failed.
# On success, confirm conclusion:
gh run view "$RUN_ID" --repo rrrozhd/zeroth-studio --json conclusion --jq '.conclusion'
# Expected: "success"
```

If CI fails:
- Read the logs with `gh run view "$RUN_ID" --repo rrrozhd/zeroth-studio --log-failed`
- Reproduce locally inside `/tmp/zeroth-studio-split/apps/studio`
- Fix, commit on main, push again, re-watch
- Do NOT force-push — just add new commits

Acceptable outcomes:
- Green CI on the first push (preferred)
- Green CI after at most 2 follow-up fix commits

Unacceptable:
- Merging failures into main
- Skipping the drift gate or lint step to "get it green"
  </action>
  <verify>
    <automated>gh run list --repo rrrozhd/zeroth-studio --branch main --limit 1 --json conclusion,status --jq '.[0] | select(.status=="completed" and .conclusion=="success") | "OK"' | grep -q OK</automated>
  </verify>
  <done>.github/workflows/ci.yml exists on main, the bootstrap commit is pushed, the latest CI run on main completed with conclusion=success, drift gate passed.</done>
</task>

</tasks>

<verification>
- All seven bootstrap artifacts present on rrrozhd/zeroth-studio main: LICENSE, CHANGELOG.md, CONTRIBUTING.md, README.md, .gitignore, .github/workflows/ci.yml, openapi/zeroth-core-openapi.json
- apps/studio/src/api/types.gen.ts committed and consistent with the snapshot
- apps/studio/package.json generate:api path is `../../openapi/zeroth-core-openapi.json`
- README contains the compat matrix with `0.1.0 ↔ 0.1.1`
- README links to `https://github.com/rrrozhd/zeroth`
- GitHub Actions main branch run conclusion is `success`
- `gh repo view rrrozhd/zeroth-studio --json isPrivate` is `false`

Phase-level cross-check for STUDIO-03 (HTTP-only, no Python imports):
```bash
grep -r "^from zeroth\|^import zeroth" /tmp/zeroth-studio-split/apps/studio /tmp/zeroth-studio-split/apps/studio-mockups
# Expected: no matches
```
</verification>

<success_criteria>
1. STUDIO-02: CI pipeline (lint, typecheck, build, test, drift-check) is green on main
2. STUDIO-04: README contains the cross-link to zeroth-core and the initial compat matrix
3. STUDIO-05: types.gen.ts is generated from the committed openapi snapshot, drift gate verifies it stays in sync
4. STUDIO-03: grep confirms no Python imports anywhere under apps/
5. No reliance on runtime HTTP fetch for type generation (offline-reproducible)
</success_criteria>

<output>
After completion, create `.planning/phases/29-studio-repo-split/29-03-SUMMARY.md` covering:
- CI run URL + conclusion (success)
- types.gen.ts size and top-level shape
- Bootstrap commit SHA
- Any deviations from the CI template (e.g. extra steps added to fix Pitfall 6)
</output>
