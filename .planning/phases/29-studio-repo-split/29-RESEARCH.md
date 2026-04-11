# Phase 29: Studio Repo Split - Research

**Researched:** 2026-04-11
**Domain:** git history extraction, Vue 3 / Vite frontend tooling, OpenAPI-driven type generation, cross-repo CI
**Confidence:** HIGH

## Summary

Phase 29 is a mechanical packaging move: lift `apps/studio/`, `apps/studio-mockups/`, and (the logical concept of) `tests/studio/` into a brand-new public repo `rrrozhd/zeroth-studio` with full git history, wire up independent GitHub Actions CI, and replace the current runtime OpenAPI fetch (`http://localhost:8000/openapi.json`) with a committed spec snapshot + drift gate. The split is low-risk because the frontend has no source-level Python dependency — all coupling is HTTP.

Three concrete gotchas the planner must handle:
1. **`tests/studio/` is effectively empty on disk** — only `__pycache__/*.pyc` remains. The real Python test that exercises the Studio API (`tests/test_studio_api.py`) imports `zeroth.core.graph.repository`, `zeroth.core.service.studio_api`, `zeroth.core.service.bootstrap`, and `zeroth.core.storage.async_sqlite`. **It cannot move** to the JS repo. Context (D-01) assumed `tests/studio/` still held meaningful code; it does not. The planner should treat `tests/studio/__pycache__/` as "delete" and leave `tests/test_studio_api.py` in zeroth-core as server-side API contract coverage. [VERIFIED: filesystem scan + grep]
2. **`apps/studio/src/api/client.ts` currently hard-codes `/api/studio/v1`** with no `VITE_API_BASE_URL` usage and no `.env*` files exist. D-09 requires adding env-driven base URL, `.env.example`, `.env.d.ts` augmentation, and updating `client.ts`. This is a real code change, not just a move. [VERIFIED: Read of client.ts, env.d.ts, apps/studio/.env* glob]
3. **zeroth-core exposes OpenAPI only at runtime** via `FastAPI(...)` — there is no committed spec file anywhere in the repo. The reproducible-offline requirement in D-06 means someone has to boot zeroth-core once to export the spec, commit it to `openapi/zeroth-core-openapi.json` in the new studio repo, and re-export whenever zeroth-core ships a new API. [VERIFIED: grep of src/zeroth/core/service/app.py + glob for openapi*.json]

**Primary recommendation:** Execute as a 5-wave plan: (0) freeze + snapshot OpenAPI from a running zeroth-core, (1) git filter-repo to a scratch directory with path filters + path-renames, (2) bootstrap new-repo files (LICENSE, CHANGELOG, CONTRIBUTING, README with compat matrix, `.env.example`, `.github/workflows/ci.yml`, `openapi/zeroth-core-openapi.json`), (3) `gh repo create` + force-push to `main`, (4) delete the three paths from zeroth-core + update its README cross-link.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01 Scope of files to move**
- `apps/studio/` — the real Vue 3 + Vue Flow frontend (PRIMARY)
- `apps/studio-mockups/` — the mockup app, kept as design reference inside the new repo
- `tests/studio/` — Studio-specific tests, currently in the zeroth-core test tree

**D-02 History preservation method**
- Use `git filter-repo` (modern, reliable, preserves full history of moved paths only)
- Filter on the three paths above so the new repo has zero zeroth-core history noise
- Tool installed via `pipx install git-filter-repo` if not already present

**D-03 Source cleanup in zeroth-core**
- After the split lands, delete `apps/studio/`, `apps/studio-mockups/`, and `tests/studio/` from zeroth-core in the same phase
- Single source of truth immediately — no transitional duplication
- Update `pytest.ini` / `pyproject.toml` test discovery if needed

**D-04 Repo creation and push**
- Claude creates the GitHub repo via `gh repo create rrrozhd/zeroth-studio --public --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"`
- Push the filtered history to `main` (force-push acceptable since the repo is fresh)
- Set the default branch to `main`

**D-05 Compatibility matrix location**
- Lives in zeroth-studio README only (the dependent side — where the version pin lives)
- Format: a markdown table mapping zeroth-studio versions ↔ zeroth-core versions
- Initial entry: `zeroth-studio 0.1.0 ↔ zeroth-core 0.1.1`

**D-06 OpenAPI sync wiring**
- `npm run generate:api` script in zeroth-studio runs `openapi-typescript` against a local or pinned zeroth-core OpenAPI spec
- Generated types committed to the repo (`src/api/types.gen.ts` or similar)
- CI gate: a job runs `generate:api` and `git diff --exit-code` — fails the build if generated types drift from checked-in version
- Source of truth: a snapshot of the zeroth-core OpenAPI JSON committed at `openapi/zeroth-core-openapi.json` so generation is reproducible offline

**D-07 CI pipeline for zeroth-studio**
- GitHub Actions (consistent with zeroth-core)
- Workflow `.github/workflows/ci.yml`: install → lint (ESLint) → typecheck (vue-tsc) → build (vite build) → test (vitest run) → openapi-drift-check
- Triggers: push and PR to `main`
- Node version: pinned to whatever `apps/studio/package.json` currently uses

