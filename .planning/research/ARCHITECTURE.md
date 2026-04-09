# Architecture Patterns

**Domain:** Visual workflow authoring UI (Zeroth Studio) integrated with existing governed multi-agent platform
**Researched:** 2026-04-09

## Recommended Architecture

### High-Level Overview

```
Browser (Vue 3 SPA)
  |
  +-- Nginx (TLS termination, static files, reverse proxy)
  |     |
  |     +-- /studio/*  -->  Vue SPA static assets (index.html fallback)
  |     +-- /v1/*      -->  FastAPI backend (REST API)
  |     +-- /v2/*      -->  FastAPI backend (Studio authoring API)
  |     +-- /ws/*      -->  FastAPI WebSocket (canvas events)
  |     +-- /health/*  -->  FastAPI health probes
  |
  +-- FastAPI (existing service + new studio routes)
        |
        +-- Existing v1 API (runs, approvals, audit, cost, webhooks)
        +-- New v2 Studio API (graph authoring, asset CRUD, environments)
        +-- New WebSocket hub (canvas sync, execution status, validation)
        +-- Existing storage layer (Postgres/SQLite, Redis, ARQ)
```

### Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| **Nginx** | TLS termination, static file serving, reverse proxy, WebSocket upgrade | Browser, FastAPI |
| **Vue SPA** | Visual graph editor, inspector panel, workflow rail, asset forms | Nginx (REST + WS) |
| **FastAPI v1 API** | Existing deployment-bound runtime API (unchanged) | Storage, ARQ, Regulus |
| **FastAPI v2 Studio API** | Graph authoring CRUD, asset management, environment config, validation | GraphRepository, Storage |
| **WebSocket Hub** | Real-time canvas events, validation feedback, execution status | Pinia stores (client), GraphRepository (server) |
| **GraphRepository** | Persistence layer for versioned graph documents (existing) | AsyncDatabase |

---

## Decision 1: Serve Vue App from FastAPI or Separate?

**Recommendation: Nginx serves static files; FastAPI serves only API.**

### Rationale

The existing architecture already has Nginx as the TLS-terminating reverse proxy in front of FastAPI. Adding static file serving to Nginx is trivial and keeps FastAPI focused on API logic. FastAPI's `StaticFiles` mount works but adds unnecessary load to the Python process for serving assets that Nginx handles far more efficiently.

### Implementation

Vite builds the Vue SPA to a `dist/` directory. The Docker build copies this into a volume that Nginx serves. Nginx handles SPA routing via `try_files $uri /index.html` for all `/studio/` paths.

```
# Updated nginx.conf structure
location /studio/ {
    alias /usr/share/nginx/html/studio/;
    try_files $uri $uri/ /studio/index.html;
}

location /v1/ {
    proxy_pass http://zeroth:8000;
    # ... existing proxy headers
}

location /v2/ {
    proxy_pass http://zeroth:8000;
    # ... same proxy headers
}

location /ws/ {
    proxy_pass http://zeroth:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

### Why NOT FastAPI StaticFiles

- FastAPI is single-process (uvicorn) -- serving static assets wastes its event loop
- Nginx can serve static files with zero-copy sendfile, gzip, and caching headers
- Existing Nginx container is already deployed; no new infrastructure needed
- Clear separation: Nginx = static + proxy, FastAPI = API only

### Why NOT Separate Origins (CORS)

- Adds CORS complexity for every API call and WebSocket connection
- Separate deployment pipelines for frontend and backend
- Cookie/auth token management becomes harder
- Same-origin via Nginx reverse proxy is simpler and more secure

---

## Decision 2: REST vs GraphQL vs Hybrid for Graph Authoring API

**Recommendation: REST-only under `/v2/studio/` prefix.**

### Rationale

The existing platform is 100% REST with OpenAPI. GraphQL adds a dependency (Strawberry/Ariadne), a new query language for the team to learn, and a different error handling model -- all for marginal benefit. The graph authoring domain has predictable, resource-oriented operations (CRUD on graphs, nodes, edges, assets) that map cleanly to REST.

GraphQL's main advantage (reducing over-fetching in deeply nested queries) does not apply here. A graph document is fetched as a whole (it is a single Pydantic model stored as a JSON blob). There is no N+1 problem because the graph is already denormalized.

### v2 Studio API Surface

```
# Graph authoring
GET    /v2/studio/graphs                      # List graphs
POST   /v2/studio/graphs                      # Create graph
GET    /v2/studio/graphs/{id}                  # Get graph (latest version)
GET    /v2/studio/graphs/{id}/versions/{ver}   # Get specific version
PUT    /v2/studio/graphs/{id}                  # Update draft graph
POST   /v2/studio/graphs/{id}/publish          # Publish graph
POST   /v2/studio/graphs/{id}/clone            # Clone to new draft
DELETE /v2/studio/graphs/{id}                  # Archive graph
GET    /v2/studio/graphs/{id}/diff/{v1}/{v2}   # Diff two versions

