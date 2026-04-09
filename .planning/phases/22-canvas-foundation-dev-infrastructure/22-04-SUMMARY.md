---
phase: 22-canvas-foundation-dev-infrastructure
plan: "04"
subsystem: studio-frontend
tags: [api-client, persistence, pinia, workflow-management]
dependency_graph:
  requires: [22-01, 22-02]
  provides: [workflow-persistence, api-client-layer, canvas-store]
  affects: [22-03, 22-05]
tech_stack:
  added: []
  patterns: [pinia-composable-store, typed-fetch-wrapper, save-load-orchestration]
key_files:
  created:
    - apps/studio/src/api/client.ts
    - apps/studio/src/api/workflows.ts
    - apps/studio/src/stores/workflow.ts
    - apps/studio/src/stores/canvas.ts
    - apps/studio/src/composables/useWorkflowPersistence.ts
    - apps/studio/src/env.d.ts
  modified:
    - apps/studio/src/components/shell/WorkflowRail.vue
    - apps/studio/src/components/shell/AppHeader.vue
    - apps/studio/src/App.vue
decisions:
  - Used lightweight CanvasNode/CanvasEdge types instead of vue-flow Node/Edge generics to avoid TS2589 deep instantiation errors
  - Passed getViewport as callback to saveWorkflow instead of calling useVueFlow inside composable (avoids requiring Vue Flow context at composable creation time)
metrics:
  duration: 165s
  completed: "2026-04-09T14:06:36Z"
---

# Phase 22 Plan 04: Workflow Persistence & API Wiring Summary

Typed fetch client wired to Studio API with Pinia workflow store, save/load composable, and WorkflowRail showing live workflow data from backend.

## What Was Built

### API Client Layer
- `api/client.ts`: Typed fetch wrapper with `apiFetch<T>()` generic function, `ApiError` class, base URL `/api/studio/v1`, 204 handling
- `api/workflows.ts`: Full CRUD -- `listWorkflows`, `createWorkflow`, `getWorkflow`, `updateWorkflow`, `deleteWorkflow` with typed response interfaces matching backend `studio_schemas.py`

### Pinia Stores
- `stores/workflow.ts`: `useWorkflowStore` with currentWorkflowId, isDirty, isSaving, lastSavedAt state; fetchWorkflows, createNew, loadWorkflow actions with error handling and user-friendly error messages
- `stores/canvas.ts`: `useCanvasStore` with lightweight `CanvasNode`/`CanvasEdge` types, addNode, removeNode, addEdge, clearCanvas actions

### Persistence Composable
- `composables/useWorkflowPersistence.ts`: Orchestrates save/load between canvas store and API; marshals canvas nodes/edges to API format (camelCase to snake_case) on save, unmarshals on load; accepts viewport getter callback for save

### UI Wiring
- `WorkflowRail.vue`: Fetches workflows on mount, renders live list with active indicator, "New Project" button creates via API and opens on canvas, error banner at bottom
- `AppHeader.vue`: Accepts isDirty/isSaving/version props; save indicator shows green dot (saved), amber dot (unsaved), pulsing cyan dot (saving) with dynamic label
- `App.vue`: Deep watches canvas nodes/edges to mark workflow dirty; Ctrl+S / Cmd+S keyboard shortcut triggers save; passes save state to header

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Created canvas store (stores/canvas.ts)**
- **Found during:** Task 1
- **Issue:** Plan references `useCanvasStore` from `stores/canvas.ts` but this file did not exist yet
- **Fix:** Created canvas store with lightweight CanvasNode/CanvasEdge types
- **Files created:** apps/studio/src/stores/canvas.ts
- **Commit:** 4fe3b44

**2. [Rule 1 - Bug] Avoided TS2589 deep type instantiation**
- **Found during:** Task 1 verification
- **Issue:** Using `Node`/`Edge` types from `@vue-flow/core` in Pinia ref caused TypeScript "excessively deep" error
- **Fix:** Defined lightweight `CanvasNode`/`CanvasEdge` interfaces with index signatures instead
- **Files modified:** apps/studio/src/stores/canvas.ts, apps/studio/src/composables/useWorkflowPersistence.ts

**3. [Rule 3 - Blocking] Created Vue SFC type declarations (env.d.ts)**
- **Found during:** Task 1 verification
- **Issue:** TypeScript could not find module for .vue files (TS2307)
- **Fix:** Added src/env.d.ts with `*.vue` module declaration
- **Files created:** apps/studio/src/env.d.ts

**4. [Rule 2 - Design] Changed saveWorkflow to accept viewport getter callback**
- **Found during:** Task 1
- **Issue:** `useVueFlow()` requires Vue Flow context that may not exist when composable is created outside VueFlow component tree
- **Fix:** `saveWorkflow` accepts a `getViewport` function parameter instead of calling `useVueFlow` internally
- **Files modified:** apps/studio/src/composables/useWorkflowPersistence.ts

## Decisions Made

1. **Lightweight canvas types over vue-flow generics** -- Vue Flow's `Node<T>` generic creates deep recursive types that TypeScript cannot resolve in a `ref()`. Using plain interfaces with index signatures avoids this while remaining compatible with Vue Flow at the component level.

2. **Viewport getter as callback** -- Rather than coupling the persistence composable to Vue Flow context, the save function accepts a viewport getter. This allows the canvas component (which has Vue Flow context) to provide the viewport at save time.

## Verification

- `vue-tsc --noEmit` passes with zero errors
- `npm run build` produces production bundle (74.81 kB JS, 13.44 kB CSS)
- All acceptance criteria from plan met

## Known Stubs

None -- all data flows are wired to real API endpoints. The viewport getter in App.vue returns a default `{x:0, y:0, zoom:1}` which will be replaced when the Vue Flow canvas component is implemented in plan 22-03.
