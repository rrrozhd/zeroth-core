# Stack Research

**Domain:** Visual workflow editor frontend (Zeroth Studio) + graph authoring API additions to existing Python/FastAPI backend
**Researched:** 2026-04-09
**Confidence:** HIGH (versions verified via npm/official sources; integration with existing FastAPI backend cross-checked)

---

## Existing Backend Stack (Do Not Duplicate)

Already present and validated through v1.1 -- no changes needed for Studio:

| Technology | Version | Role |
|------------|---------|------|
| Python | >=3.12 | Backend language |
| FastAPI | >=0.115 | REST API + WebSocket support (built-in) |
| Pydantic | >=2.10 | Validation, settings, API schemas |
| SQLAlchemy | >=2.0.49 | ORM / query builder |
| asyncpg | >=0.31.0 | Async Postgres driver |
| Redis | >=5.0.0 | Distributed state, pub/sub for WS fan-out |
| Uvicorn | >=0.30 | ASGI server (HTTP + WebSocket) |

---

## Frontend Stack -- New Additions

### Core Framework (Already Decided)

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Vue 3 | 3.5.x | UI framework | Composition API, excellent TypeScript support, same ecosystem as n8n reference design. Vue 3.5+ required for VueUse 14.x compatibility |
| Vite | 8.x | Build tool + dev server | Current major version. Uses Rolldown/Oxc for faster builds. First-class Vue plugin via `@vitejs/plugin-vue`. HMR for instant feedback during canvas development |
| Pinia | 3.0.x | State management | Official Vue state library. v3 drops deprecated APIs, requires Vue 3 + TypeScript 5. Stores for workflow graph state, selection state, inspector state, canvas viewport |
| TypeScript | 5.x | Type safety | Required by Pinia 3. Type-safe node definitions, edge contracts, and API layer reduce runtime errors in graph editor logic |

**Confidence:** HIGH (versions verified via npm April 2026).

---

### Graph Editor Core (Already Decided)

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `@vue-flow/core` | 1.48.x | Interactive flow canvas | Purpose-built Vue 3 flowchart library. Handles pan/zoom, node drag, edge drawing, minimap, controls. MIT licensed. Same library n8n uses. 95 dependents on npm -- active ecosystem |
| `@dagrejs/dagre` | 3.0.0 | Automatic graph layout | Directed acyclic graph layout algorithm. Use for auto-layout button and initial workflow rendering. v3.0.0 is the actively maintained fork (released March 2026). Do NOT use the legacy `dagre` package |
| `codemirror` | 6.0.x | Code/config editing | Modular editor for JSON config, Python snippets in node inspector. Use `@codemirror/lang-json`, `@codemirror/lang-python` for language support. Active maintenance (last updated April 2026) |

**Confidence:** HIGH (versions verified via npm).

**Note on vue-codemirror wrappers:** Skip `vue-codemirror` and `vue-codemirror6` wrapper libraries. CodeMirror 6's `EditorView` API is straightforward to wrap in a Vue composable (`useCodeMirror`) with `onMounted`/`onUnmounted` lifecycle hooks. Avoids a dependency that adds abstraction without meaningful value. The wrapper is ~30 lines of code.

---

### Component Library: Reka UI + Tailwind CSS 4

**Recommendation: Reka UI (headless) + Tailwind CSS 4 (styling)**

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `reka-ui` | 2.9.x | Headless accessible UI primitives | 40+ unstyled components (Dialog, Dropdown, Tabs, Tooltip, Popover, Select). WAI-ARIA compliant out of the box. Headless means full visual control -- critical for a custom design system like a graph editor. 590K weekly downloads, actively maintained. Formerly Radix Vue |
| `tailwindcss` | 4.x | Utility-first CSS | v4 has first-party Vite plugin (`@tailwindcss/vite`) -- no PostCSS config needed. 5x faster full builds, 100x faster incremental. Pairs naturally with headless components since you control all styling |
| `@tailwindcss/vite` | 4.x | Vite integration | Direct Vite plugin integration, no postcss.config.js needed |

**Why NOT Element Plus:**
- Opinionated design system (Material-ish) clashes with building a custom graph editor UI. You fight the component styles constantly
- Heavier bundle (~300KB gzip for full import). Tree-shaking helps but still larger than headless
- Less flexible for the unique UI patterns a workflow editor needs (split panes, custom panels, floating toolbars)