# Node operations (convenience endpoints over graph mutations)
POST   /v2/studio/graphs/{id}/nodes            # Add node
PUT    /v2/studio/graphs/{id}/nodes/{nid}      # Update node
DELETE /v2/studio/graphs/{id}/nodes/{nid}      # Remove node
PATCH  /v2/studio/graphs/{id}/nodes/{nid}/position  # Move node (canvas drag)

# Edge operations
POST   /v2/studio/graphs/{id}/edges            # Add edge
PUT    /v2/studio/graphs/{id}/edges/{eid}      # Update edge
DELETE /v2/studio/graphs/{id}/edges/{eid}      # Remove edge

# Validation
POST   /v2/studio/graphs/{id}/validate         # Validate graph

# Asset catalog (reusable components)
GET    /v2/studio/assets/node-types             # Available node types
GET    /v2/studio/assets/contracts              # Available contracts
GET    /v2/studio/assets/policies               # Available policies

# Environment management
GET    /v2/studio/environments                  # List environments
POST   /v2/studio/environments                  # Create environment
PUT    /v2/studio/environments/{id}             # Update environment
```

### Why NOT GraphQL

- Existing codebase is 100% REST; GraphQL would be a foreign paradigm
- Graph document is a denormalized JSON blob -- no over-fetching problem
- OpenAPI spec generation works out of the box with FastAPI
- No subscriptions needed (WebSocket hub handles real-time)
- Adds Strawberry/Ariadne dependency and learning curve for zero gain

### Mapping to Existing Pydantic Models

The v2 Studio API uses the **exact same Pydantic models** from `zeroth.graph.models` (Graph, Node, Edge). No translation layer is needed. The Studio API is a thin REST wrapper around `GraphRepository` operations:

| Studio API Operation | Backend Implementation |
|---------------------|----------------------|
| Create graph | `GraphRepository.create(Graph(...))` |
| Update draft | `GraphRepository.save(graph)` |
| Publish | `GraphRepository.publish(graph_id)` |
| Clone | `GraphRepository.clone_published_to_draft(graph_id)` |
| Add node | Load graph, append node to `graph.nodes`, save |
| Move node | Load graph, update node position in `DisplayMetadata`, save |
| Add edge | Load graph, append edge to `graph.edges`, save |
| Validate | `GraphValidator.validate(graph)` |
| Diff | `GraphRepository.diff(graph_id, v1, v2)` |

The critical insight: graphs are stored as a single JSON document. Node/edge CRUD endpoints are **convenience APIs** that load the graph, mutate it in memory, re-validate, and save. They do NOT correspond to separate database tables.

---

## Decision 3: WebSocket Architecture for Real-Time Canvas Updates

**Recommendation: Single WebSocket connection per canvas session, topic-based message routing.**

### Architecture

```
Client (Vue SPA)                          Server (FastAPI)
  |                                          |
  +-- connect /ws/studio/{graph_id} -------> WebSocket endpoint
  |     (JWT in query param or first msg)    |
  |                                          +-- Authenticate
  |                                          +-- Register in ConnectionManager
  |                                          |
  +-- { type: "node:move", ... } ----------> Process mutation
  |                                          +-- Update GraphRepository
  |                                          +-- Broadcast to other clients
  |                                          |
  <-- { type: "graph:updated", ... } ------- Broadcast
  <-- { type: "validation:result", ... } --- Push validation results
  <-- { type: "execution:status", ... } ---- Push execution status
