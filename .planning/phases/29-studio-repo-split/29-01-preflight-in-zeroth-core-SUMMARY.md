---
phase: 29-studio-repo-split
plan: 01
subsystem: platform-packaging
tags: [openapi, vite, eslint, docker, preflight]
requires:
  - A running zeroth-core dev tree with uv-installed deps (fastapi, pydantic v2)
  - Node 22 + npm 11 in apps/studio for ESLint/Vite toolchain
provides:
  - scripts/dump_openapi.py — reproducible in-process OpenAPI dumper
  - openapi/zeroth-core-openapi.json — committed OpenAPI snapshot (33 paths, incl. 3 /api/studio/v1 paths)
  - apps/studio env-driven API base URL (VITE_API_BASE_URL)
  - apps/studio ESLint flat config + split typecheck/build scripts
  - apps/studio self-contained Dockerfile (bundles nginx.conf)
affects:
  - src/zeroth/core/service/health.py — hoisted Request import to module level
  - apps/studio/** — formatting normalization across 16 .vue files via eslint --fix
tech-stack:
  added:
    - eslint@^9
    - eslint-plugin-vue@^9
    - "@vue/eslint-config-typescript@^14"
    - "@eslint/js@^9"
  patterns:
    - Reproducible OpenAPI snapshot via stub-bootstrap create_app (no DB)
    - Vite env-driven API base URL (import.meta.env.VITE_API_BASE_URL)
    - Flat ESLint config for Vue 3 + TypeScript
key-files:
  created:
    - scripts/dump_openapi.py
    - openapi/zeroth-core-openapi.json
    - apps/studio/eslint.config.js
    - apps/studio/nginx.conf
    - apps/studio/.env.example
  modified:
    - apps/studio/package.json
    - apps/studio/package-lock.json
    - apps/studio/Dockerfile
    - apps/studio/.gitignore
    - apps/studio/src/api/client.ts
    - apps/studio/src/env.d.ts
    - src/zeroth/core/service/health.py
key-decisions:
  - Use a SimpleNamespace stub bootstrap inside scripts/dump_openapi.py instead of booting the full FastAPI wiring (avoids needing an Alembic-migrated SQLite/Postgres at spec-generation time; routes are registered statically so OpenAPI schema walk works)
  - Allow underscore-prefixed unused vars in ESLint (argsIgnorePattern ^_) to preserve the existing `catch (_e)` convention in workflow.ts/useWorkflowPersistence.ts
  - Run eslint --fix across all apps/studio components in this preflight so the new repo starts with a clean lint baseline (16 .vue files reformatted for Vue attribute ordering and spacing)
requirements-completed:
  - STUDIO-02
  - STUDIO-03
  - STUDIO-05
duration: "~15 min"
completed: 2026-04-11
---

# Phase 29 Plan 01: Preflight in zeroth-core Summary

One-liner: Staged all apps/studio-side changes that must survive git filter-repo (env-driven API client, ESLint flat config, split typecheck/build, self-contained Dockerfile) plus added a reusable in-process OpenAPI dumper and committed the first snapshot used by zeroth-studio for type generation.

## Tasks Completed

| # | Task | Commit |
|---|------|--------|
| 1 | Add scripts/dump_openapi.py and commit first OpenAPI snapshot | 0dd8abd |
| 2 | Wire VITE_API_BASE_URL through apps/studio (client.ts, env.d.ts, .env.example, .gitignore) | fc44884 |
| 3 | ESLint flat config, split typecheck/build, bundle nginx.conf, update Dockerfile | 99422b3 |

Task count: 3. File count: 5 created + 7 modified = 12.

## scripts/dump_openapi.py — entrypoint choice

The initial attempt used `zeroth.core.service.entrypoint.app_factory`, which calls `bootstrap_service()` and in turn issues `SELECT … FROM deployment_versions` against a fresh SQLite DB — which fails with `no such table: deployment_versions` because Alembic hasn't run.

Final approach: import `zeroth.core.service.app.create_app` directly and pass a `types.SimpleNamespace` with all bootstrap attributes set to `None`. `create_app` only touches `bootstrap.authenticator` at request-time inside the middleware, and `bootstrap.regulus_client` at startup (gracefully skipped when `None`). Route registration (`register_*_routes(v1_router)`) is bootstrap-independent, so FastAPI's `app.openapi()` call walks the statically-registered routes and produces a complete spec with no DB, no secrets, and no uvicorn.

This makes the dumper safe to run in CI (`uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json`) with zero extra setup — the reproducible-offline requirement from D-06.

## openapi/zeroth-core-openapi.json — snapshot details

- Commit: 0dd8abd
- Size: 109 KB, ~4122 lines
- Total paths: 33
- `/api/studio/v1` paths: 3 (`/node-types`, `/workflows`, `/workflows/{workflow_id}`)
- Duplicate operationIds: 0 (compat routes are `include_in_schema=False`, per D-06 / Pitfall 3)
- Valid JSON per `python -m json.tool`

## ESLint config decisions

- Flat config at `apps/studio/eslint.config.js`.
- Presets: `@eslint/js` recommended → `eslint-plugin-vue` flat/recommended → `@vue/eslint-config-typescript` (TS parsing for `<script setup lang="ts">`).
- Ignores: `dist/**`, `node_modules/**`, `src/api/types.gen.ts`, `src/api/schema.d.ts`, `**/*.d.ts` (generated + declaration files should never lint).
- Globals: `window`, `document`, `fetch`, `console`, `RequestInit` (avoids `no-undef` spam without pulling the full browser env preset).
- Rules:
  - `vue/multi-word-component-names`: off (we have single-word component names like `NodeInspector` used as `<NodeInspector>` — the rule is a stylistic preference that fights every single component).
  - `@typescript-eslint/no-unused-vars`: allow `^_` prefix for args, vars, and caught errors (preserves the `catch (_e)` convention).

Lint baseline was normalized via one-time `eslint --fix` across 16 Vue components (attribute ordering, single-line element content, `html-closing-bracket-spacing`). After the fix: `npm run lint` exits 0 with `--max-warnings=0`.

## New devDependencies in apps/studio/package.json

| Package | Version |
|---------|---------|
| eslint | ^9.39.4 |
| eslint-plugin-vue | ^9.33.0 |
| @vue/eslint-config-typescript | ^14.7.0 |
| @eslint/js | ^9.39.4 |

Installed via `npm install --save-dev` (added 133 transitive packages, 0 vulnerabilities).

## package.json scripts — before / after

Before:
```json
{
  "dev": "vite",
  "build": "vue-tsc -b && vite build",
  "preview": "vite preview",
  "generate-types": "openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts",
  "test": "vitest run",
  "test:watch": "vitest"
}
```

After:
```json
{
  "dev": "vite",
  "typecheck": "vue-tsc -b --noEmit",
  "build": "vite build",
  "preview": "vite preview",
  "lint": "eslint . --max-warnings=0",
  "generate:api": "openapi-typescript openapi/zeroth-core-openapi.json -o src/api/types.gen.ts",
  "test": "vitest run",
  "test:watch": "vitest"
}
```

Notes:
- `build` no longer runs vue-tsc; CI will run lint → typecheck → build as three gates.
- `generate:api` now points at the repo-root `openapi/zeroth-core-openapi.json` (the file lives at `apps/studio/../openapi/` in zeroth-core today, and will live at `openapi/` in zeroth-studio after filter-repo). At preflight time the path does not yet resolve from inside `apps/studio/`, but that's fine — the script is only exercised in zeroth-studio CI.

## Dockerfile before / after

Before:
```dockerfile
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html/studio
# Nginx config is mounted via docker-compose volume (D-11)
EXPOSE 80
```

After:
```dockerfile
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html/studio
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