**D-08 README cross-linking**
- zeroth-studio README links to `https://github.com/rrrozhd/zeroth` (zeroth-core)
- zeroth-core README gains a "Studio" section linking to `https://github.com/rrrozhd/zeroth-studio`
- Both have a brief one-line description of what the other provides

**D-09 Local development workflow**
- After clone: `npm install && npm run dev` works against any running zeroth-core instance
- Default `VITE_API_BASE_URL` from `.env.example`, overridable via `.env.local`
- Development workflow does not require zeroth-core source — only a running service (local docker, hosted, etc.)

### Claude's Discretion
- Exact CI matrix details (Node 20 vs 22, single or multi-version)
- Whether to publish zeroth-studio to npm or leave as source-only
- Whether to use pnpm/npm/yarn (default: keep whatever apps/studio currently uses)
- Initial CHANGELOG.md and LICENSE files (Apache-2.0 like zeroth-core)
- README structure and screenshots

### Deferred Ideas (OUT OF SCOPE)
- npm publishing of zeroth-studio (source-only repo for v0)
- Per-version git tags backfilled into zeroth-studio (complex, not required)
- Phases 24-26 work (continues in zeroth-studio after this phase ships)
- Storybook / component catalog
- E2E tests across both repos
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| STUDIO-01 | Public repo `rrrozhd/zeroth-studio` with full git history preserved | §"git filter-repo procedure" below — verified command, filter paths, Node 22 + tool availability |
| STUDIO-02 | Independent CI pipeline (lint, typecheck, build, test) passing on default branch | §"CI pipeline template" — ESLint is NOT currently configured in apps/studio (finding P-4); must add it |
| STUDIO-03 | HTTP-only consumption — no Python imports, no shared source tree | §"Cross-reference audit" — no Python imports found in apps/studio/**, only one HTTP surface (`client.ts`) |
| STUDIO-04 | READMEs cross-link + compatibility matrix documented | §"Compat matrix template" — initial entry: 0.1.0 ↔ 0.1.1 |
| STUDIO-05 | Frontend types generated from zeroth-core OpenAPI via `openapi-typescript` | §"OpenAPI snapshot + drift gate" — `openapi-typescript@7.13.0` already a devDep, current script uses runtime HTTP fetch which must be rewritten |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Context efficiency:** plans for zeroth-studio (future work) should not assume agents read root PLAN.md. Not directly actionable for this phase.
- **Progress logging:** every implementation session MUST use the `progress-logger` skill — applies to plan execution, not research.
- **Build & test commands** for zeroth-core: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`. Relevant because Phase 29 also deletes paths from zeroth-core and must verify its test suite and ruff check still pass after deletion.
- **Project layout:** `src/zeroth/` (Python package), `tests/` (pytest). Confirms that `apps/` and `tests/studio/` are not part of the Python package and can be removed without import impact.

## Standard Stack

### Core — Already Pinned in `apps/studio/package.json`
| Library | Version | Purpose | Source |
|---------|---------|---------|--------|
| vue | ^3.5 | Frontend framework | [VERIFIED: package.json] |
| @vue-flow/core | ^1.48 | Node-based canvas engine | [VERIFIED: package.json, lockfile 1.48.2] |
| @vue-flow/minimap | ^1.5 | Minimap overlay | [VERIFIED: package.json] |
| @vue-flow/controls | ^1.1 | Zoom/pan controls | [VERIFIED: package.json] |
| @vue-flow/background | ^1.3 | Canvas background patterns | [VERIFIED: package.json] |
| pinia | ^3.0 | State management | [VERIFIED: package.json] |
| @dagrejs/dagre | ^3.0 | Auto-layout | [VERIFIED: package.json] |
| tailwindcss | ^4.2 | Styling (Vite plugin variant) | [VERIFIED: package.json] |

### Tooling — Already Pinned
| Tool | Version | Purpose | Source |
|------|---------|---------|--------|
| vite | ^6.0 | Build tool | [VERIFIED: package.json] |
| @vitejs/plugin-vue | ^6.0 | Vue SFC support | [VERIFIED: package.json] |
| @tailwindcss/vite | ^4.2 | Tailwind v4 Vite plugin | [VERIFIED: package.json] |
| typescript | ^5.8 | Types | [VERIFIED: package.json] |
| vue-tsc | ^2.2 | Vue-aware TS checker | [VERIFIED: package.json] |
| openapi-typescript | ^7.13 | Generate TS types from OpenAPI | [VERIFIED: npm view openapi-typescript version → 7.13.0, published 2026-02-11] |
| vitest | ^3.1 | Test runner | [VERIFIED: package.json] |

### Tooling — To Add in Phase 29
| Tool | Recommended | Purpose | Why |
|------|-------------|---------|-----|
| eslint | ^9.x (flat config) | Linting | D-07 requires lint in CI; **apps/studio has no ESLint config today** [VERIFIED: no `.eslintrc*` or `eslint.config.*` in apps/studio] |
| @vue/eslint-config-typescript | latest | Vue+TS ESLint preset | Standard Vue 3 lint stack |
| eslint-plugin-vue | latest | Vue SFC lint rules | Standard Vue 3 lint stack |

### Package Manager
**Decision: npm** — `apps/studio/package-lock.json` exists (lockfileVersion 3), no pnpm-lock.yaml or yarn.lock. [VERIFIED: filesystem scan]. This matches the existing Dockerfile which uses `npm ci`.

### Git History Extraction
| Tool | Version | Purpose | Source |
|------|---------|---------|--------|
| git-filter-repo | a40bce548d2c | Path-based history rewriting | [VERIFIED: `git filter-repo --version` → `a40bce548d2c`, already installed at `/opt/homebrew/bin/git-filter-repo`] |
| gh | 2.88.0 | GitHub repo creation + push | [VERIFIED: `gh --version` → 2.88.0 (2026-03-10)] |
| node | v25.2.1 | Local dev | [VERIFIED: `node --version`] |
| npm | 11.9.0 | Local dev | [VERIFIED: `npm --version`] |

### Node Version for CI
**Recommendation: Node 22 LTS** (single version). Rationale:
- The existing Dockerfile uses `node:22-alpine` — matches production. [VERIFIED: apps/studio/Dockerfile line 2]
- Single-version matrix minimizes CI time; Studio is browser-targeted JS, not a library.
- D-07 says "pinned to whatever apps/studio currently uses" → the Dockerfile is the authoritative signal.

[ASSUMED] Node 22 LTS is the only version that needs CI validation. If the planner wants multi-version, Node 20 LTS is the alternative, but adds no value for a browser build target.

## Architecture Patterns

### Recommended `zeroth-studio` Repository Layout (Post-Split)
```
zeroth-studio/
├── apps/
│   ├── studio/              # Vue 3 + Vue Flow app (preserved from monorepo)
│   └── studio-mockups/      # Mockup reference app (preserved)
├── openapi/
│   └── zeroth-core-openapi.json    # Committed snapshot, source of truth for drift gate
├── .github/workflows/
│   └── ci.yml               # lint → typecheck → build → test → drift-check
├── .env.example             # VITE_API_BASE_URL default
├── README.md                # compat matrix + cross-link + dev quickstart
├── CHANGELOG.md             # keepachangelog
├── LICENSE                  # Apache-2.0
├── CONTRIBUTING.md
└── .gitignore
```

**Why preserve the `apps/studio/` + `apps/studio-mockups/` nesting instead of flattening:**
- `git filter-repo --path apps/studio --path apps/studio-mockups` preserves the **existing directory prefix** in the rewritten history. Flattening would require additional `--path-rename` steps AND rewrite every commit's paths, breaking history simplicity.
- Two apps in one repo (primary + mockups) is a legitimate npm monorepo pattern. Keep them side-by-side; only `apps/studio/` is wired into CI.
- D-07 says CI targets `apps/studio` build; mockups stay as a design reference only.

### Pattern 1: git filter-repo Path Extraction
**What:** Use `git filter-repo --path` (repeat flag) in a fresh clone to retain only the specified paths and drop all unrelated history, refs, and blobs.

**When to use:** Always, for this phase. Subtree split is an older pattern that ties you to a single directory; filter-repo is the modern standard.

**Procedure (for the planner — verified against current tool version):**
```bash
# 1. Make a throwaway clone (never run filter-repo on the working repo)
git clone --no-local /Users/dondoe/coding/zeroth /tmp/zeroth-studio-split
cd /tmp/zeroth-studio-split

# 2. Rewrite history to keep only the three paths
git filter-repo \
  --path apps/studio \
  --path apps/studio-mockups \
  --path tests/studio

# 3. Verify the result
git log --oneline | head
ls -la                      # should contain only apps/ and tests/
git log --all --oneline | wc -l   # expected: significantly fewer than full repo
```
[VERIFIED: git-filter-repo is installed at /opt/homebrew/bin/git-filter-repo]

**Expected commit count:** The full monorepo history touching these paths is **32 commits** across the three paths. [VERIFIED: `git log --oneline -- apps/studio apps/studio-mockups tests/studio tests/test_studio_api.py | wc -l → 32`]. This is the lower bound for the filtered repo's history length.

**Why `--no-local`:** Without it, `git clone` hardlinks the objects directory and `filter-repo` would rewrite history in the source repo's object store. [CITED: https://github.com/newren/git-filter-repo/blob/main/Documentation/git-filter-repo.txt — "For safety, it refuses to run on a repo that is not a fresh clone"]. `filter-repo` actively refuses non-fresh clones unless `--force` is passed.

### Pattern 2: OpenAPI Snapshot Generation
**What:** Boot zeroth-core once, curl its `/openapi.json`, commit the result.

**Procedure:**
```bash
# In zeroth-core working tree
uv run uvicorn zeroth.core.service.app:app --port 8000 &
APP_PID=$!
sleep 2
curl -s http://localhost:8000/openapi.json | python -m json.tool > /tmp/zeroth-core-openapi.json
kill $APP_PID

# Copy into the new studio repo
cp /tmp/zeroth-core-openapi.json /path/to/zeroth-studio/openapi/zeroth-core-openapi.json
```

**Why not a build-time extraction script in zeroth-core:** Phase 29 is packaging-only. An automated "emit OpenAPI as part of zeroth-core release" is a legitimate follow-up but belongs to a future phase or to Phase 32 (Reference Docs, which also needs the same spec).

### Pattern 3: `openapi-typescript` v7 Invocation
**What:** `openapi-typescript` v7 reads an OpenAPI 3.x spec and emits a single `.ts` file with `paths` and `components` interfaces.

**Current (runtime fetch, broken for the new repo):**
```json
"generate-types": "openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts"
```
[VERIFIED: apps/studio/package.json line 10]

**Replacement (offline, reproducible):**
```json
"generate:api": "openapi-typescript openapi/zeroth-core-openapi.json -o src/api/types.gen.ts"
```

**Why the path `src/api/types.gen.ts`:** D-06 wording ("`src/api/types.gen.ts` or similar"). The `.gen.ts` suffix signals to humans and ESLint that this file is generated; a `/* eslint-disable */` header from the tool complements it. [CITED: https://openapi-ts.dev/introduction — openapi-typescript v7 outputs `.ts`, not `.d.ts`, by default]

### Pattern 4: CI Drift Gate
**What:** Run `generate:api` in CI, then `git diff --exit-code` on the generated file.

```yaml
- name: Generate API types
  run: npm run generate:api