```

### Message Protocol

```typescript
// Client -> Server
interface ClientMessage {
  type: "node:move" | "node:add" | "edge:add" | "edge:remove"
        | "graph:save" | "graph:validate" | "ping";
  payload: Record<string, unknown>;
  seq: number;  // Client sequence number for ack
}

// Server -> Client
interface ServerMessage {
  type: "graph:updated" | "graph:saved" | "validation:result"
        | "execution:status" | "error" | "ack" | "pong";
  payload: Record<string, unknown>;
  seq: number;      // Server sequence number
  ack_seq?: number; // Acknowledges client seq
}
```

### Why Single WebSocket (Not REST Polling)

- Node drag operations generate rapid position updates (10-30/sec during drag)
- Validation results should appear immediately after edge changes
- Execution status needs to stream without polling delay
- Future multi-user collaboration requires broadcast capability

### Why NOT Full CRDT/OT Collaboration (Yet)

Multi-user real-time co-editing (like Google Docs) requires CRDTs or Operational Transforms. This is extremely complex for graph structures and is NOT needed for v2.0. The WebSocket hub supports:

1. **Single-user real-time**: Instant validation feedback, execution status
2. **Presence awareness**: Show who else has the graph open (read-only indicator)
3. **Optimistic locking**: Last-write-wins with conflict detection on save

Full collaboration can be added later by upgrading the WebSocket protocol without changing the architecture.

### Connection Manager (Server-Side)

```python
class StudioConnectionManager:
    """Manages WebSocket connections grouped by graph_id."""
    
    def __init__(self):
        self._connections: dict[str, list[WebSocket]] = {}  # graph_id -> connections
    
    async def connect(self, graph_id: str, websocket: WebSocket, principal: Principal):
        await websocket.accept()
        self._connections.setdefault(graph_id, []).append(websocket)
    
    async def broadcast(self, graph_id: str, message: dict, exclude: WebSocket | None = None):
        for ws in self._connections.get(graph_id, []):
            if ws != exclude:
                await ws.send_json(message)
