---
phase: 29-studio-repo-split
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - scripts/dump_openapi.py
  - openapi/zeroth-core-openapi.json
  - apps/studio/package.json
  - apps/studio/eslint.config.js
  - apps/studio/src/api/client.ts
  - apps/studio/src/env.d.ts
  - apps/studio/.env.example
  - apps/studio/.gitignore
  - apps/studio/nginx.conf
  - apps/studio/Dockerfile
autonomous: true
requirements:
  - STUDIO-02
  - STUDIO-03
  - STUDIO-05
user_setup: []

must_haves:
  truths:
    - "scripts/dump_openapi.py produces openapi/zeroth-core-openapi.json reproducibly"
    - "apps/studio/src/api/client.ts reads VITE_API_BASE_URL from import.meta.env"
    - "apps/studio has a working ESLint flat config with Vue + TS presets"
    - "npm run typecheck and npm run build are separate scripts in apps/studio/package.json"
    - "apps/studio/Dockerfile is self-contained (bundles nginx.conf, no external mount required)"
  artifacts:
    - path: "scripts/dump_openapi.py"
      provides: "In-process OpenAPI JSON dumper for zeroth-core"
      exports: ["main"]
    - path: "openapi/zeroth-core-openapi.json"
      provides: "Committed OpenAPI snapshot used by zeroth-studio type generation"
      contains: "openapi"
    - path: "apps/studio/eslint.config.js"
      provides: "ESLint flat config for Vue 3 + TypeScript"
      min_lines: 15
    - path: "apps/studio/.env.example"
      provides: "Default VITE_API_BASE_URL"
      contains: "VITE_API_BASE_URL"
    - path: "apps/studio/nginx.conf"
      provides: "Standalone nginx config for Studio Docker image"
      min_lines: 8
    - path: "apps/studio/src/api/client.ts"
      provides: "Env-driven API client base URL"
      contains: "import.meta.env.VITE_API_BASE_URL"
  key_links:
    - from: "apps/studio/src/api/client.ts"
      to: "import.meta.env.VITE_API_BASE_URL"
      via: "Vite env substitution"
      pattern: "import\\.meta\\.env\\.VITE_API_BASE_URL"
    - from: "apps/studio/Dockerfile"
      to: "apps/studio/nginx.conf"
      via: "COPY into /etc/nginx/conf.d/default.conf"
      pattern: "COPY nginx\\.conf"
    - from: "scripts/dump_openapi.py"
      to: "zeroth.core.service.app"
      via: "in-process FastAPI import + app.openapi()"
      pattern: "from zeroth\\.core\\.service\\.app"
---

<objective>
Wave 1 preflight in zeroth-core: stage every change that must survive the git filter-repo extraction. This plan modifies apps/studio/ in place inside zeroth-core (same working tree you read) so the changes become part of the history that filter-repo carries into zeroth-studio. Also adds a reusable OpenAPI dumper and commits the first snapshot — unblocking Phase 32 and giving Wave 3 a file to consume.

Purpose: Per 29-RESEARCH Wave 0 gaps and resolved Q2/Q3, apps/studio must already have ESLint config, env-driven API client, split typecheck/build scripts, and a self-contained Dockerfile BEFORE filter-repo runs — otherwise they'd need to be redone in the new repo with no history.

Output: Five code changes in apps/studio/ + a new scripts/dump_openapi.py + a committed openapi/zeroth-core-openapi.json at the zeroth-core repo root.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/29-studio-repo-split/29-CONTEXT.md
@.planning/phases/29-studio-repo-split/29-RESEARCH.md
@CLAUDE.md
@apps/studio/package.json
@apps/studio/Dockerfile
@apps/studio/src/api/client.ts
@apps/studio/src/env.d.ts

<interfaces>
<!-- Current client.ts (to be modified): -->
```typescript
const BASE_URL = '/api/studio/v1'
// ...apiFetch<T>(path, options)
```

<!-- Current env.d.ts (to be augmented): -->
```typescript
/// <reference types="vite/client" />
declare module '*.vue' { ... }
```

<!-- Current package.json scripts (to be split): -->
```json
"build": "vue-tsc -b && vite build",
"generate-types": "openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts",
```

<!-- zeroth-core FastAPI app to introspect: -->
From src/zeroth/core/service/app.py (existing):
```python
# app = FastAPI(...) is constructed inside a factory — check app.py for the
# actual entrypoint. If bootstrap requires env/DB, use the factory not a bare import.
```
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add scripts/dump_openapi.py and commit the first OpenAPI snapshot</name>
  <files>scripts/dump_openapi.py, openapi/zeroth-core-openapi.json</files>
  <action>
Create `scripts/dump_openapi.py` — a tiny CLI that imports the FastAPI app in-process (no uvicorn needed) and writes the OpenAPI JSON to stdout or to a file via `--out`. Read `src/zeroth/core/service/app.py` first to determine how to obtain the FastAPI instance — there is likely a factory (e.g. `create_app()` or similar). If the factory requires settings/bootstrap (likely, given Phase 20 wiring), call the minimal bootstrap path to get a usable app; if a bare `app = FastAPI(...)` module global exists, prefer that for simplicity.