**Why NOT Naive UI:**
- Better than Element Plus for customization but still styled -- you're paying for CSS you'll override
- Smaller community than Element Plus, smaller than Reka UI's weekly downloads
- TypeScript support is good but Reka UI + Tailwind gives equivalent DX with less bundle weight

**Why headless (Reka UI) wins for graph editors:**
- Workflow editors need custom chrome: floating panels, contextual toolbars, minimap overlays, node palettes
- Pre-styled components fight your layout. Headless components give you behavior (focus trap, keyboard nav, ARIA) without visual opinions
- n8n built their own design system (`@n8n/design-system`) for exactly this reason -- pre-built UI kits don't fit workflow editors well

**Confidence:** HIGH (Reka UI version verified; Tailwind v4 Vite integration verified via official docs).

---

### WebSocket: Native FastAPI WebSocket + VueUse `useWebSocket`

**Recommendation: No additional WebSocket library needed.**

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| FastAPI WebSocket (backend) | built-in | Server-side WS endpoint | FastAPI has native WebSocket support via Starlette. No library needed. `@app.websocket("/ws/workflow/{id}")` handles connection lifecycle. Use existing Redis pub/sub for multi-worker fan-out (already in stack) |
| `@vueuse/core` `useWebSocket` (frontend) | 14.2.x | Reactive WS client with auto-reconnect | VueUse's `useWebSocket` composable provides reactive `data`, `status`, `send()` with built-in `autoReconnect`, `heartbeat`, and typed message handling. No separate reconnecting-websocket library needed |

**Why NOT reconnecting-websocket:** Last published 6 years ago (v4.4.0). VueUse's `useWebSocket` provides the same auto-reconnect capability with Vue-native reactivity and active maintenance.

**Why NOT Socket.IO:** Adds a custom protocol layer and requires a Socket.IO server (not native WebSocket). FastAPI's native WebSocket + VueUse covers all needs without the overhead. Socket.IO's room/namespace features are overkill -- Zeroth's workflow-scoped channels map directly to WebSocket URL paths.

**WebSocket architecture for Studio:**

```
Browser                    FastAPI                      Redis
  |                          |                           |
  |-- WS /ws/workflow/123 -->|                           |
  |                          |-- SUBSCRIBE workflow:123 ->|
  |                          |                           |
  |<-- graph delta ---------|<-- PUBLISH workflow:123 ---|
  |                          |                           |
```

- One WS connection per open workflow
- JSON messages with `{type: "node_moved"|"edge_added"|"config_changed", payload: {...}}`
- Backend validates mutations against governance rules before broadcasting
- Redis pub/sub enables multi-worker scaling (already in stack from v1.1)

**Confidence:** HIGH (FastAPI WebSocket is Starlette built-in; VueUse version verified via npm).

---

### HTTP Client: ky

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `ky` | latest | HTTP client for REST API calls | ~2KB gzip. Wraps native Fetch API with retry, timeout, hooks (interceptors), and JSON shortcuts. TypeScript-first. Lighter than Axios (no XMLHttpRequest polyfill). Perfect for a modern Vite/Vue 3 app that only targets modern browsers |

**Why NOT Axios:** Axios is 13KB+ gzip, uses XMLHttpRequest under the hood (legacy API), and carries polyfill weight. `ky` provides the same DX (interceptors, retry, JSON handling) at ~15% of the bundle size using native Fetch.

**Why NOT ofetch:** Poor TypeScript support and lacks interceptor/plugin system. Not suitable for an API layer that needs auth token injection and error normalization.

**API layer pattern:**

```typescript
// src/api/client.ts
import ky from 'ky'

export const api = ky.create({
  prefixUrl: '/api/v1',
  hooks: {
    beforeRequest: [(req) => {
      req.headers.set('Authorization', `Bearer ${useAuthStore().token}`)
    }],
    afterResponse: [async (_req, _opts, res) => {
      if (res.status === 401) useAuthStore().logout()
    }]
  }
})

// src/api/workflows.ts
export const workflowApi = {
  list: () => api.get('workflows').json<Workflow[]>(),
  get: (id: string) => api.get(`workflows/${id}`).json<Workflow>(),
  create: (data: CreateWorkflow) => api.post('workflows', { json: data }).json<Workflow>(),
}
```