```

For horizontal scaling (multiple uvicorn workers), broadcast messages go through Redis pub/sub -- the existing Redis infrastructure is already deployed.

---

## Decision 4: Vue Frontend Structure

**Recommendation: Feature-sliced architecture organized by domain, not by technical layer.**

### Directory Structure

```
studio/                          # Vue 3 SPA root
  src/
    app/                         # App shell, routing, global providers
      App.vue
      router.ts
      main.ts
    
    features/                    # Feature modules (domain-driven)
      canvas/                    # Graph canvas (Vue Flow wrapper)
        components/
          CanvasView.vue         # Main canvas component wrapping VueFlow
          CanvasNode.vue         # Custom node renderer (governance decorators)
          CanvasEdge.vue         # Custom edge renderer
          CanvasMinimap.vue
          CanvasControls.vue
        composables/
          useCanvasMapping.ts    # Maps Zeroth Graph <-> Vue Flow elements
          useCanvasOperations.ts # Node/edge CRUD with undo/redo
          useCanvasDrag.ts       # Drag-drop from node palette
          useAutoLayout.ts       # Dagre-based auto-layout
        stores/
          canvasStore.ts         # Canvas-specific state (zoom, selection, mode)
        types/
          canvas.ts
      
      inspector/                 # Right-side property inspector
        components/
          InspectorPanel.vue
          NodeInspector.vue      # Edit selected node properties
          EdgeInspector.vue      # Edit selected edge properties
          AgentConfig.vue        # Agent node configuration
          ApprovalConfig.vue     # Approval node configuration
          ExecUnitConfig.vue     # Executable unit configuration
        composables/
          useInspector.ts
      
      workflow-rail/             # Left sidebar: workflow list + navigation
        components/
          WorkflowRail.vue
          WorkflowList.vue
          WorkflowCard.vue
        stores/
          workflowListStore.ts
      
      validation/                # Graph validation feedback
        components/
          ValidationPanel.vue
          ValidationBadge.vue
        composables/
          useValidation.ts
      
      execution/                 # Execution monitoring
        components/
          ExecutionPanel.vue
          ExecutionTimeline.vue
        composables/
          useExecution.ts
      
      environments/              # Environment management
        components/
          EnvironmentEditor.vue
        stores/
          environmentStore.ts
    
    shared/                      # Cross-cutting, no business logic
      api/
        client.ts               # Axios/fetch wrapper for REST API
        websocket.ts            # WebSocket client with reconnection
        types.ts                # API response types (generated from OpenAPI)
      ui/
        Button.vue
        Modal.vue
        CodeEditor.vue          # CodeMirror 6 wrapper
      composables/
        useAuth.ts
        useToast.ts
      stores/
        graphStore.ts           # Central graph document state (Pinia)
        authStore.ts            # Auth state
    
    types/                       # Shared TypeScript types
      graph.ts                  # Mirrors zeroth.graph.models Pydantic schema
      api.ts
```

### Why Feature-Sliced Over Flat Technical Layers

| Approach | Pros | Cons |
|----------|------|------|
| **Feature-sliced (recommended)** | Co-locates related code, scales with features, clear ownership | Shared code needs explicit boundary |
| Flat layers (`/components`, `/stores`, `/composables`) | Simple for small apps | Everything in one folder at scale, unclear dependencies |
| Full FSD (7 layers) | Very rigorous | Overengineered for a team of 1-3, high ceremony |

Feature-sliced is the pragmatic middle ground: organized by domain (canvas, inspector, validation) with a `shared/` layer for truly cross-cutting concerns.

### Critical Composable: `useCanvasMapping`

The most important piece of frontend architecture. Maps between Zeroth's `Graph` Pydantic model and Vue Flow's internal representation:

```typescript
// Zeroth Graph model (from backend)
interface ZerothGraph {
  graph_id: string;
  nodes: ZerothNode[];  // AgentNode | ExecutableUnitNode | HumanApprovalNode
  edges: ZerothEdge[];
  // ...
}

// Vue Flow model (canvas rendering)
interface VueFlowNode {
  id: string;
  type: string;           // 'agent' | 'executable_unit' | 'human_approval'
  position: { x: number, y: number };
  data: { zerothNode: ZerothNode };  // Embed full node data for inspector
}

// useCanvasMapping bridges these bidirectionally
function useCanvasMapping(graph: Ref<ZerothGraph>) {
  const vfNodes = computed(() => graph.value.nodes.map(toVueFlowNode));
  const vfEdges = computed(() => graph.value.edges.map(toVueFlowEdge));
  
  function applyVueFlowChange(changes: NodeChange[]) {
    // Update Zeroth graph from Vue Flow events (position, selection)
  }
  
  return { vfNodes, vfEdges, applyVueFlowChange };
}
```

### State Management: Pinia with Graph as Single Source of Truth

```
graphStore (Pinia)          -- THE source of truth for graph document
  |
  +-- canvasStore           -- Canvas viewport state (zoom, pan, selection)
  +-- workflowListStore     -- List of graphs for rail sidebar
  +-- environmentStore      -- Environment config
  