The new `apps/studio/nginx.conf` provides a 10-line standalone server block that serves the built `dist/` under both `/` and `/studio/` with SPA fallback to `index.html`. This resolves Q2 (the old Dockerfile depended on a docker-compose volume mount that no longer applies once zeroth-studio is a standalone repo).

## Verification Results

- `uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json` → exits 0, writes valid JSON
- `python -m json.tool openapi/zeroth-core-openapi.json` → clean
- `grep -q "import.meta.env.VITE_API_BASE_URL" apps/studio/src/api/client.ts` → match
- `grep -q "VITE_API_BASE_URL" apps/studio/src/env.d.ts` → match
- `grep -q ".env.local" apps/studio/.gitignore` → match
- `test -f apps/studio/eslint.config.js` → present
- `test -f apps/studio/nginx.conf` → present
- `grep -q "COPY nginx.conf" apps/studio/Dockerfile` → match
- `grep -q '"typecheck"' apps/studio/package.json` → match
- `grep -q '"lint"' apps/studio/package.json` → match
- `grep -q '"generate:api"' apps/studio/package.json` → match
- `cd apps/studio && npm run lint` → 0 problems
- `cd apps/studio && npm run typecheck` → exit 0
- `cd apps/studio && npm run build` → 97 modules transformed, dist emitted (366 KB JS, 24 KB CSS)
- `uv run pytest -v tests/test_studio_api.py` → 10 passed
- `uv run pytest -v tests/ -k health` → 25 passed (health.py refactor did not regress)