**Confidence:** MEDIUM (ky is well-established but version not pinned -- check npm for latest at install time).

---

### Utility Library

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `@vueuse/core` | 14.2.x | Vue composition utilities | Essential composables: `useWebSocket` (WS client), `useDebounceFn` (canvas events), `useResizeObserver` (panel resizing), `useLocalStorage` (UI preferences), `useDraggable` (node palette DnD), `useEventListener` (keyboard shortcuts). Requires Vue 3.5+ |

**Confidence:** HIGH (version verified via npm).

---

### Testing Stack

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `vitest` | 4.x | Unit + component tests | Vite-native test runner. Zero-config with Vite projects. Same transform pipeline as dev server. Supports Vue SFC testing via `@vitejs/plugin-vue` |
| `@vue/test-utils` | latest | Vue component mounting | Official Vue test library. `mount()`, `shallowMount()`, props/emits assertions. Required for testing Vue Flow wrapper components and inspector panels |
| `@vitest/browser` + `playwright` | 4.x / 1.59.x | Browser-mode component tests | For canvas/graph tests that need real DOM (Vue Flow relies on getBoundingClientRect, ResizeObserver). JSDOM can't simulate these. Use Vitest browser mode with Playwright provider for the ~30% of tests that need real rendering |
| `@playwright/test` | 1.59.x | E2E tests | Full browser E2E for critical workflows: create workflow, add node, draw edge, save. Separate from Vitest -- runs against dev server. Use for smoke tests, not exhaustive coverage |

**Testing strategy:**
- **70% Vitest + JSDOM:** Store logic, composables, API layer, utility functions, simple component rendering
- **20% Vitest browser mode (Playwright):** Vue Flow canvas interactions, drag-and-drop, resize behavior
- **10% Playwright E2E:** Critical user journeys (create workflow end-to-end, deploy workflow)

**Why Vitest over Jest:** Vitest shares Vite's transform pipeline. Jest requires separate Babel/TypeScript config. With a Vite project, Vitest is zero-config and 2-5x faster for Vue SFCs.

**Confidence:** HIGH (Vitest 4.x, Playwright 1.59.x verified via npm).

---

### Additional Critical Libraries

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `@vueuse/core` | 14.2.x | See Utility Library section above | |
| `vue-router` | 4.x | Client-side routing | Studio needs routes: `/workflows`, `/workflows/:id/edit`, `/settings`, `/environments`. Official Vue router |
| `splitpanes` | 3.x | Resizable panel layout | MIT licensed. Vue 3 compatible. For the three-pane Studio layout (workflow rail \| canvas \| inspector). Simpler than building custom resize handles. Used in VS Code-style layouts |
| `@iconify/vue` | 4.x | Icon system | Unified icon API accessing 200K+ icons from 150+ icon sets. Load only used icons (tree-shakeable). Better than bundling an entire icon font. Works with Tailwind classes |
| `nanoid` | 5.x | Client-side ID generation | URL-safe unique IDs for nodes/edges created in the browser before server persistence. 130 bytes gzip. Cryptographically strong. Used for optimistic UI -- node gets a temp ID immediately, server confirms or replaces |

**Confidence:** MEDIUM (versions estimated from latest npm; verify at install time).

---

## Installation

```bash
# Initialize Vue 3 project
npm create vite@latest studio -- --template vue-ts
cd studio

# Core framework (already in template)
npm install vue@3.5 pinia@3 vue-router@4

# Graph editor
npm install @vue-flow/core @dagrejs/dagre

# UI components + styling
npm install reka-ui
npm install -D tailwindcss @tailwindcss/vite

# Code editor
npm install codemirror @codemirror/lang-json @codemirror/lang-python @codemirror/theme-one-dark

# HTTP + WebSocket (via VueUse)
npm install ky @vueuse/core

# Layout + icons + IDs
npm install splitpanes @iconify/vue nanoid

# Testing
npm install -D vitest @vue/test-utils @vitest/browser @playwright/test happy-dom
npx playwright install chromium
```

---

## Alternatives Considered