useCanvasMapping            -- Reactive bridge: graphStore <-> Vue Flow
useCanvasOperations         -- Mutation API with undo/redo history
```

The `graphStore` holds the authoritative `Graph` object. Canvas mutations go through `useCanvasOperations` which updates the store and optionally sends WebSocket messages. The canvas reactively re-renders from the store via `useCanvasMapping`.

---

## Decision 5: Build and Deployment Integration

**Recommendation: Multi-stage Docker build; Vite builds in Node stage, output copied to Nginx stage.**

### Updated Dockerfile Strategy

```dockerfile
# Stage 1: Build Vue SPA
FROM node:20-alpine AS frontend-builder
WORKDIR /app/studio
COPY studio/package.json studio/package-lock.json ./
RUN npm ci
COPY studio/ ./
RUN npm run build  # Output: /app/studio/dist/

# Stage 2: Build Python backend (existing, unchanged)
FROM python:3.12-slim AS backend-builder
# ... existing builder stage ...

# Stage 3: Runtime
FROM python:3.12-slim
# ... existing runtime setup ...
COPY --from=backend-builder /app/.venv /app/.venv
COPY --from=backend-builder /app/src /app/src

# Copy built SPA into Nginx-served directory
COPY --from=frontend-builder /app/studio/dist /app/studio-dist
```

### Updated docker-compose.yml

```yaml
services:
  zeroth:
    build: .
    # ... existing config unchanged ...

  nginx:
    image: nginx:1.27-alpine
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./docker/nginx/certs:/etc/nginx/certs:ro
      # Mount built SPA assets from zeroth container or build context
    depends_on:
      - zeroth
```

Two viable approaches for getting built assets to Nginx:

1. **Shared volume** (recommended for simplicity): The zeroth container copies `studio-dist` to a shared Docker volume on startup. Nginx serves from that volume.
2. **Build Nginx image with assets baked in**: A separate Dockerfile for Nginx that includes the SPA in the image.

Option 1 is simpler and avoids a second custom image build.

### Development Workflow

```bash
# Backend (existing)
uv sync && uv run uvicorn zeroth.service.entrypoint:app_factory --factory --reload

# Frontend (new)
cd studio && npm run dev  # Vite dev server on :5173, proxies API to :8000

# vite.config.ts proxy configuration
export default defineConfig({
  server: {
    proxy: {
      '/v1': 'http://localhost:8000',
      '/v2': 'http://localhost:8000',
      '/ws': { target: 'ws://localhost:8000', ws: true },
      '/health': 'http://localhost:8000',
    }
  },
  base: '/studio/',
})
```

This gives hot module replacement (HMR) for the frontend while the backend runs natively.

---

## Patterns to Follow

### Pattern 1: Optimistic UI with Server Reconciliation

**What:** Apply canvas changes immediately in the UI, then persist to backend. If the backend rejects (validation failure), roll back the UI change.

**When:** All canvas operations (node add/move/delete, edge connect/disconnect).

**Why:** Canvas interactions must feel instant (< 16ms). Waiting for a server round-trip on every node drag would make the UI unusable.

```typescript
// useCanvasOperations.ts
async function moveNode(nodeId: string, position: Position) {
  // 1. Optimistic: update store immediately
  graphStore.updateNodePosition(nodeId, position);
  
  // 2. Debounced: send to server (batch position updates during drag)
  debouncedSync(() => {
    api.patch(`/v2/studio/graphs/${graphId}/nodes/${nodeId}/position`, { position });
  });
}
```

### Pattern 2: Command Pattern for Undo/Redo

**What:** Every canvas mutation is a reversible command object. Undo pops the command stack and applies the inverse.

**When:** Node add/remove, edge add/remove, node property changes.

```typescript
interface CanvasCommand {
  execute(): void;
  undo(): void;
  description: string;
}

