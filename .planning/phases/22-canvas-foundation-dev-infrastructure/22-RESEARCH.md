# Phase 22: Canvas Foundation & Dev Infrastructure - Research

**Researched:** 2026-04-09
**Domain:** Vue 3 interactive graph canvas, REST API, Docker/Nginx serving
**Confidence:** HIGH

## Summary

Phase 22 establishes the Zeroth Studio frontend application: a Vue 3 + Vite project with an interactive graph canvas (Vue Flow), Pinia state management, Tailwind CSS styling, a graph authoring REST API mounted on the existing FastAPI backend, and a Docker multi-container deployment with Nginx serving static files. The existing codebase provides strong foundations -- FastAPI app factory, graph models/repository, auth middleware, and Docker infrastructure are all in place and need extension rather than creation.

The primary technical challenge is integrating Vue Flow's node/edge/handle system with 8 custom node types that have typed ports, while keeping the Pinia store architecture clean for future phases (palette, inspector, undo/redo, WebSocket). The secondary challenge is the Docker infrastructure change from single-container API-only to multi-container with Nginx serving the Vue SPA and proxying API requests.

**Primary recommendation:** Use Vue Flow 1.x with custom node components per type, Pinia stores split by concern (canvas, workflow, UI), openapi-typescript for type generation, and Tailwind CSS v4 with the Vite plugin. Extend the existing Nginx container (already in docker-compose.yml) to serve built Vue assets and proxy `/api/studio/` to FastAPI.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Production Studio app lives in `apps/studio/` (new directory). Keep `apps/studio-mockups/` as reference only.
- **D-02:** Vue 3 + Vite scaffold with Vue Flow (@vue-flow/core), Pinia for state management, dagre for auto-layout, Tailwind CSS for styling.
- **D-03:** Frontend types generated from backend OpenAPI spec using `openapi-typescript` at build time (INFRA-02).
- **D-04:** Pinia stores for canvas state, workflow data, and UI state -- separate concerns across stores.
- **D-05:** Extended node type set (8 types): Agent, Execution Unit, Approval Gate, Memory Resource, Condition/Branch, Start, End, Data Mapping.
- **D-06:** Each node has typed input and output ports. Edges connect output-to-input. Port types enforce valid connections. Uses Vue Flow's handle system.
- **D-07:** Phase 22 nodes are visual placement + label only. No inline config editing -- that's Phase 23 inspector (CANV-04).
- **D-08:** Dedicated Studio API router in `src/zeroth/service/studio_api.py`, mounted on the existing FastAPI app.
- **D-09:** URL prefix: `/api/studio/v1/` -- namespaced separately from the existing `/v1/` deployment API.
- **D-10:** Authentication required from the start -- reuse existing auth middleware from `src/zeroth/service/auth.py`. RBAC enforcement is a downstream requirement (GOV-04).
- **D-11:** Multi-container Docker deployment: separate containers for Nginx (serves Vue static files, proxies API) and FastAPI (uvicorn).
- **D-12:** Development workflow: Vite dev server with HMR + FastAPI running separately. Vite proxies `/api/` requests to FastAPI backend.

