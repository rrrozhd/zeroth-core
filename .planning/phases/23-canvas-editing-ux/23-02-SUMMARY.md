---
phase: 23-canvas-editing-ux
plan: 02
subsystem: studio-frontend
tags: [palette, drag-and-drop, node-creation, sidebar]
dependency_graph:
  requires: [23-01]
  provides: [node-palette, drag-to-add]
  affects: [WorkflowRail, StudioCanvas]
tech_stack:
  added: []
  patterns: [HTML5-drag-and-drop, composable-pattern, conditional-rendering]
key_files:
  created:
    - apps/studio/src/components/palette/NodePalette.vue
    - apps/studio/src/components/palette/PaletteCategory.vue
    - apps/studio/src/components/palette/PaletteItem.vue
    - apps/studio/src/composables/useDragAndDrop.ts
  modified:
    - apps/studio/src/components/shell/WorkflowRail.vue
    - apps/studio/src/components/canvas/StudioCanvas.vue
decisions:
  - "Palette replaces workflow list conditionally using v-if/v-else based on editor mode AND workflow loaded"
  - "useDragAndDrop composable encapsulates VueFlow screenToFlowCoordinate for correct canvas positioning"
  - "Rail header eyebrow text dynamically switches between NODE PALETTE and WORKFLOWS based on mode"
metrics:
  duration: 137s
  completed: "2026-04-09T21:34:09Z"
  tasks: 2
  files: 6
---

# Phase 23 Plan 02: Node Palette & Drag-to-Add Summary

Categorized node palette sidebar with search filtering and HTML5 drag-to-add using screenToFlowCoordinate for accurate canvas positioning.

## What Was Built

### Task 1: Palette Components (NodePalette, PaletteCategory, PaletteItem)
**Commit:** d7e1273

Three new Vue components in `apps/studio/src/components/palette/`:

- **NodePalette.vue** -- Container with "NODE PALETTE" eyebrow header, search input (`v-model` bound to `searchQuery`), and iteration over `PALETTE_CATEGORIES` rendering `PaletteCategory` for each of the 4 categories (Agents, Logic, Data, Lifecycle). Glassmorphic search input styling consistent with Studio design.

- **PaletteCategory.vue** -- Collapsible category group. Props: `label`, `nodeTypes`, `searchFilter`. Internal `expanded` ref defaults to `true`. Computed `filteredTypes` filters node types by search query against `NODE_TYPE_REGISTRY` labels (case-insensitive). Renders nothing when filtered list is empty. Chevron rotates on collapse.

- **PaletteItem.vue** -- Draggable node type row. Props: `nodeType`, `label`, `icon`. Sets `draggable="true"` with `onDragStart` handler that writes `event.dataTransfer.setData('application/vueflow', nodeType)` and `effectAllowed = 'move'`. Styled as 40px row with grab cursor and hover highlight.

### Task 2: Drag-and-Drop Composable & Wiring
**Commit:** 7d03c66

- **useDragAndDrop.ts** -- Composable providing `onDragOver` (prevents default, sets `dropEffect = 'move'`) and `onDrop` (reads `application/vueflow` data, converts screen coords via `screenToFlowCoordinate`, calls `canvasStore.addNodeWithUndo`).

- **WorkflowRail.vue** -- Updated to import `useUiStore` and `NodePalette`. When `uiStore.currentMode === 'editor'` AND `workflowStore.currentWorkflowId` is set, renders `NodePalette` instead of workflow list. Rail header eyebrow dynamically shows "NODE PALETTE" or "WORKFLOWS". Collapse/expand functionality preserved.

- **StudioCanvas.vue** -- Added `useDragAndDrop` import, destructured `{ onDragOver, onDrop }`, bound `@dragover="onDragOver"` and `@drop="onDrop"` on the `.studio-canvas` wrapper div.

## Deviations from Plan

None -- plan executed exactly as written.

## Pre-existing Issues Noted

The following type errors exist in files NOT modified by this plan (out of scope):
- `CanvasControls.vue`: unused `useAutoLayout` import (TS6133)
- `InspectorField.vue`: unused `props` variable (TS6133)
- `useWorkflowPersistence.ts`: type mismatch on data record (TS2739)
- `canvas.ts`: possible undefined access on history (TS2532)

These are logged for awareness but not fixed (pre-existing, not caused by this plan).

## Known Stubs

None -- all components are fully wired with real data sources (PALETTE_CATEGORIES, NODE_TYPE_REGISTRY, canvas store).

## Verification

- vue-tsc --noEmit: No errors in new/modified files (pre-existing errors in other files only)
- All 3 palette components created with correct props and event handlers
- PaletteItem correctly sets `application/vueflow` dataTransfer
- useDragAndDrop uses `screenToFlowCoordinate` for accurate positioning
- WorkflowRail conditionally renders palette vs workflow list
- StudioCanvas handles drop events

## Self-Check: PASSED