class AddNodeCommand implements CanvasCommand {
  constructor(private graph: Graph, private node: ZerothNode) {}
  execute() { this.graph.nodes.push(this.node); }
  undo() { this.graph.nodes = this.graph.nodes.filter(n => n.node_id !== this.node.node_id); }
}
```

### Pattern 3: Debounced Auto-Save with Dirty Tracking

**What:** Track whether the graph has unsaved changes. Auto-save after a period of inactivity (e.g., 2 seconds). Show "Saving..." / "Saved" indicator.

**When:** Any graph mutation.

**Why:** Prevents data loss without overwhelming the server with saves on every keystroke.

### Pattern 4: Governance-Aware Node Rendering

**What:** Custom Vue Flow node components that visually indicate governance state -- approval gates show a shield icon, sandboxed nodes show a lock, nodes with policy bindings show a badge.

**When:** All node rendering in the canvas.

**Why:** This is Zeroth's differentiator from n8n. The canvas must make governance visible, not hidden in config panels.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Dual Source of Truth for Graph State

**What:** Storing the graph document separately in both the Pinia store and Vue Flow's internal state, with manual synchronization between them.

**Why bad:** Synchronization bugs cause the canvas and inspector to show different data. Changes in one are not reflected in the other.

**Instead:** Pinia `graphStore` is the single source of truth. Vue Flow receives computed props derived from the store. Vue Flow events write back to the store through `useCanvasOperations`.

### Anti-Pattern 2: Per-Field REST Endpoints for Graph Mutations

**What:** Creating individual REST endpoints for every possible graph field mutation (node name, node instruction, edge condition expression, etc.).

**Why bad:** The graph is a single JSON document. Dozens of fine-grained endpoints create a huge API surface that is hard to maintain and version.

**Instead:** Use coarse-grained endpoints: update the entire node, or update the entire graph. The frontend batches changes. Node/edge convenience endpoints are the minimum viable granularity.

### Anti-Pattern 3: WebSocket for All API Operations

**What:** Routing CRUD operations through WebSocket instead of REST.

**Why bad:** WebSocket has no built-in request/response semantics, no status codes, no caching, no OpenAPI spec generation. Error handling is ad-hoc.

**Instead:** REST for CRUD (create, read, update, delete). WebSocket for push notifications (validation results, execution status, presence). The canvas can send rapid position updates via WebSocket as a performance optimization, but graph saves go through REST.

### Anti-Pattern 4: Putting Graph Layout in the Backend

**What:** Computing auto-layout (dagre) on the server side.

**Why bad:** Layout is a UI concern. The backend should not know about pixel positions. Layout computation is fast in the browser. Sending layout requests to the server adds latency for no reason.

**Instead:** Dagre runs in the browser. Node positions are stored in `DisplayMetadata` on the graph model and persisted, but layout computation is client-only.

---

## New vs Modified Components

### New Components (Studio-Specific)

| Component | Location | Purpose |
|-----------|----------|---------|
| `studio_api.py` | `src/zeroth/service/studio_api.py` | v2 REST routes for graph authoring |
| `studio_ws.py` | `src/zeroth/service/studio_ws.py` | WebSocket hub for canvas events |
| `connection_manager.py` | `src/zeroth/service/connection_manager.py` | WebSocket connection tracking |
| `studio/` | `studio/` (project root) | Entire Vue SPA (new directory) |

### Modified Components (Existing)

| Component | Modification |
|-----------|-------------|
| `service/app.py` | Add v2 router, mount WebSocket endpoint |
| `graph/models.py` | Add `position` field to `DisplayMetadata` for canvas coordinates |
| `docker-compose.yml` | Add studio volume mount to Nginx |
| `docker/nginx/nginx.conf` | Add `/studio/`, `/v2/`, `/ws/` locations |
| `Dockerfile` | Add Node.js frontend build stage |

### Unchanged Components

Everything in `v1/` remains untouched. The runtime API, orchestrator, approvals, audit, cost, webhooks, dispatch -- all unchanged. The Studio API is a new surface that reads/writes the same `GraphRepository`.

---

## Data Flow: End-to-End Graph Authoring

```
1. User opens Studio (/studio/)
   -> Nginx serves Vue SPA (index.html)
   -> Vue app loads, authenticates, fetches graph list
      GET /v2/studio/graphs -> graphStore.setGraphList(response)