- name: Verify no drift
  run: git diff --exit-code src/api/types.gen.ts
```

**Why `--exit-code`:** Returns 1 on any diff, failing the CI job. Standard idiom for "committed artifact must match regenerated artifact." [CITED: https://git-scm.com/docs/git-diff — "--exit-code ... exit with 1 if there were differences"]

### Anti-Patterns to Avoid
- **Subtree split instead of filter-repo** — `git subtree split` is slower, produces worse history, and has no multi-path support. filter-repo is the replacement tool the git community has standardized on since 2019. [CITED: https://git-scm.com/docs/git-filter-branch — "git-filter-branch ... is not recommended ... use an alternative history filtering tool such as git filter-repo"]
- **Runtime OpenAPI fetch in `generate:api`** — breaks offline builds, breaks CI (no zeroth-core available), breaks reproducibility. Always a committed snapshot.
- **Flattening `apps/studio/` to repo root during filter-repo** — adds `--path-rename apps/studio/:` and rewrites every commit's tree. Not worth it; monorepo-shaped layouts are a valid pattern.
- **Keeping `/api/studio/v1` hardcoded in `client.ts`** — breaks D-09. Must read from `import.meta.env.VITE_API_BASE_URL` with a sensible default.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Filter git history to specific paths | Custom `git-filter-branch` script | `git filter-repo` | Officially recommended by git.git; handles refs, tags, reflogs, and commit graph rewriting correctly |
| Generate TS types from OpenAPI | Hand-write interfaces | `openapi-typescript` (already a devDep) | Already in the project; maintained; emits union types correctly for `oneOf`/`anyOf` |
| Detect drift between source spec and generated types | Hash comparison | `git diff --exit-code` | Works with any generator; zero code; familiar to reviewers |
| Create a GitHub repo from CLI | `curl` to REST API | `gh repo create` | Auth is already handled; supports `--public`, `--description`, `--source`, `--push` flags |
| Parse `.env` files in Vite app | Custom loader | Vite's built-in `import.meta.env.VITE_*` | First-class support; respects `.env.local` override order |

## Runtime State Inventory

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| **Stored data** | None. The studio is a pure frontend — no databases, no Mem0, no ChromaDB. Backend workflow storage stays in zeroth-core (SQLite/Postgres under `zeroth.core.graph.*`). | None — verified by Grep of apps/studio for database/ORM references |
| **Live service config** | None. The studio has no external service registrations (n8n, Datadog, Tailscale, Cloudflare). | None — verified by directory inspection |
| **OS-registered state** | None. No launchd/systemd/pm2/Task Scheduler entries reference apps/studio. | None — studio runs only via `npm run dev` or the Docker image |
| **Secrets/env vars** | `VITE_API_BASE_URL` — **does not yet exist**. Must be introduced as part of D-09. No existing env vars referencing studio. | Introduce env var + `.env.example` + `src/env.d.ts` typing + `client.ts` refactor |
| **Build artifacts / installed packages** | `apps/studio/node_modules/`, `apps/studio/dist/`, `apps/studio/tsconfig.tsbuildinfo`, `apps/studio-mockups/node_modules/`, `apps/studio-mockups/dist/`. git-filter-repo will not touch `.gitignore`d working tree files, but the new repo's clone will be clean. | Ensure `.gitignore` in the new repo excludes `node_modules/`, `dist/`, `tsconfig.tsbuildinfo`, `.env.local` |
| **External references (non-standard)** | `apps/studio/Dockerfile` references nginx config "mounted via docker-compose volume (D-11)" — that docker-compose file lives in zeroth-core and will NOT move. | Either update the Dockerfile comment to note that zeroth-studio is served standalone, or bundle a standalone nginx.conf. Recommend: bundle a minimal nginx.conf in the new repo so the Dockerfile is self-contained |

### Cross-Reference Audit (Source-Level Coupling)
- **Python → apps/studio:** No imports found. [VERIFIED: Grep of `from apps|apps\.studio` → only matches in `.planning/` docs]
- **apps/studio → zeroth-core (TS/Vue):** Zero source-level imports. The only contact point is HTTP via `apps/studio/src/api/client.ts` which fetches `/api/studio/v1/*`. [VERIFIED: Read of client.ts]
- **tests/studio/ → zeroth-core:** `tests/studio/` contains ONLY stale `__pycache__` bytecode. The actual Python test `tests/test_studio_api.py` lives at `tests/` root and imports `zeroth.core.graph.repository`, `zeroth.core.service.studio_api`, `zeroth.core.service.bootstrap`, `zeroth.core.storage.async_sqlite`. **This test CANNOT move to the JS repo** — it is a server-side contract test and belongs in zeroth-core permanently. [VERIFIED: Read of tests/test_studio_api.py lines 1-30]

**Critical finding for planner:** D-01 and D-03 say "move `tests/studio/`" and "delete `tests/studio/` from zeroth-core in the same phase." Taking this literally is correct — the directory is empty of source. But the planner should NOT also delete `tests/test_studio_api.py`, which is unrelated despite the name. A wave task should explicitly state: "delete `tests/studio/` directory only; leave `tests/test_studio_api.py` in place."

## Common Pitfalls

### Pitfall 1: Cloning with hardlinks breaks filter-repo
**What goes wrong:** Running `git filter-repo` against a clone made with default `git clone /path/to/local/repo` fails with "Aborting: Refusing to overwrite repo history since this does not look like a fresh clone. (Expected git dir: packed-refs; got: ...)"
**Why it happens:** Local clones hardlink objects by default; filter-repo's safety check sees "unexpected" content.
**How to avoid:** Always use `git clone --no-local` when making the source clone for filter-repo.
**Warning signs:** "Refusing to overwrite" in the filter-repo output.
[CITED: https://github.com/newren/git-filter-repo/blob/main/Documentation/git-filter-repo.txt — "FRESH CLONES" section]

### Pitfall 2: `gh repo create ... --push` expects a local remote
**What goes wrong:** `gh repo create rrrozhd/zeroth-studio --public --source=. --push` fails if the local repo has no commits or no `origin` remote set.
**Why it happens:** `--push` runs `git push -u origin HEAD`; filter-repo **removes** the origin remote as part of its safety contract.
**How to avoid:** Two-step the creation:
```bash
gh repo create rrrozhd/zeroth-studio --public \
  --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"
git remote add origin https://github.com/rrrozhd/zeroth-studio.git
git branch -M main
git push -u origin main
```
[CITED: https://github.com/newren/git-filter-repo/blob/main/Documentation/git-filter-repo.txt — "filter-repo will remove 'origin'" ; https://cli.github.com/manual/gh_repo_create]
**Warning signs:** `fatal: 'origin' does not appear to be a git repository` after filter-repo.

### Pitfall 3: OpenAPI spec has `operationId` collisions
**What goes wrong:** `openapi-typescript` generates valid TS, but downstream code that tries to use `operationId`-keyed helpers (e.g., `openapi-fetch`) gets duplicate key errors.
**Why it happens:** The zeroth-core codebase already acknowledges this — `src/zeroth/core/service/app.py` line 272 has a comment: `# excluded from OpenAPI spec to avoid duplicate operationIds (per D-06, Pitfall 3)`. Some routes are intentionally excluded to prevent this.
**How to avoid:** Snapshot the spec AFTER any route-registration changes; verify by running `jq '[.paths[][] | .operationId] | group_by(.) | map(select(length > 1))' openapi/zeroth-core-openapi.json` — must return `[]`.
**Warning signs:** openapi-typescript output compiles but `openapi-fetch` types are `never`. [VERIFIED: Grep of src/zeroth/core/service/app.py]

### Pitfall 4: Dockerfile references a non-existent nginx config
**What goes wrong:** `apps/studio/Dockerfile` comment says "Nginx config is mounted via docker-compose volume (D-11)". That docker-compose file lives in zeroth-core and stays there. A user running `docker build` in zeroth-studio followed by `docker run` gets a bare nginx image with no routing config.
**How to avoid:** In Phase 29, either (a) add a minimal `nginx.conf` to `apps/studio/` and `COPY` it in the Dockerfile, or (b) remove the Dockerfile from zeroth-studio with a README note that production deployment is out of scope for v0. Recommend (a) for self-containment.
**Warning signs:** Users report "nginx default welcome page" when running the studio Docker image standalone. [VERIFIED: Read of apps/studio/Dockerfile]

### Pitfall 5: `tests/studio/` is empty but git history still holds source
**What goes wrong:** Planner assumes `tests/studio/` on disk reflects what git will carry into the new repo. It doesn't — `git filter-repo --path tests/studio` preserves **every historical commit that touched** that path, including commits that added and later removed the source .py files.
**How to avoid:** This is actually desired — filter-repo preserves the full history of moves AND deletions inside those paths, which is good (searchable history). Just document it in the RESEARCH for the planner so there's no surprise when the filtered repo has more commits than on-disk files suggest.
**Warning signs:** None — this is correct behavior, just counter-intuitive.

### Pitfall 6: Generated types committed but ESLint/tsc flags them as unused
**What goes wrong:** `openapi-typescript` emits large union types; if any aren't imported by source code, `noUnusedLocals` (currently `true` in `apps/studio/tsconfig.json`) triggers on nothing, but the file itself has an unused `export` that ESLint `no-unused-exports` might flag.
**How to avoid:** (1) Add `/* eslint-disable */` to the top of the generated file (openapi-typescript v7 supports `--enable-all-comments` and emits linting-friendly code by default; verify). (2) Use `noUnusedLocals` only inside `src/` — generated file is still a `.ts` under `src/api/`, so this applies. Test locally before wiring CI. [VERIFIED: apps/studio/tsconfig.json has `noUnusedLocals: true`]

## Code Examples

### git filter-repo extraction
```bash
# Source: https://github.com/newren/git-filter-repo/blob/main/Documentation/git-filter-repo.txt
cd /tmp
git clone --no-local /Users/dondoe/coding/zeroth zeroth-studio-split
cd zeroth-studio-split