| Recommended | Alternative | Why Not |
|-------------|-------------|---------|
| Reka UI (headless) | Element Plus | Opinionated styling fights custom graph editor UI. Heavier bundle. Not suitable for floating panels and canvas overlays |
| Reka UI (headless) | Naive UI | Still styled -- you override CSS constantly. Smaller ecosystem than Reka UI (by weekly downloads) |
| Reka UI (headless) | Vuetify 3 | Material Design opinions even stronger than Element Plus. Massive bundle. Wrong aesthetic for a dev-tool/IDE-style UI |
| Tailwind CSS 4 | UnoCSS | UnoCSS is faster but Tailwind 4 closed the performance gap significantly. Tailwind has broader ecosystem (plugins, examples, component libraries). Reka UI docs assume Tailwind |
| `ky` | Axios | 6x larger bundle, XMLHttpRequest-based. No advantage for modern-browser-only SPA |
| `ky` | `ofetch` | Weak TypeScript support, no interceptors. Not suitable for auth-injecting API layer |
| VueUse `useWebSocket` | Socket.IO | Custom protocol overhead, requires server-side Socket.IO. FastAPI native WS + VueUse covers all needs |
| VueUse `useWebSocket` | `reconnecting-websocket` | Unmaintained (6 years). VueUse provides same auto-reconnect with Vue reactivity |
| Vitest browser mode | Cypress component testing | Cypress is slower, requires separate runner, and component testing is still experimental. Vitest browser mode is first-class |
| `splitpanes` | Custom CSS resize | Keyboard accessibility, min/max constraints, and persistent sizes are non-trivial. splitpanes handles all of these |
| `@iconify/vue` | Font Awesome | FA requires loading entire icon font. Iconify loads individual SVGs on demand |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `vue-codemirror` / `vue-codemirror6` wrappers | Unnecessary abstraction over CodeMirror 6's already-clean API. Adds a dependency for ~30 lines of composable code | Write a `useCodeMirror` composable directly |
| Socket.IO | Adds custom protocol + server dependency. FastAPI's native WebSocket is sufficient | FastAPI WebSocket + VueUse `useWebSocket` |
| Vuex | Deprecated in favor of Pinia for Vue 3. Pinia 3 is the official recommendation | Pinia 3 |
| Element Plus / Vuetify / Naive UI | Pre-styled component libraries fight a custom graph editor's visual requirements | Reka UI (headless) + Tailwind CSS 4 |
| `dagre` (legacy package) | Unmaintained. Only `@dagrejs/dagre` receives updates (v3.0.0 March 2026) | `@dagrejs/dagre` |
| `reconnecting-websocket` | Unmaintained (last release 2020). Security risk | VueUse `useWebSocket` with `autoReconnect: true` |
| Lodash (full) | 70KB+ for utilities VueUse and native JS already provide (debounce, throttle, cloneDeep via structuredClone) | VueUse composables + native JS |
| Server-Sent Events (SSE) for canvas sync | SSE is server-to-client only. Canvas editing requires bidirectional communication (client sends mutations, server broadcasts) | WebSocket (bidirectional) |
| GraphQL | Adds schema layer + codegen complexity. Zeroth's REST API is well-defined with OpenAPI. Graph queries don't justify the tooling overhead for this domain | REST + OpenAPI + `ky` |
| Electron / Tauri | Desktop wrapper adds build complexity. Web-only is an explicit project constraint | Browser-only SPA |

---

## Backend Additions for Studio API

No new Python packages needed. All capabilities exist in the current stack:

| Capability | Existing Technology | Notes |
|------------|-------------------|-------|
| REST endpoints for graph CRUD | FastAPI + Pydantic | New routers under `/api/v1/studio/` |
| WebSocket for real-time sync | FastAPI (Starlette WebSocket) | Built-in. Add `@app.websocket()` routes |
| Multi-worker WS fan-out | Redis pub/sub | Already in stack. Use `redis.pubsub()` for cross-worker broadcast |
| CORS for dev server | FastAPI CORSMiddleware | Already configured. Add Vite dev server origin |
| Static file serving (production) | FastAPI StaticFiles or Nginx | Serve built Vue app. Nginx preferred for production |
| API schema generation | FastAPI OpenAPI | Auto-generated. Frontend can use generated types |

