# Phase 22: Canvas Foundation & Dev Infrastructure - Context

**Gathered:** 2026-04-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 22 delivers the foundational Studio frontend application, an interactive graph canvas with node placement and edge drawing, a graph authoring REST API, and Docker/Nginx production serving. This is the first phase of the v2.0 Studio milestone — it establishes the Vue app scaffold, canvas interaction layer, backend API surface, and deployment infrastructure that all subsequent Studio phases build upon.

This phase does NOT deliver the palette sidebar (Phase 23), property inspector editing (Phase 23), undo/redo (Phase 23), WebSocket real-time updates (Phase 24), or governance visualization (Phase 25).

</domain>

<decisions>
## Implementation Decisions

### Frontend Project Setup
- **D-01:** Production Studio app lives in `apps/studio/` (new directory). Keep `apps/studio-mockups/` as reference only.
- **D-02:** Vue 3 + Vite scaffold with Vue Flow (@vue-flow/core), Pinia for state management, dagre for auto-layout, Tailwind CSS for styling.
- **D-03:** Frontend types generated from backend OpenAPI spec using `openapi-typescript` at build time (INFRA-02).
- **D-04:** Pinia stores for canvas state, workflow data, and UI state — separate concerns across stores.

### Initial Node Types
- **D-05:** Extended node type set (8 types): Agent, Execution Unit, Approval Gate, Memory Resource, Condition/Branch, Start, End, Data Mapping.
- **D-06:** Each node has typed input and output ports. Edges connect output-to-input. Port types enforce valid connections. Uses Vue Flow's handle system.
- **D-07:** Phase 22 nodes are visual placement + label only. No inline config editing — that's Phase 23 inspector (CANV-04).

### API Surface Design
- **D-08:** Dedicated Studio API router in `src/zeroth/service/studio_api.py`, mounted on the existing FastAPI app.
- **D-09:** URL prefix: `/api/studio/v1/` — namespaced separately from the existing `/v1/` deployment API.
- **D-10:** Authentication required from the start — reuse existing auth middleware from `src/zeroth/service/auth.py`. RBAC enforcement is a downstream requirement (GOV-04).

### Docker Serving
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

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product & UX Direction
- `docs/superpowers/specs/2026-03-29-zeroth-studio-design.md` -- Validated Studio shell, navigation, asset, environment, and runtime UX decisions

### Existing Backend Foundation
- `src/zeroth/service/app.py` -- FastAPI app factory and route registration (mount point for new Studio API router)
- `src/zeroth/service/bootstrap.py` -- Service wiring, auth, repositories
- `src/zeroth/service/auth.py` -- Auth middleware to reuse for Studio API
- `src/zeroth/graph/` -- Workflow graph models, versioning, persistence (backend graph domain)
- `src/zeroth/studio/` -- Emerging Studio authoring control plane (leases, workflows stubs)

### Existing Frontend Reference
- `apps/studio-mockups/` -- Vue 3 + Vite mockup app (reference for initial scaffold patterns)
- `apps/studio-mockups/src/components/StudioEditorMockup.vue` -- Single mockup component

### Infrastructure
- `docker-compose.yml` -- Existing Docker deployment config (extend for multi-container Studio)
- `Dockerfile` -- Existing container build (reference for multi-stage build)

### Planning & Scope
- `.planning/PROJECT.md` -- Project framing, tech stack decisions (Vue 3, Vue Flow, Pinia, dagre, CodeMirror 6)
- `.planning/REQUIREMENTS.md` -- Requirement IDs: CANV-01, CANV-02, CANV-06, CANV-09, CANV-10, API-01, API-03, INFRA-01, INFRA-02
- `.planning/ROADMAP.md` -- Phase 22 scope, success criteria, dependencies
- `.planning/codebase/ARCHITECTURE.md` -- Backend architecture map
- `.planning/codebase/STACK.md` -- Full technology stack reference
- `.planning/codebase/STRUCTURE.md` -- Codebase directory layout

### Prior Phase Context
- `.planning/phases/10-studio-shell-workflow-authoring/10-CONTEXT.md` -- Phase 10 decisions: canvas-first layout, three-panel shell, minimal editor, reusable assets model

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `apps/studio-mockups/` -- Vue 3 + Vite scaffold with basic config (can reference for vite.config.ts patterns)
- `src/zeroth/service/auth.py` -- Auth middleware ready to mount on Studio API routes
- `src/zeroth/graph/` -- Graph models and persistence layer (Studio API wraps these for CRUD)
- `src/zeroth/studio/workflows/` -- Backend workflow authoring stub (extend for graph CRUD)
- `src/zeroth/studio/leases/` -- Authoring lease management (already scaffolded)

### Established Patterns
- FastAPI router pattern: separate `*_api.py` files mounted via `app.include_router()` in `app.py`
- Pydantic models for all API request/response schemas
- SQLAlchemy + Alembic for persistence and migrations
- Docker + Nginx + uvicorn deployment pattern already in place

### Integration Points
- `src/zeroth/service/app.py` -- Mount new `studio_api.py` router here
- `docker-compose.yml` -- Add Nginx container for static file serving
- `apps/studio/vite.config.ts` -- Configure proxy to FastAPI for dev, build output for Nginx

</code_context>

<specifics>
## Specific Ideas

- n8n's editor posture is the design reference (canvas-first, minimal shell) but no code reuse
- 8 node types from day one gives a rich canvas experience for testing workflow modeling
- Multi-container Docker keeps concerns separated (Nginx for static, FastAPI for API)
- openapi-typescript ensures frontend types never drift from backend spec

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 22-canvas-foundation-dev-infrastructure*
*Context gathered: 2026-04-09*