# Multi-path filter — keeps the paths AND their history, drops everything else
git filter-repo \
  --path apps/studio \
  --path apps/studio-mockups \
  --path tests/studio

# Confirm
git log --oneline                # should show ~32+ commits
ls                               # should show apps/ and tests/ only
```

### gh repo create + push sequence
```bash
# Source: https://cli.github.com/manual/gh_repo_create
gh repo create rrrozhd/zeroth-studio \
  --public \
  --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"

cd /tmp/zeroth-studio-split
git remote add origin https://github.com/rrrozhd/zeroth-studio.git
git branch -M main
git push -u origin main
```

### OpenAPI snapshot + generate invocation
```bash
# Source: verified against current apps/studio/package.json + openapi-typescript docs
# 1. Snapshot from a running zeroth-core
uv run uvicorn zeroth.core.service.app:app --port 8000 &
sleep 2
curl -s http://localhost:8000/openapi.json | python -m json.tool \
  > openapi/zeroth-core-openapi.json
kill %1

# 2. Generate TS types (runs in CI and locally)
npx openapi-typescript openapi/zeroth-core-openapi.json -o src/api/types.gen.ts
```

### client.ts env-driven base URL (replaces current hardcoded value)
```typescript
// Source: Vite env handling — https://vite.dev/guide/env-and-mode.html
const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? '/api/studio/v1'