### Claude's Discretion
- Exact Pinia store boundaries and naming
- Vue Flow configuration details (minimap position, zoom limits, default viewport)
- Tailwind theme configuration and color palette
- Nginx configuration specifics (caching, gzip, headers)
- openapi-typescript script integration (npm script vs build plugin)
- Test framework choice for frontend (Vitest recommended given Vite)
- Exact file/component structure within `apps/studio/src/`

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CANV-01 | User can drag nodes from palette onto a pannable, zoomable canvas | Vue Flow provides `addNodes()` API + drag-to-place. 8 custom node types with Handle system. Phase 22 uses canvas control "Add node" button (palette is Phase 23). |
| CANV-02 | User can draw edges between typed node ports to create connections | Vue Flow Handle component with `type="source"` / `type="target"`. ConnectionMode.Strict enforces source-to-target only. Port type validation via `isValidConnection` callback. |
| CANV-06 | User can save and load workflow graphs via the authoring API | Studio API wraps existing `GraphRepository.save()` / `.get()` / `.list()`. Frontend-to-backend graph model mapping via openapi-typescript generated types. |
| CANV-09 | User can navigate with pan, zoom, fit-to-view, and minimap | Vue Flow built-in: `@vue-flow/minimap` plugin, `fitView()` composable, scroll-to-zoom, click-drag pan. Canvas controls bar with zoom in/out/fit buttons. |
| CANV-10 | User can work in a responsive three-panel layout with collapsible panels | Shell layout: header + workflow rail (304px, collapsible) + canvas (flex-1) + inspector (0px/hidden Phase 22). CSS Grid or Flexbox. |
| API-01 | Studio can CRUD workflow graphs via REST endpoints | FastAPI router at `/api/studio/v1/workflows/` with Pydantic request/response models. Wraps GraphRepository. Auth middleware reused. |
| API-03 | Studio can retrieve available node type schemas | Endpoint returning the 8 node type definitions with port configurations, field schemas, and validation rules. Static or config-driven. |
| INFRA-01 | Studio frontend served via Nginx alongside FastAPI in Docker | Extend existing `docker-compose.yml` nginx service to serve `apps/studio/dist/` and proxy `/api/` to FastAPI. Multi-stage Dockerfile for Vue build. |
| INFRA-02 | Frontend types generated from backend OpenAPI spec | `openapi-typescript` generates `.d.ts` from FastAPI's `/openapi.json`. npm script: `"generate-types": "openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts"` |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| vue | 3.5.32 | UI framework | Locked decision D-02. Current latest. |
| @vue-flow/core | 1.48.2 | Interactive graph canvas | Purpose-built Vue 3 flow/node editor. MIT licensed. Same ecosystem as n8n reference. |
| @vue-flow/minimap | 1.5.4 | Minimap overlay for canvas | Official Vue Flow plugin for CANV-09 minimap. |
| @vue-flow/controls | 1.1.3 | Zoom/fit controls UI | Official Vue Flow plugin for canvas navigation controls. |
| @vue-flow/background | 1.3.2 | Dot grid background | Official Vue Flow plugin for canvas background pattern. |
| pinia | 3.0.4 | State management | Locked decision D-04. Official Vue state management. |
| @dagrejs/dagre | 3.0.0 | Automatic graph layout | Locked decision D-02. Standard DAG layout algorithm. |
| tailwindcss | 4.2.2 | Utility-first CSS | Locked decision D-02. v4 uses Vite plugin (no PostCSS config). |
| @tailwindcss/vite | latest | Tailwind Vite integration | v4 official Vite plugin. Replaces PostCSS setup. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| vite | 6.0+ | Build tool + dev server | Build and HMR. Existing mockup uses Vite 5, upgrade to 6 for new project. |
| @vitejs/plugin-vue | 6.0.5 | Vue SFC compilation for Vite | Required for .vue file support in Vite 6. |
| typescript | 6.0.2 | Type checking | Strict mode for Studio frontend. |
| vue-tsc | 3.2.6 | Vue TypeScript type-checking | Build-time type verification for .vue files. |
| openapi-typescript | 7.13.0 | Generate TS types from OpenAPI | INFRA-02: type generation from FastAPI spec. |
| vitest | 4.1.4 | Test framework | Natural pairing with Vite. Fast, same config. |
| @testing-library/vue | latest | Component testing utilities | DOM-based Vue component tests with Vitest. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| @vue-flow/core | cytoscape.js | Cytoscape is more general-purpose but heavier; Vue Flow is Vue-native with better DX. Decision locked. |
| Tailwind CSS v4 | Tailwind v3 | v4 is simpler setup (Vite plugin, no PostCSS). v3 is more documented but v4 is stable since Jan 2025. Use v4. |
| openapi-typescript | hey-api/openapi-ts | hey-api generates full client code; openapi-typescript generates types only (lighter, composable with fetch). Types-only is sufficient. |
| Vitest | Jest | Vitest shares Vite config, faster for Vite projects. Jest requires separate transform config. Use Vitest. |