## Deviations from Plan

### [Rule 3 - Blocking] Hoist `Request` import in src/zeroth/core/service/health.py
- **Found during:** Task 1, first run of `scripts/dump_openapi.py`
- **Issue:** `health_ready(request: Request)` is defined inside `register_health_routes`, which imports `Request` locally inside the function body. With `from __future__ import annotations` at module top, all annotations are strings and FastAPI's schema walker calls `typing.get_type_hints(endpoint)`, which resolves names against the endpoint's `__globals__` — not its enclosing function scope. Result: `name 'Request' is not defined` → Pydantic raises `PydanticUserError: TypeAdapter[Annotated[ForwardRef('Request'), …]] is not fully defined`, blocking `app.openapi()`.
- **Fix:** Added `from fastapi import Request` at module level and removed the local import inside `register_health_routes`.
- **Files modified:** src/zeroth/core/service/health.py
- **Verification:** `uv run pytest tests/ -k health` → 25 passed; OpenAPI dump succeeds.
- **Commit:** 0dd8abd

### [Rule 1 - Bug] Normalize Vue component formatting via `eslint --fix`
- **Found during:** Task 3, `npm run lint` reported 386 warnings + 4 errors with `--max-warnings=0`.
- **Issue:** Pre-existing Vue components were never linted — they used inconsistent attribute ordering, single-line element content, and missing spaces before self-closing tags. Four `catch (_e)` blocks also tripped `no-unused-vars` because the rule didn't recognize `^_`-prefix convention.
- **Fix:** Ran `npx eslint . --fix` (autofixed 386 formatting warnings across 16 .vue files) and added `argsIgnorePattern: '^_'`, `varsIgnorePattern: '^_'`, `caughtErrorsIgnorePattern: '^_'` to `@typescript-eslint/no-unused-vars` in `eslint.config.js`.
- **Files modified:** apps/studio/eslint.config.js + 16 .vue component files (CanvasControls, StudioCanvas, InspectorField, NodeInspector, AgentNode, ApprovalGateNode, BaseNode, ConditionBranchNode, DataMappingNode, EndNode, ExecutionUnitNode, MemoryResourceNode, StartNode, NodePalette, PaletteCategory, AppHeader, WorkflowRail)
- **Verification:** `npm run lint` → 0 problems; `npm run typecheck && npm run build` still pass.
- **Commit:** 99422b3

**Total deviations:** 2 auto-fixed (1 Rule 3 blocking bug, 1 Rule 1 pre-existing formatting debt). **Impact:** low — health.py fix is a correctness improvement that unblocks OpenAPI generation permanently; the Vue autofix establishes a clean lint baseline for the zeroth-studio repo that filter-repo carries forward.

## Authentication Gates

None — this plan made no network calls or cloud-provider calls.

## Issues Encountered

None — all three tasks verified on first try after fixing the two deviations inline.

## Next Phase Readiness

Ready for **29-02-filter-repo-extract-and-create-remote**. The git history at `main` now contains all apps/studio/ changes that must survive extraction, plus the OpenAPI snapshot and dumper that Phase 32 / Wave 3 will consume.

## Self-Check: PASSED

- scripts/dump_openapi.py: FOUND
- openapi/zeroth-core-openapi.json: FOUND (109 KB)
- apps/studio/eslint.config.js: FOUND
- apps/studio/nginx.conf: FOUND
- apps/studio/.env.example: FOUND
- Commit 0dd8abd: FOUND in git log
- Commit fc44884: FOUND in git log
- Commit 99422b3: FOUND in git log
