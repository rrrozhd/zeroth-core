# Phase 29: Studio Repo Split - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning
**Mode:** Inline discuss (autonomous workflow)

<domain>
## Phase Boundary

Extract the Vue 3 + Vue Flow Studio frontend from the `zeroth` monorepo into a new public GitHub repository `rrrozhd/zeroth-studio` with full git history preserved. After the split: zeroth-studio has its own independent CI, consumes zeroth-core only via HTTP/OpenAPI, regenerates its API types from the zeroth-core OpenAPI spec, and the studio source is removed from this repo. Both repos cross-link and a compatibility matrix is documented in zeroth-studio's README.

This phase is a structural / packaging move — no new Studio features are built. Phases 24-26 (originally cancelled) continue in the new repo after this phase.

</domain>

<decisions>
## Implementation Decisions

### D-01 Scope of files to move
- `apps/studio/` — the real Vue 3 + Vue Flow frontend (PRIMARY)
- `apps/studio-mockups/` — the mockup app, kept as design reference inside the new repo
- `tests/studio/` — Studio-specific tests, currently in the zeroth-core test tree

### D-02 History preservation method
- Use **`git filter-repo`** (modern, reliable, preserves full history of moved paths only)
- Filter on the three paths above so the new repo has zero zeroth-core history noise
- Tool installed via `pipx install git-filter-repo` if not already present

### D-03 Source cleanup in zeroth-core
- After the split lands, **delete `apps/studio/`, `apps/studio-mockups/`, and `tests/studio/`** from zeroth-core in the same phase
- Single source of truth immediately — no transitional duplication
- Update `pytest.ini` / `pyproject.toml` test discovery if needed

### D-04 Repo creation and push
- Claude creates the GitHub repo via `gh repo create rrrozhd/zeroth-studio --public --description "Zeroth Studio — Vue 3 + Vue Flow frontend for governed multi-agent workflows"`
- Push the filtered history to `main` (force-push acceptable since the repo is fresh)
- Set the default branch to `main`

### D-05 Compatibility matrix location
- Lives in **`zeroth-studio` README only** (the dependent side — where the version pin lives)
- Format: a markdown table mapping `zeroth-studio` versions ↔ `zeroth-core` versions
- Initial entry: `zeroth-studio 0.1.0 ↔ zeroth-core 0.1.1`

### D-06 OpenAPI sync wiring
- **`npm run generate:api`** script in zeroth-studio runs `openapi-typescript` against a local or pinned `zeroth-core` OpenAPI spec
- Generated types committed to the repo (`src/api/types.gen.ts` or similar)
- **CI gate:** a job runs `generate:api` and `git diff --exit-code` — fails the build if generated types drift from checked-in version
- Source of truth: a snapshot of the zeroth-core OpenAPI JSON committed at `openapi/zeroth-core-openapi.json` so generation is reproducible offline

### D-07 CI pipeline for zeroth-studio
- GitHub Actions (consistent with zeroth-core)
- Workflow `.github/workflows/ci.yml`: install → lint (ESLint) → typecheck (vue-tsc) → build (vite build) → test (vitest run) → openapi-drift-check
- Triggers: push and PR to `main`
- Node version: pinned to whatever `apps/studio/package.json` currently uses

### D-08 README cross-linking
- `zeroth-studio` README links to `https://github.com/rrrozhd/zeroth` (zeroth-core)
- `zeroth-core` README gains a "Studio" section linking to `https://github.com/rrrozhd/zeroth-studio`
- Both have a brief one-line description of what the other provides

### D-09 Local development workflow
- After clone: `npm install && npm run dev` works against any running zeroth-core instance
- Default `VITE_API_BASE_URL` from `.env.example`, overridable via `.env.local`
- Development workflow does not require zeroth-core source — only a running service (local docker, hosted, etc.)

### Claude's Discretion
- Exact CI matrix details (Node 20 vs 22, single or multi-version)
- Whether to publish zeroth-studio to npm or leave as source-only
- Whether to use pnpm/npm/yarn (default: keep whatever apps/studio currently uses)
- Initial CHANGELOG.md and LICENSE files (Apache-2.0 like zeroth-core)
- README structure and screenshots

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `apps/studio/` — Vue 3 + Vue Flow editor with package.json, vite config, Dockerfile
- `apps/studio-mockups/` — separate Vue mockup app
- `tests/studio/` — Studio-specific test suite
- All three are self-contained — no Python imports, only HTTP API calls

### Established Patterns
- zeroth-core uses GitHub Actions with trusted publisher (Phase 28 reference)
- Apache-2.0 license, keepachangelog format CHANGELOG, CONTRIBUTING.md (mirror Phase 28 conventions)
- Conventional Commits

### Integration Points
- HTTP client in `apps/studio/src/` already calls zeroth-core REST endpoints
- No source-level dependency exists today — the split is mostly mechanical
- `tests/studio/` may have setup that imports from zeroth-core test fixtures — needs scrubbing

</code_context>

<specifics>
## Specific Ideas

- The final repo URL is `https://github.com/rrrozhd/zeroth-studio`
- Use `pipx install git-filter-repo` if not present
- The compatibility matrix entry format: `| zeroth-studio | zeroth-core | Notes |`
- Pre-commit hook (optional, deferred): block commits if generated types are stale

</specifics>

<deferred>
## Deferred Ideas

- npm publishing of zeroth-studio (out of scope — source-only repo for v0)
- Per-version git tags backfilled into zeroth-studio (complex, not required by success criteria)
- Phases 24-26 work (continues in zeroth-studio after this phase ships)
- Storybook / component catalog (Phase 31+ if at all)
- E2E tests across both repos (deferred to a future integration phase)

</deferred>