**Installation:**
```bash
cd apps/studio
npm create vite@latest . -- --template vue-ts
npm install vue@^3.5 @vue-flow/core @vue-flow/minimap @vue-flow/controls @vue-flow/background pinia @dagrejs/dagre tailwindcss @tailwindcss/vite
npm install -D typescript vue-tsc @vitejs/plugin-vue openapi-typescript vitest @testing-library/vue
```

## Architecture Patterns

### Recommended Project Structure
```
apps/studio/
├── index.html
├── package.json
├── vite.config.ts              # Vue + Tailwind plugins, dev proxy
├── tsconfig.json
├── Dockerfile                  # Multi-stage: build + nginx
├── src/
│   ├── main.ts                 # App entry, Pinia + Vue Flow setup
│   ├── App.vue                 # Root component, shell layout
│   ├── style.css               # @import "tailwindcss"; + theme
│   ├── api/
│   │   ├── schema.d.ts         # Generated from openapi-typescript (gitignored)
│   │   ├── client.ts           # Typed fetch wrapper using generated types
│   │   └── workflows.ts        # Workflow CRUD API functions
│   ├── stores/
│   │   ├── canvas.ts           # Vue Flow state: nodes, edges, viewport
│   │   ├── workflow.ts         # Workflow data: current workflow, save/load, dirty flag
│   │   └── ui.ts               # UI state: panel visibility, selection, mode
│   ├── components/
│   │   ├── shell/
│   │   │   ├── AppHeader.vue       # Header bar with title, mode switch, actions
│   │   │   ├── WorkflowRail.vue    # Left sidebar with workflow list
│   │   │   └── CanvasArea.vue      # Center canvas wrapper
│   │   ├── canvas/
│   │   │   ├── StudioCanvas.vue    # Vue Flow instance with config
│   │   │   ├── CanvasControls.vue  # Zoom/fit/add-node toolbar
│   │   │   └── CanvasMinimap.vue   # Minimap wrapper
│   │   └── nodes/
│   │       ├── BaseNode.vue        # Shared node card layout
│   │       ├── AgentNode.vue       # Agent node type
│   │       ├── ExecutionUnitNode.vue
│   │       ├── ApprovalGateNode.vue
│   │       ├── MemoryResourceNode.vue
│   │       ├── ConditionBranchNode.vue
│   │       ├── StartNode.vue
│   │       ├── EndNode.vue
│   │       └── DataMappingNode.vue
│   ├── composables/
│   │   ├── useCanvasActions.ts     # Add node, delete, fit-view
│   │   ├── useWorkflowPersistence.ts # Save/load orchestration
│   │   └── usePortValidation.ts    # Connection validation logic
│   └── types/
│       ├── nodes.ts            # Node type definitions, port configs
│       └── workflow.ts         # Frontend workflow model types
└── dist/                       # Built output (gitignored)
```

### Pattern 1: Custom Node Types with Vue Flow
**What:** Register 8 custom node types via Vue Flow's `nodeTypes` prop. Each custom node component renders handles at configured positions.
**When to use:** Every node on the canvas is a custom type.
**Example:**
```typescript
// Source: https://vueflow.dev/guide/node.html
// In StudioCanvas.vue
import { VueFlow } from '@vue-flow/core'
import AgentNode from '../nodes/AgentNode.vue'
import StartNode from '../nodes/StartNode.vue'
// ... etc

const nodeTypes = {
  agent: AgentNode,
  executionUnit: ExecutionUnitNode,
  approvalGate: ApprovalGateNode,
  memoryResource: MemoryResourceNode,
  conditionBranch: ConditionBranchNode,
  start: StartNode,
  end: EndNode,
  dataMapping: DataMappingNode,
}

// <VueFlow :node-types="nodeTypes" :nodes="nodes" :edges="edges" />
```

### Pattern 2: Typed Port Validation
**What:** Use Vue Flow's `isValidConnection` callback to enforce port type compatibility. Each handle has an `id` encoding its port type.
**When to use:** When a user drags an edge between handles.
**Example:**
```typescript
// Source: https://vueflow.dev/guide/handle.html
import { Connection } from '@vue-flow/core'

function isValidConnection(connection: Connection): boolean {
  const sourcePort = getPortType(connection.sourceHandle)
  const targetPort = getPortType(connection.targetHandle)
  // Enforce type compatibility
  return sourcePort === targetPort || targetPort === 'any'
}

// In custom node component:
// <Handle type="source" :position="Position.Right" id="output-data" />
// <Handle type="target" :position="Position.Left" id="input-data" />
```