**Key integration point:** Generate TypeScript types from FastAPI's OpenAPI schema. Use `openapi-typescript` (npm) to auto-generate `src/api/types.ts` from `/openapi.json`. This keeps frontend types in sync with backend Pydantic models without manual duplication.

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `openapi-typescript` | latest | Generate TS types from OpenAPI spec | Dev dependency. Run as build step: `npx openapi-typescript http://localhost:8000/openapi.json -o src/api/types.ts`. Ensures type safety between Python Pydantic models and TypeScript API layer |

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| `vue@3.5.x` | `pinia@3.0.x` | Pinia 3 requires Vue 3, TypeScript 5 |
| `vue@3.5.x` | `@vueuse/core@14.x` | VueUse 14 requires Vue 3.5+ |
| `vue@3.5.x` | `@vue-flow/core@1.48.x` | Vue Flow requires Vue 3 |
| `vue@3.5.x` | `reka-ui@2.9.x` | Reka UI requires Vue 3.4+ |
| `vite@8.x` | `vitest@4.x` | Vitest 4.x aligns with Vite 8 transform pipeline |
| `vite@8.x` | `@tailwindcss/vite@4.x` | First-party Vite plugin |
| `vite@8.x` | `@vitejs/plugin-vue` | Official Vue Vite plugin, always compatible with latest Vite |
| `@vue-flow/core@1.48.x` | `@dagrejs/dagre@3.0.0` | Vue Flow's auto-layout examples use dagre. No tight coupling -- dagre computes positions, Vue Flow renders |
| `vitest@4.x` | `@vue/test-utils` | Standard pairing. Vue Test Utils is test-runner agnostic |
| `vitest@4.x` | `@vitest/browser` + `playwright@1.59.x` | Vitest browser mode uses Playwright as provider |

---

## Sources

- [npm @vue-flow/core](https://www.npmjs.com/package/@vue-flow/core) -- v1.48.2 confirmed (HIGH confidence)
- [npm @dagrejs/dagre](https://www.npmjs.com/package/@dagrejs/dagre) -- v3.0.0 confirmed (HIGH confidence)
- [npm vitest](https://www.npmjs.com/package/vitest) -- v4.1.3 confirmed (HIGH confidence)
- [npm playwright](https://www.npmjs.com/package/playwright) -- v1.59.1 confirmed (HIGH confidence)
- [npm pinia](https://www.npmjs.com/package/pinia) -- v3.0.4 confirmed (HIGH confidence)
- [npm @vueuse/core](https://www.npmjs.com/package/@vueuse/core) -- v14.2.1 confirmed (HIGH confidence)
- [Reka UI GitHub](https://github.com/unovue/reka-ui) -- v2.9.0 confirmed, headless primitives (HIGH confidence)
- [Tailwind CSS v4 announcement](https://tailwindcss.com/blog/tailwindcss-v4) -- Vite plugin, performance improvements (HIGH confidence)
- [Vite 8 announcement](https://vite.dev/blog/announcing-vite7) -- v8.0.7 current, Rolldown-based (HIGH confidence)
- [FastAPI WebSocket docs](https://fastapi.tiangolo.com/advanced/websockets/) -- native Starlette WS support (HIGH confidence)
- [VueUse useWebSocket](https://vueuse.org/core/usewebsocket/) -- auto-reconnect, reactive API (HIGH confidence)
- [Vue Testing docs](https://vuejs.org/guide/scaling-up/testing) -- Vitest + Vue Test Utils recommended (HIGH confidence)
- [n8n Frontend Architecture (DeepWiki)](https://deepwiki.com/n8n-io/n8n/6.1-server-and-api-architecture) -- design system patterns reference (MEDIUM confidence)
- [npm-compare component libraries](https://npm-compare.com/ant-design-vue,bootstrap-vue,element-plus,naive-ui,vuetify) -- download comparisons (MEDIUM confidence)
- [Axios vs ky 2026](https://www.pkgpulse.com/blog/axios-vs-ky-2026) -- bundle size and feature comparison (MEDIUM confidence)
- [CodeMirror changelog](https://codemirror.net/docs/changelog/) -- active maintenance confirmed (HIGH confidence)

---

*Stack research for: Zeroth v2.0 Studio Frontend*
*Researched: 2026-04-09*
