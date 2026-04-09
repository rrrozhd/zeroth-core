# Phase 22: Canvas Foundation & Dev Infrastructure - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-09
**Phase:** 22-canvas-foundation-dev-infrastructure
**Areas discussed:** Frontend project setup, Initial node types, API surface design, Docker serving

---

## Frontend Project Setup

### Where should the production Studio Vue app live?

| Option | Description | Selected |
|--------|-------------|----------|
| New apps/studio/ | Clean start in a dedicated directory. Keep apps/studio-mockups/ as reference. | ✓ |
| Upgrade apps/studio-mockups/ | Evolve the existing mockup in place. Rename later if needed. | |
| New frontend/ | Top-level frontend/ directory. Simpler path, breaks apps/ convention. | |

**User's choice:** New apps/studio/
**Notes:** Clean separation from the spike mockup.

### How should frontend types stay in sync with the backend OpenAPI spec (INFRA-02)?

| Option | Description | Selected |
|--------|-------------|----------|
| openapi-typescript | Generate TypeScript types from OpenAPI JSON at build time. | ✓ |
| You decide | Claude picks the best approach. | |

**User's choice:** openapi-typescript
**Notes:** None.

### State management approach for the Studio app?

| Option | Description | Selected |
|--------|-------------|----------|
| Pinia stores | Official Vue state management. Separate stores for canvas, workflow, UI state. | ✓ |
| Vue Flow state only | Let Vue Flow manage all graph state, minimal external stores. | |
| You decide | Claude picks based on what works best. | |

**User's choice:** Pinia stores
**Notes:** None.

### CSS/styling approach?

| Option | Description | Selected |
|--------|-------------|----------|
| Tailwind CSS | Utility-first, fast iteration, good Vue ecosystem support. | ✓ |
| Plain CSS / scoped styles | Vue scoped styles, no framework. | |
| You decide | Claude picks based on project needs. | |

**User's choice:** Tailwind CSS
**Notes:** None.

---

## Initial Node Types

### Which Zeroth domain concepts should be available as canvas node types in Phase 22?

| Option | Description | Selected |
|--------|-------------|----------|
| Core set (4 types) | Agent, EU, Approval Gate, Memory Resource. | |
| Minimal set (2 types) | Agent + EU only. | |
| Extended set (6+ types) | Core 4 plus Condition/Branch, Start/End. | ✓ |

**User's choice:** Extended set (6+ types)
**Notes:** None.

### Specific types beyond the core 4?

| Option | Description | Selected |
|--------|-------------|----------|
| Condition + Start/End | Add Condition/Branch, Start, End. Total: 7 types. | |
| Condition + Start/End + Mapping | Above plus Data Mapping node. Total: 8 types. | ✓ |
| You decide | Claude picks based on Zeroth's graph model. | |

**User's choice:** Condition + Start/End + Mapping (8 types total)
**Notes:** None.

### How should typed ports work for edge connections (CANV-02)?

| Option | Description | Selected |
|--------|-------------|----------|
| Input/Output ports | Typed input and output ports. Edges connect output to input. | ✓ |
| Single connection point | One in and one out per node. | |
| You decide | Claude designs the port system. | |

**User's choice:** Input/Output ports
**Notes:** Standard flow graph pattern, matches Vue Flow's handle system.

### Should node configuration be editable in Phase 22?

| Option | Description | Selected |
|--------|-------------|----------|
| Visual only + label | Nodes placed, moved, connected, labeled. Config editing in Phase 23. | ✓ |
| Basic config inline | Allow editing a few key fields directly on the node. | |

**User's choice:** Visual only + label
**Notes:** Phase 22 focuses on canvas foundation, CANV-04 is Phase 23.

---

## API Surface Design

### Where should the graph authoring API live in the backend?

| Option | Description | Selected |
|--------|-------------|----------|
| New studio_api.py router | Dedicated router in src/zeroth/service/studio_api.py, mounted on existing app. | ✓ |
| Separate FastAPI app | Completely separate FastAPI app for Studio. | |
| Extend existing service | Add routes directly to app.py. | |

**User's choice:** New studio_api.py router
**Notes:** Aligns with Phase 10 decision for dedicated Studio/backend authoring layer.

### URL prefix for Studio API endpoints?

| Option | Description | Selected |
|--------|-------------|----------|
| /api/studio/v1/ | Namespaced separately from existing /v1/ deployment API. | ✓ |
| /v1/studio/ | Under existing /v1/ prefix. | |
| You decide | Claude picks URL structure. | |

**User's choice:** /api/studio/v1/
**Notes:** None.

### Should the Studio API require authentication in Phase 22?

| Option | Description | Selected |
|--------|-------------|----------|
| Auth from the start | Reuse existing auth middleware. | ✓ |
| No auth initially | Skip auth, add in Phase 25. | |
| You decide | Claude decides based on existing auth setup. | |

**User's choice:** Auth from the start
**Notes:** RBAC is a downstream requirement (GOV-04), retro-fitting auth is painful.

---

## Docker Serving

### How should Nginx serve the Vue SPA alongside FastAPI in Docker (INFRA-01)?

| Option | Description | Selected |
|--------|-------------|----------|
| Single container, Nginx reverse proxy | Nginx serves static + proxies /api/ to FastAPI. Single Dockerfile. | |
| Multi-container | Separate containers for Nginx and FastAPI. | ✓ |
| You decide | Claude designs Docker serving strategy. | |

**User's choice:** Multi-container
**Notes:** None.

### Dev server strategy during development?

| Option | Description | Selected |
|--------|-------------|----------|
| Vite dev + FastAPI separate | Vite dev server (HMR) and FastAPI separately. Vite proxies /api/. | ✓ |
| Docker-compose for dev too | docker-compose for development as well. | |
| You decide | Claude picks best DX. | |

**User's choice:** Vite dev + FastAPI separate
**Notes:** Standard Vue dev workflow.

---

## Claude's Discretion

- Pinia store boundaries and naming
- Vue Flow configuration details
- Tailwind theme configuration
- Nginx configuration specifics
- openapi-typescript integration approach
- Frontend test framework choice
- File/component structure within apps/studio/src/

## Deferred Ideas

None -- discussion stayed within phase scope.