### Pattern 3: Pinia Store Separation
**What:** Three stores with clear boundaries. Canvas store owns Vue Flow reactive state. Workflow store handles API persistence. UI store manages shell state.
**When to use:** Always -- this is the state architecture.
**Example:**
```typescript
// stores/canvas.ts
export const useCanvasStore = defineStore('canvas', () => {
  const nodes = ref<Node[]>([])
  const edges = ref<Edge[]>([])

  function addNode(type: string, position: XYPosition) { /* ... */ }
  function removeNode(id: string) { /* ... */ }
  function addEdge(connection: Connection) { /* ... */ }

  return { nodes, edges, addNode, removeNode, addEdge }
})

// stores/workflow.ts
export const useWorkflowStore = defineStore('workflow', () => {
  const currentWorkflowId = ref<string | null>(null)
  const isDirty = ref(false)
  const workflows = ref<WorkflowSummary[]>([])

  async function save() { /* marshal canvas -> API format, POST */ }
  async function load(id: string) { /* GET, unmarshal API -> canvas format */ }

  return { currentWorkflowId, isDirty, workflows, save, load }
})

// stores/ui.ts
export const useUiStore = defineStore('ui', () => {
  const railCollapsed = ref(false)
  const selectedNodeId = ref<string | null>(null)
  const currentMode = ref<'editor' | 'executions'>('editor')

  return { railCollapsed, selectedNodeId, currentMode }
})
```

### Pattern 4: Studio API Router (Backend)
**What:** Dedicated FastAPI APIRouter for Studio endpoints at `/api/studio/v1/`.
**When to use:** All Studio-specific API endpoints.
**Example:**
```python
# src/zeroth/service/studio_api.py
from fastapi import APIRouter, Depends, Request
from zeroth.graph.repository import GraphRepository

router = APIRouter(prefix="/api/studio/v1", tags=["studio"])

@router.get("/workflows")
async def list_workflows(request: Request):
    repo: GraphRepository = request.app.state.bootstrap.graph_repository
    graphs = await repo.list()
    return [_to_summary(g) for g in graphs]

@router.get("/workflows/{workflow_id}")
async def get_workflow(workflow_id: str, request: Request):
    repo: GraphRepository = request.app.state.bootstrap.graph_repository
    graph = await repo.get(workflow_id)
    if graph is None:
        raise HTTPException(404, detail="Workflow not found")
    return _to_studio_graph(graph)

@router.post("/workflows")
async def create_workflow(body: CreateWorkflowRequest, request: Request):
    # ...

@router.put("/workflows/{workflow_id}")
async def update_workflow(workflow_id: str, body: UpdateWorkflowRequest, request: Request):
    # ...

@router.delete("/workflows/{workflow_id}")
async def delete_workflow(workflow_id: str, request: Request):
    # ...

@router.get("/node-types")
async def list_node_types():
    """Return available node type schemas with port definitions."""
    return NODE_TYPE_REGISTRY
```