2. User selects a graph
   -> GET /v2/studio/graphs/{id} -> graphStore.setCurrentGraph(response)
   -> useCanvasMapping computes Vue Flow nodes/edges from graphStore
   -> Vue Flow renders the canvas
   -> WebSocket connects to /ws/studio/{graph_id}

3. User drags a node
   -> Vue Flow emits onNodeDragStop
   -> useCanvasOperations.moveNode() updates graphStore (optimistic)
   -> WebSocket sends { type: "node:move", ... } (debounced)
   -> Server updates graph, broadcasts to other clients

4. User adds an edge
   -> Vue Flow emits onConnect
   -> useCanvasOperations.addEdge() updates graphStore (optimistic)
   -> REST POST /v2/studio/graphs/{id}/edges
   -> Server validates graph (edge creates cycle? missing contracts?)
   -> WebSocket pushes { type: "validation:result", ... }
   -> ValidationPanel shows issues if any

5. User clicks "Publish"
   -> REST POST /v2/studio/graphs/{id}/publish
   -> Server runs full validation via GraphValidator
   -> If valid: graph status -> PUBLISHED, response 200
   -> If invalid: response 422 with validation issues
   -> UI shows success/failure

6. User clicks "Deploy" (links to existing v1 deployment flow)
   -> Existing deployment machinery creates a deployment from the published graph
```

---

## Build Order (Dependency-Aware)

Recommended implementation sequence based on component dependencies:

1. **Backend: v2 Studio REST API** -- Thin CRUD wrapper around existing `GraphRepository`. No frontend dependency. Can be tested with curl/httpie immediately.

2. **Frontend: App shell + Router + API client** -- Skeleton Vue app with Vite, basic routing, API client configured with proxy. No canvas yet.

3. **Frontend: Graph store + Canvas** -- Pinia graphStore, `useCanvasMapping`, basic Vue Flow rendering of a graph fetched from v2 API. This is the core integration point.

4. **Backend: WebSocket hub** -- Connection manager, message protocol. Tested independently with wscat.

5. **Frontend: Canvas operations + Inspector** -- Node/edge CRUD, inspector panel, undo/redo. Depends on canvas and WebSocket.

6. **Frontend: Governance decorators** -- Custom node renderers showing approval gates, sandbox badges, policy indicators. Depends on canvas.

7. **Frontend: Workflow rail + Validation panel** -- List view, validation feedback. Can be built in parallel with step 5-6.

8. **Deployment: Docker multi-stage build** -- Integrate Vite build into Dockerfile, update Nginx config. Last because it is mechanical.

## Sources

- [Vue Flow documentation](https://vueflow.dev/)
- [Vue Flow GitHub](https://github.com/bcakmakoglu/vue-flow)
- [n8n Canvas Architecture (DeepWiki)](https://deepwiki.com/n8n-io/n8n/6.2-workflow-canvas-and-node-management)
- [n8n Frontend Architecture (DeepWiki)](https://deepwiki.com/gwolf999/n8n/3-frontend-architecture)
- [Feature-Sliced Design for Vue](https://feature-sliced.design/blog/vue-application-architecture)
- [Vue Best Practices 2026](https://onehorizon.ai/blog/vue-best-practices-in-2026-architecting-for-speed-scale-and-sanity)
- [FastAPI WebSocket docs](https://fastapi.tiangolo.com/advanced/websockets/)
- [FastAPI Static Files docs](https://fastapi.tiangolo.com/tutorial/static-files/)
- [Serving Vue from FastAPI](https://dimmaski.com/serve-vue-fastapi/)