// env.d.ts addition:
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL?: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

### .env.example
```ini
# URL of a running zeroth-core service (include the /api/studio/v1 prefix if proxying is off)
VITE_API_BASE_URL=http://localhost:8000/api/studio/v1
```

### GitHub Actions CI template (`.github/workflows/ci.yml`)
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/studio
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: apps/studio/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck   # add script: "vue-tsc -b --noEmit"
      - run: npm run build
      - run: npm run test
      - run: npm run generate:api
      - name: Check for OpenAPI type drift
        run: git diff --exit-code src/api/types.gen.ts
```

### Compatibility matrix (zeroth-studio README section)
```markdown
## Compatibility

This repo is the frontend for [rrrozhd/zeroth](https://github.com/rrrozhd/zeroth) (the `zeroth-core` Python service).
The matrix below shows verified pairings.

| zeroth-studio | zeroth-core | Notes                                    |
|---------------|-------------|------------------------------------------|
| 0.1.0         | 0.1.1       | Initial split — Phase 29 of v3.0         |
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `git filter-branch` | `git filter-repo` | git 2.24 (2019) — filter-branch officially deprecated with a manpage warning | Much faster, safer, supports multi-path; the current de facto standard |
| Runtime OpenAPI fetch (`openapi-typescript http://...`) | Committed spec snapshot | Has always been better practice; the current apps/studio script is the outlier | Reproducible builds, offline CI, drift detection |
| Subtree split (`git subtree split --prefix=apps/studio`) | `git filter-repo --path` | ~2020 | Multi-path support, full history preservation, cleaner refs |

**Deprecated/outdated:**
- `git subtree split` for repo extraction: still works but offers no advantage over filter-repo. [CITED: https://git-scm.com/docs/git-filter-branch]

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| git-filter-repo | Wave 1 history extraction | ✓ | a40bce548d2c | — |
| gh (GitHub CLI) | Wave 3 repo creation + push | ✓ | 2.88.0 | Manual `curl` to GitHub REST API |
| node | Wave 2 dependency install + build verification | ✓ | v25.2.1 | — |
| npm | Wave 2 dependency install | ✓ | 11.9.0 | — |
| uv (for running zeroth-core to snapshot OpenAPI) | Wave 0 OpenAPI snapshot | Assumed present (phase 27/28 used it) | — | `python -m uvicorn` if uv unavailable |
| jq (optional, OpenAPI validation) | Wave 0 verification | [NOT CHECKED] | — | `python -c "import json"` equivalent |
| GitHub authentication for rrrozhd user | Wave 3 push | Assumed present | — | **BLOCKER if not** — user must authenticate `gh auth login` |

**Missing dependencies with no fallback:** None identified, but GitHub CLI authentication for user `rrrozhd` must be verified before Wave 3.

**Missing dependencies with fallback:** None.

## Validation Architecture

### Test Framework (for the new zeroth-studio repo)
| Property | Value |
|----------|-------|
| Framework | vitest ^3.1 (already a devDep) |
| Config file | currently none — `vitest run` uses vite.config.ts. Add `vitest.config.ts` only if test setup diverges from vite config. |
| Quick run command | `npm run test` (maps to `vitest run`) |
| Full suite command | `npm run test` |

### Test Framework (for zeroth-core verification after deletion)
| Property | Value |
|----------|-------|
| Framework | pytest (existing) |
| Config file | `pyproject.toml [tool.pytest.ini_options]` — `testpaths = ["tests"]` |
| Quick run command | `uv run pytest -v tests/test_studio_api.py` |
| Full suite command | `uv run pytest -v --no-header -ra` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| STUDIO-01 | Public repo exists with preserved git history | smoke | `gh repo view rrrozhd/zeroth-studio --json isPrivate,defaultBranchRef`; `git log --oneline \| wc -l` (≥32) in fresh clone | N/A — verified externally |
| STUDIO-02 | CI passes on default branch | smoke | `gh run list --repo rrrozhd/zeroth-studio --branch main --limit 1 --json conclusion --jq '.[0].conclusion'` → `success` | N/A — GHA artifact |
| STUDIO-03 | No Python imports in studio sources | unit | `grep -r "^from zeroth\\|^import zeroth" apps/studio/ apps/studio-mockups/` must return empty | N/A — repo-wide grep |
| STUDIO-04 | Compat matrix documented in README | manual + smoke | `grep -q "zeroth-studio.*zeroth-core" README.md` | N/A — string match |
| STUDIO-05 | TS types generated from OpenAPI | integration | `npm run generate:api && git diff --exit-code src/api/types.gen.ts` | Wave 2 creates `src/api/types.gen.ts` |
| D-03 side effect | zeroth-core still passes tests after deletion | integration | `uv run pytest -v --no-header -ra` (must remain green) | Existing suite |
| D-09 | `.env.example` present and `VITE_API_BASE_URL` respected | unit | `test -f apps/studio/.env.example && grep -q VITE_API_BASE_URL apps/studio/.env.example` | Wave 2 creates |

### Sampling Rate
- **Per task commit:** Relevant subset — for file moves, verify `ls`; for config changes, run `npm run build` or `uv run pytest tests/test_studio_api.py`
- **Per wave merge:** Full CI equivalent — `npm run lint && npm run typecheck && npm run build && npm run test && npm run generate:api && git diff --exit-code` (new repo) + `uv run pytest && uv run ruff check src/` (zeroth-core)
- **Phase gate:** GHA CI green on zeroth-studio main + zeroth-core full test suite green after deletion

### Wave 0 Gaps
- [ ] Install ESLint + flat config for `apps/studio` — no `.eslintrc*` or `eslint.config.*` exists today [VERIFIED: filesystem scan]. Needed to satisfy D-07 "lint" step.
- [ ] Add `typecheck` npm script — current `build` script is `vue-tsc -b && vite build` which conflates typecheck and build. Split for better CI signal.
- [ ] Snapshot `openapi/zeroth-core-openapi.json` — no committed spec exists in zeroth-core.
- [ ] Create `nginx.conf` standalone for `apps/studio/Dockerfile` OR remove Dockerfile from scope.
- [ ] Add `VITE_API_BASE_URL` wiring in `client.ts` + `env.d.ts` + `.env.example`.
- [ ] Create `LICENSE` (Apache-2.0), `CHANGELOG.md` (keepachangelog, initial entry), `CONTRIBUTING.md` for zeroth-studio.
- [ ] Add "Studio" section to zeroth-core `README.md` with link to `https://github.com/rrrozhd/zeroth-studio`.

## Security Domain

### Applicable ASVS Categories
| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | no | No auth code in this phase — zeroth-core handles auth; studio is a dumb HTTP client |
| V3 Session Management | no | Same reason |
| V4 Access Control | no | Same reason |
| V5 Input Validation | marginal | `openapi-typescript`-generated types provide compile-time input shape checks at the fetch boundary — not runtime validation |
| V6 Cryptography | no | None in this phase |
| V14 Configuration | yes | `.env.example` must NOT contain real secrets; CI workflow must not embed tokens |

### Known Threat Patterns for This Phase
| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Accidentally committing `.env.local` with a real API token | Information Disclosure | Add `.env.local`, `.env*.local` to `.gitignore`; only commit `.env.example` with placeholder values |
| Force-pushing to a wrong repo during Wave 3 | Tampering | Gate `git push` behind `gh repo view rrrozhd/zeroth-studio` success; human confirm before push |
| OpenAPI snapshot leaking internal admin endpoints | Information Disclosure | The snapshot reflects the public `/api/studio/v1` router — no new exposure, but the planner should verify by inspecting the spec before commit that no admin routes leaked in |
| Force-push destroys history after initial push | Tampering | After the first successful `git push -u origin main`, any follow-up commit must be a regular push, not `--force` |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Node 22 LTS is sufficient for CI; no need for multi-version matrix | Standard Stack — Node Version for CI | Low — can add matrix later if a contributor hits a Node 20 issue |
| A2 | `uv` is installed on the machine running Wave 0 OpenAPI snapshot | Environment Availability | Low — zeroth-core is already installed via uv; phases 27/28 depend on it |
| A3 | GitHub CLI is authenticated as `rrrozhd` (or a user with write access to that org/user) | Environment Availability | **HIGH** — blocks Wave 3. Planner must include an explicit `gh auth status` verification task |
| A4 | The `jq` binary is present for OpenAPI validation | Environment Availability | Low — optional; Python one-liner is a fallback |
| A5 | `openapi-typescript` v7 emits an ESLint-friendly header (`/* eslint-disable */` or equivalent) | Pitfall 6 | Medium — if not, the generated file may fail lint. Planner should test locally before committing CI |
| A6 | No other files in the monorepo reference `apps/studio` or `tests/studio` paths (e.g., Makefiles, scripts, docs) | Cross-Reference Audit | Medium — a broader grep is warranted in Wave 1 before the zeroth-core delete |
| A7 | The `apps/studio/Dockerfile` is worth keeping standalone in zeroth-studio | Pitfall 4 / Runtime State Inventory | Low — can be removed from the new repo if it's not valuable; D-07 does not require a Docker build step |

**Assumed items requiring user confirmation before plan execution:** A3, A5, A7.

## Open Questions

1. **Should the `apps/studio-mockups/` app also run in CI?**
   - What we know: D-01 says move it as a "design reference"; D-07 says CI does lint/typecheck/build/test.
   - What's unclear: Whether CI should also `npm run build` the mockups, or leave them un-CI'd.
   - Recommendation: Leave mockups un-CI'd for v0 (simpler workflow, no disk pressure). Document it in the new README as "design reference, not build-verified."

2. **Where does the `apps/studio/Dockerfile` belong after the split?**
   - What we know: Currently references a nginx config mounted by a docker-compose file in zeroth-core.
   - What's unclear: Whether to fix it up (bundle standalone nginx.conf) or drop it and defer "how to deploy the studio" to a future phase.
   - Recommendation: Bundle a minimal standalone `nginx.conf` in `apps/studio/` and update the Dockerfile to `COPY nginx.conf /etc/nginx/conf.d/default.conf`. 10 lines of nginx config, permanently unblocks standalone deployment.

3. **Should the Phase 29 plan include a zeroth-core task to emit OpenAPI at build time?**
   - What we know: Today the spec is only available at runtime. Phase 32 (Reference Docs) will also need it.
   - What's unclear: Whether Phase 29 should pre-empt Phase 32 with a small `scripts/dump_openapi.py` in zeroth-core.
   - Recommendation: Yes — add a tiny `scripts/dump_openapi.py` to zeroth-core that boots the FastAPI app in-process (no uvicorn) and writes `openapi/zeroth-core-openapi.json`. It's 20 lines, benefits Phase 32, and makes zeroth-studio's snapshot regeneration trivial: `uv run python scripts/dump_openapi.py > /path/to/zeroth-studio/openapi/zeroth-core-openapi.json`. This is in D-06's "claude's discretion" space since D-06 only mandates the committed snapshot location, not how it's produced.

4. **Does zeroth-core need a symmetric CI check that `apps/studio` and `tests/studio` stay deleted?**
   - Recommendation: Out of scope for Phase 29. A `.github/workflows/ci.yml` guard ("fail if `apps/studio/` reappears") is cute but not required by any STUDIO-* requirement. Defer.

## Sources

### Primary (HIGH confidence)
- `apps/studio/package.json` — all version pins
- `apps/studio/package-lock.json` — confirmed npm, lockfileVersion 3
- `apps/studio/Dockerfile` — Node 22 alpine, nginx target
- `apps/studio/vite.config.ts` — Vite 6 config
- `apps/studio/tsconfig.json` — strict TS + `noUnusedLocals`
- `apps/studio/src/api/client.ts` — hardcoded base URL (current state)
- `tests/test_studio_api.py` — server-side studio test, confirmed to import `zeroth.core.*`
- `.github/workflows/ci.yml` — existing zeroth-core CI (for style consistency)
- `src/zeroth/core/service/app.py` line 272 — existing OpenAPI operationId pitfall comment
- `npm view openapi-typescript version` → 7.13.0 (published 2026-02-11)
- `git filter-repo --version` → a40bce548d2c (verified installed)
- `gh --version` → 2.88.0 (verified installed)
- `node --version` → v25.2.1 (local dev only; CI uses Node 22)
- [git-filter-repo docs](https://github.com/newren/git-filter-repo/blob/main/Documentation/git-filter-repo.txt) — fresh clone requirement, origin removal behavior
- [git-filter-branch manpage deprecation notice](https://git-scm.com/docs/git-filter-branch)
- [gh repo create manual](https://cli.github.com/manual/gh_repo_create)
- [Vite env handling](https://vite.dev/guide/env-and-mode.html) — `import.meta.env.VITE_*` pattern
- [openapi-typescript docs](https://openapi-ts.dev/introduction) — v7 output format

### Secondary (MEDIUM confidence)
- `.planning/phases/22-canvas-foundation-dev-infrastructure/22-VERIFICATION.md` — historical reference for original studio CI patterns (not re-verified in this research)

### Tertiary (LOW confidence)
- None — everything actionable in this research is HIGH or MEDIUM.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — every version pin read directly from package.json / lockfile
- Architecture: HIGH — filter-repo procedure verified against tool docs + local install
- Pitfalls: HIGH — five out of six pitfalls verified in-repo; Pitfall 6 flagged as "test locally before committing"
- Runtime state: HIGH — exhaustive grep + filesystem scan
- CI template: MEDIUM — template is standard GHA but not executed end-to-end in this research; planner should dry-run it

**Research date:** 2026-04-11
**Valid until:** 2026-05-11 (30 days; stable tooling, no fast-moving dependencies)