### Anti-Patterns to Avoid
- **Monolithic Pinia store:** Don't put canvas state, workflow data, and UI state in one store. This creates coupling that makes Phase 23 (undo/redo, inspector) harder.
- **Direct Vue Flow state mutation outside store:** Always go through Pinia actions so state changes are trackable and eventually undoable.
- **Hand-rolling node positioning:** Use Vue Flow's built-in drag system and dagre for auto-layout. Don't write custom position calculation.
- **Coupling frontend node types to backend node_type discriminator values:** The backend has 3 node types (agent, executable_unit, human_approval). The frontend has 8 visual types. Maintain a mapping layer, not a 1:1 relationship.
- **Skipping the frontend-backend graph model mapping:** Backend `Graph` model has execution-focused fields. Frontend needs visual-focused fields (position, dimensions). Create an explicit mapping layer in the API.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Graph canvas with pan/zoom | Custom SVG/Canvas renderer | @vue-flow/core | Handles viewport transforms, node rendering, edge paths, interaction events. Months of work otherwise. |
| Minimap | Custom viewport-scaled rendering | @vue-flow/minimap | Automatic sync with main canvas state. |
| DAG auto-layout | Custom node positioning algorithm | @dagrejs/dagre | Proper layered DAG layout with edge routing. Dagre is the standard. |
| TypeScript types from API | Manual type definitions | openapi-typescript | Types auto-generated from FastAPI OpenAPI spec. Prevents drift. |
| Edge path rendering | Custom bezier math | Vue Flow default edge types | Cubic bezier, step, smoothstep all built in. |
| Drag-and-drop node placement | Custom mouse event handling | Vue Flow `addNodes()` + native drag | Vue Flow handles coordinate transforms between screen and flow space. |
| CSS utility system | Custom CSS classes | Tailwind CSS | Consistent, composable, tree-shaken. |

**Key insight:** Vue Flow handles the entire canvas interaction layer -- viewport transforms, node rendering lifecycle, edge routing, handle interaction, selection, keyboard shortcuts. Building this from scratch would be 3-6 months of work.

## Common Pitfalls

### Pitfall 1: Vue Flow Node Position Coordinate Space
**What goes wrong:** Nodes placed at screen coordinates instead of flow coordinates, causing them to appear at wrong positions after pan/zoom.
**Why it happens:** Vue Flow uses its own coordinate space. Screen coordinates must be converted using `project()` / `screenToFlowPosition()`.
**How to avoid:** Always use `useVueFlow().screenToFlowPosition()` when converting click/drop positions to node positions.
**Warning signs:** Nodes appear in wrong location when canvas is panned or zoomed.

### Pitfall 2: Frontend-Backend Graph Model Mismatch
**What goes wrong:** Frontend graph format (with positions, visual types) conflicts with backend graph format (with execution config, backend node types).
**Why it happens:** Backend `Graph` model has 3 discriminated node types (agent, executable_unit, human_approval). Frontend has 8 visual types. Backend has no position data.
**How to avoid:** Create a Studio-specific graph representation in the API that includes both visual metadata (positions, viewport) and core graph data. Store visual metadata in the graph's `metadata` field or a separate studio-specific storage.
**Warning signs:** Losing node positions on save/load. Frontend types not matching API response shapes.

### Pitfall 3: Tailwind CSS v4 Configuration Differences
**What goes wrong:** Using v3 configuration patterns (tailwind.config.js, PostCSS setup) that don't apply to v4.
**Why it happens:** Most tutorials and training data reference v3. v4 uses a Vite plugin and CSS-based configuration.
**How to avoid:** Use `@tailwindcss/vite` plugin in vite.config.ts. Configure theme via `@theme` directive in CSS, not a JS config file. Custom colors go in `src/style.css` with `@theme { ... }`.
**Warning signs:** PostCSS errors, config file not being read, `tailwind.config.js` having no effect.

### Pitfall 4: Vite Proxy Not Working for Studio API Prefix
**What goes wrong:** Dev proxy fails because the rewrite removes the wrong prefix or doesn't match the `/api/studio/v1/` path.
**Why it happens:** Proxy rewrite rules are tricky with nested prefixes.
**How to avoid:** Proxy `/api/` to the FastAPI backend WITHOUT path rewriting. FastAPI routes are already mounted at `/api/studio/v1/`, so the path should pass through unchanged.
**Warning signs:** 404s from the backend during development.

### Pitfall 5: Nginx SPA Routing Fallback Missing
**What goes wrong:** Direct URL navigation or page refresh returns 404 in production.
**Why it happens:** Nginx serves files from disk. Client-side routes don't have corresponding files.
**How to avoid:** Add `try_files $uri $uri/ /index.html;` in the Nginx location block for the Studio SPA.
**Warning signs:** Any non-root URL returns 404 after deployment.