Script structure:
```python
"""Dump the zeroth-core OpenAPI spec to JSON for offline consumption (Phase 29, Phase 32)."""
import json, sys, argparse
from pathlib import Path

def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--out", type=Path, default=None, help="Output file (default: stdout)")
    args = ap.parse_args()

    # Import after argparse so --help is fast
    from zeroth.core.service.app import <app_or_factory>  # pick the right symbol
    app = <factory_call_if_needed>

    spec = app.openapi()
    text = json.dumps(spec, indent=2, sort_keys=True) + "\n"
    if args.out:
        args.out.parent.mkdir(parents=True, exist_ok=True)
        args.out.write_text(text)
    else:
        sys.stdout.write(text)
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

Then run it to produce the snapshot:
```bash
uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json
```

Verify the resulting file:
- Is valid JSON (`python -m json.tool openapi/zeroth-core-openapi.json > /dev/null`)
- Has a top-level `"openapi"` key and `"paths"` object
- Contains at least one `/api/studio/v1/*` path (sanity check)
- No duplicate operationIds (per Pitfall 3): `python -c "import json,collections; d=json.load(open('openapi/zeroth-core-openapi.json')); ops=[op.get('operationId') for p in d['paths'].values() for op in p.values() if isinstance(op,dict) and op.get('operationId')]; dupes=[k for k,v in collections.Counter(ops).items() if v>1]; print('DUPES:',dupes); assert not dupes"`

Commit both files. This satisfies D-06 (reproducible offline snapshot) and the resolved Q3 (zeroth-core gains the dumper so Phase 32 is unblocked too).
  </action>
  <verify>
    <automated>uv run python scripts/dump_openapi.py --out /tmp/verify-openapi.json && python -m json.tool /tmp/verify-openapi.json > /dev/null && python -c "import json; d=json.load(open('openapi/zeroth-core-openapi.json')); assert 'openapi' in d and 'paths' in d and any('/api/studio/v1' in p for p in d['paths']), 'spec malformed'"</automated>
  </verify>
  <done>scripts/dump_openapi.py exists, is executable via uv run, openapi/zeroth-core-openapi.json is committed, valid JSON, contains studio paths, no duplicate operationIds.</done>
</task>

<task type="auto">
  <name>Task 2: Wire VITE_API_BASE_URL through apps/studio (client.ts, env.d.ts, .env.example, .gitignore)</name>
  <files>apps/studio/src/api/client.ts, apps/studio/src/env.d.ts, apps/studio/.env.example, apps/studio/.gitignore</files>
  <action>
Per D-09 and 29-RESEARCH Code Examples, replace the hardcoded `/api/studio/v1` with a Vite env-driven value so zeroth-studio can develop against any running zeroth-core.

1. `apps/studio/src/api/client.ts`: replace line 1 with
```typescript
const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? '/api/studio/v1'
```
Leave the rest of the file (ApiError, apiFetch) untouched.

2. `apps/studio/src/env.d.ts`: extend with the ImportMetaEnv interface so TS knows about VITE_API_BASE_URL:
```typescript
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_BASE_URL?: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}

declare module '*.vue' {
  import type { DefineComponent } from 'vue'
  const component: DefineComponent<object, object, unknown>
  export default component
}
```

3. Create `apps/studio/.env.example`:
```ini
# URL of a running zeroth-core service. Include the /api/studio/v1 prefix.
# Override locally by creating .env.local (gitignored).
VITE_API_BASE_URL=http://localhost:8000/api/studio/v1
```

4. Create or update `apps/studio/.gitignore` to include `.env.local` and `.env.*.local` (in addition to any existing ignores — read first if present). If the file doesn't exist yet, include a minimal Vite gitignore:
```
node_modules
dist
.env.local
.env.*.local
*.tsbuildinfo
```

Do NOT commit a `.env.local` file. Verify by running `npm run build` inside apps/studio to confirm TS still compiles with the new ImportMetaEnv declaration (no new errors).
  </action>
  <verify>
    <automated>cd apps/studio && grep -q "import.meta.env.VITE_API_BASE_URL" src/api/client.ts && grep -q "VITE_API_BASE_URL" src/env.d.ts && test -f .env.example && grep -q "VITE_API_BASE_URL" .env.example && grep -q ".env.local" .gitignore && npx vue-tsc -b --noEmit</automated>
  </verify>
  <done>client.ts reads from import.meta.env, env.d.ts declares ImportMetaEnv, .env.example present with VITE_API_BASE_URL, .gitignore excludes .env.local, vue-tsc passes with no new errors.</done>
</task>

<task type="auto">
  <name>Task 3: Add ESLint flat config, split typecheck/build scripts, bundle standalone nginx.conf</name>
  <files>apps/studio/package.json, apps/studio/eslint.config.js, apps/studio/nginx.conf, apps/studio/Dockerfile</files>
  <action>
Three wave-1 gaps from 29-RESEARCH §Wave 0 Gaps: ESLint missing, build conflates typecheck, Dockerfile references an external nginx config.

1. **ESLint flat config** (`apps/studio/eslint.config.js`) — Vue 3 + TS preset, ignores generated + build output:
```javascript
import js from '@eslint/js'
import vue from 'eslint-plugin-vue'
import vueTsConfig from '@vue/eslint-config-typescript'

export default [
  { ignores: ['dist/**', 'node_modules/**', 'src/api/types.gen.ts', '**/*.d.ts'] },
  js.configs.recommended,
  ...vue.configs['flat/recommended'],
  ...vueTsConfig(),
  {
    languageOptions: {
      ecmaVersion: 'latest',
      sourceType: 'module',
      globals: { window: 'readonly', document: 'readonly', fetch: 'readonly', console: 'readonly' },
    },
    rules: {
      'vue/multi-word-component-names': 'off',
    },
  },
]
```

2. **package.json** — add devDependencies and split scripts. Use `npm install --save-dev` (do NOT hand-edit lockfile):
```bash
cd apps/studio
npm install --save-dev eslint@^9 eslint-plugin-vue@^9 @vue/eslint-config-typescript@^14 @eslint/js@^9
```
Then edit `apps/studio/package.json` scripts to:
```json
"scripts": {
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
- `build` no longer runs vue-tsc (typecheck is its own script now; CI runs both serially).
- `generate:api` replaces the old `generate-types` (runtime fetch is gone). The path `openapi/zeroth-core-openapi.json` is repo-root relative in zeroth-studio; at preflight time this path does not yet exist inside apps/studio — that's fine, it's only exercised in zeroth-studio CI.
- `lint` targets `.` (flat config discovers files).

3. **Standalone nginx.conf** (`apps/studio/nginx.conf`) — per resolved Q2, bundle a ~10-line standalone config so the Dockerfile stops depending on a docker-compose-mounted file from zeroth-core:
```nginx
server {
  listen       80;
  server_name  _;
  root   /usr/share/nginx/html/studio;
  index  index.html;

  location / {
    try_files $uri $uri/ /studio/index.html;
  }

  location /studio/ {
    try_files $uri $uri/ /studio/index.html;
  }
}
```

4. **Dockerfile** (`apps/studio/Dockerfile`) — add COPY of nginx.conf into `/etc/nginx/conf.d/default.conf` and drop the stale comment:
```dockerfile
# Stage 1: Build Vue app
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html/studio
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Verify locally: `npm run lint` passes, `npm run typecheck` passes, `npm run build` produces `dist/`. Do NOT run `npm run generate:api` yet — the openapi path is relative to the zeroth-studio repo root and will only exist after Wave 3.
  </action>
  <verify>
    <automated>cd apps/studio && test -f eslint.config.js && test -f nginx.conf && grep -q "COPY nginx.conf" Dockerfile && grep -q '"typecheck"' package.json && grep -q '"lint"' package.json && grep -q '"generate:api"' package.json && npm run lint && npm run typecheck && npm run build</automated>
  </verify>
  <done>ESLint flat config runs clean, typecheck + build are separate passing scripts, nginx.conf bundled, Dockerfile copies it, all three pass locally inside apps/studio.</done>
</task>

</tasks>

<verification>
After all three tasks:
- `scripts/dump_openapi.py` exists and produces `openapi/zeroth-core-openapi.json` reproducibly
- `openapi/zeroth-core-openapi.json` is committed, valid JSON, contains `/api/studio/v1` paths, no duplicate operationIds
- `apps/studio/src/api/client.ts` reads `import.meta.env.VITE_API_BASE_URL`
- `apps/studio/.env.example` and updated `.gitignore` are in place
- `apps/studio/eslint.config.js` + new devDependencies installed
- `apps/studio/package.json` has separate `typecheck`, `build`, `lint`, `generate:api` scripts
- `apps/studio/nginx.conf` exists and is referenced by the Dockerfile
- `npm run lint && npm run typecheck && npm run build` all pass inside apps/studio
- Existing zeroth-core tests unaffected: `uv run pytest -v tests/test_studio_api.py` still green
</verification>

<success_criteria>
1. Every change lands inside the current zeroth working tree so git filter-repo (Plan 02) carries them into zeroth-studio automatically.
2. No file outside `apps/studio/`, `scripts/`, or `openapi/` is modified.
3. A single commit per task is acceptable; the phase-level commit bundles them.
4. The OpenAPI snapshot is the authoritative input for Plan 03's `generate:api` drift gate.
</success_criteria>

<output>
After completion, create `.planning/phases/29-studio-repo-split/29-01-SUMMARY.md` covering:
- scripts/dump_openapi.py entrypoint (which symbol from zeroth.core.service.app was used)
- openapi/zeroth-core-openapi.json commit hash and size
- ESLint config decisions (which presets, which ignores)
- List of new devDependencies added to apps/studio/package.json
- Dockerfile before/after diff
</output>