### Pitfall 6: openapi-typescript Running Against Wrong Spec
**What goes wrong:** Generated types include deployment API routes (the existing `/v1/` surface) but not Studio routes, or vice versa.
**Why it happens:** FastAPI generates one OpenAPI spec for the entire app. Studio types generation may include irrelevant operations.
**How to avoid:** Either (a) generate from the full spec and use only the `paths['/api/studio/v1/...']` types, or (b) create a dedicated OpenAPI spec endpoint for the Studio API using FastAPI's `openapi_url` on a sub-application. Option (a) is simpler.
**Warning signs:** Type file contains deployment API types, confusing frontend developers.

### Pitfall 7: Docker Build Context Too Large
**What goes wrong:** Docker build is slow because it copies node_modules, .venv, or other large directories.
**Why it happens:** Missing or incomplete `.dockerignore`.
**How to avoid:** Create `.dockerignore` in `apps/studio/` excluding `node_modules/`, `dist/`, `.git/`.
**Warning signs:** Docker build taking minutes when it should take seconds.

## Code Examples

### Vite Configuration with Proxy and Tailwind
```typescript
// apps/studio/vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [vue(), tailwindcss()],
  server: {
    port: 5173,
    proxy: {
      '/api/': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        // No rewrite -- FastAPI routes are at /api/studio/v1/
      },
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: false,
  },
})
```

### Tailwind CSS v4 Theme (CSS-based)
```css
/* apps/studio/src/style.css */
@import "tailwindcss";

@theme {
  --color-studio-text: #123044;
  --color-studio-accent: rgba(79, 205, 255, 1);
  --color-studio-border: rgba(118, 182, 205, 0.3);
  --color-studio-edge: rgba(79, 180, 220, 0.6);
  --font-family-studio: 'Inter Tight', 'IBM Plex Sans', 'Segoe UI', sans-serif;
  --radius-panel: 12px;
  --radius-control: 8px;
}
```

### Custom Node Component with Handles
```vue
<!-- apps/studio/src/components/nodes/AgentNode.vue -->
<script setup lang="ts">
import { Handle, Position } from '@vue-flow/core'
import type { NodeProps } from '@vue-flow/core'

const props = defineProps<NodeProps>()
</script>

<template>
  <div class="node-card">
    <Handle type="target" :position="Position.Left" id="input-data" />
    <div class="node-icon"><!-- sparkle SVG --></div>
    <div class="node-title">{{ data.label }}</div>
    <div class="node-meta">AGENT</div>
    <Handle type="source" :position="Position.Right" id="output-data" />
  </div>
</template>
```

### Studio Nginx Configuration (Production)
```nginx
# docker/nginx/studio.conf
server {
    listen 80;
    server_name _;

    # Serve Vue SPA static files
    location / {
        root /usr/share/nginx/html/studio;
        index index.html;
        try_files $uri $uri/ /index.html;

        # Cache static assets aggressively
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # Proxy API requests to FastAPI
    location /api/ {
        proxy_pass http://zeroth:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 300s;
    }

    # Health and existing API passthrough
    location /health {
        proxy_pass http://zeroth:8000;
    }
    location /v1/ {
        proxy_pass http://zeroth:8000;
    }

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;
    gzip_min_length 1000;
}
```

### Multi-Stage Dockerfile for Studio
```dockerfile
# apps/studio/Dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html/studio
# Nginx config is mounted via docker-compose volume
EXPOSE 80
```

### Mounting Studio Router in FastAPI
```python
# In src/zeroth/service/app.py (addition)
from zeroth.service.studio_api import router as studio_router
app.include_router(studio_router)
# Studio routes are at /api/studio/v1/* with auth middleware applied
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Tailwind v3 + PostCSS + tailwind.config.js | Tailwind v4 + @tailwindcss/vite + CSS @theme | Jan 2025 | Simpler config, no PostCSS, CSS-native theming |
| Vite 5 | Vite 6 | Late 2025 | Environment API changes, faster builds |
| Vue Flow 1.x has been stable | Still 1.x (no v2 yet) | Current | Stable API, no migration concerns |
| openapi-typescript v6 | openapi-typescript v7 | 2024 | Better OAS 3.1 support, faster |

**Deprecated/outdated:**
- Tailwind CSS `tailwind.config.js` file: Still works in v4 but the CSS-based `@theme` is the new standard approach
- Vuex: Replaced by Pinia as official Vue state management. Do not use Vuex.
- vue-cli: Replaced by Vite. Do not use vue-cli.

## Open Questions

1. **Node position storage in backend**
   - What we know: Backend `Graph` model has `metadata: dict[str, Any]` field. Node models have `execution_config: dict`.
   - What's unclear: Best place to store x/y positions -- in node metadata, graph metadata, or a separate Studio-specific table.
   - Recommendation: Store positions in each node's `execution_config` or a new `display` field. The `DisplayMetadata` model already exists on nodes. Alternatively, use graph-level `metadata["studio"]` to store a position map. Either works; graph-level metadata is simpler for save/load.

2. **Frontend-to-backend node type mapping**
   - What we know: Backend has 3 node types (agent, executable_unit, human_approval). Frontend has 8 visual types.
   - What's unclear: How to represent Start, End, Condition/Branch, Data Mapping, Memory Resource in the backend.
   - Recommendation: Some of these map naturally -- Start/End are execution config markers, Condition/Branch maps to edge conditions, Memory Resource maps to agent memory_refs. The API should accept the 8 visual types and translate to backend types. Start/End can be lightweight stub nodes stored in graph metadata or as special-cased nodes.

3. **Auth middleware scope for Studio routes**
   - What we know: Existing auth middleware applies to all non-health routes. Studio API needs auth (D-10).
   - What's unclear: Whether the existing middleware at `/api/studio/v1/` works without modification since it currently checks paths starting with `/health` for bypass.
   - Recommendation: The existing middleware should work -- it authenticates everything except `/health*`. Verify during implementation that `/api/studio/v1/` routes go through auth correctly.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Frontend build + dev | Yes | 25.2.1 | -- |
| npm | Package management | Yes | 11.9.0 | -- |
| Docker | Production deployment | Yes | 29.2.0 | -- |
| Python 3.12+ | Backend | Yes | (project .python-version) | -- |
| Nginx | Static file serving | Yes (Docker image) | 1.27-alpine (in compose) | -- |

**Missing dependencies with no fallback:** None

**Missing dependencies with fallback:** None

## Project Constraints (from CLAUDE.md)

- **Build/test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Project layout:** `src/zeroth/` for backend, `tests/` for pytest tests
- **Context efficiency:** Read only what's needed for the current task
- **Implementation tracking:** PROGRESS.md is single source of truth

## Sources

### Primary (HIGH confidence)
- Vue Flow official docs (https://vueflow.dev/guide/node.html, https://vueflow.dev/guide/handle.html) - Custom nodes, handles, typed connections
- Tailwind CSS v4 installation guide (https://tailwindcss.com/docs/guides/vite) - Vite plugin setup
- FastAPI docs (https://fastapi.tiangolo.com/tutorial/cors/) - CORS and proxy patterns
- npm registry - Verified all package versions via `npm view`

### Secondary (MEDIUM confidence)
- Vite proxy configuration patterns (https://www.thatsoftwaredude.com/content/14032/setting-up-a-dev-server-proxy-in-vite) - Dev proxy setup
- Docker multi-stage Vue builds (https://dev.to/it-wibrc/guide-to-containerizing-a-modern-javascript-spa-vuevitereact-with-a-multi-stage-nginx-build-1lma) - Nginx SPA serving
- openapi-typescript GitHub (https://github.com/openapi-ts/openapi-typescript) - Type generation from OpenAPI specs

### Tertiary (LOW confidence)
- None -- all findings verified with official sources or npm registry.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All packages verified against npm registry with current versions. Locked decisions from CONTEXT.md.
- Architecture: HIGH - Patterns directly from Vue Flow docs, existing codebase patterns (FastAPI router, graph repository), and established Vue/Pinia patterns.
- Pitfalls: HIGH - Based on known coordinate-space issues in flow editors, verified Tailwind v4 config differences, standard SPA/Nginx gotchas.

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (30 days -- stable ecosystem, no fast-moving dependencies)
