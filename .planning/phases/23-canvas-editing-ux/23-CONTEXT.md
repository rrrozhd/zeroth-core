# Phase 23: Canvas Editing UX - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 23 delivers the complete editing experience for the Studio canvas: a categorized node palette sidebar, a property inspector panel, dagre-based auto-layout, undo/redo with command pattern, keyboard shortcuts, and validation indicators on nodes. This builds on Phase 22's foundation (interactive canvas, 8 node types, typed ports, three-panel shell, API CRUD).

This phase does NOT deliver WebSocket real-time updates (Phase 24), workflow execution (Phase 24), governance visualization (Phase 25), or version diffing (Phase 26).

</domain>

<decisions>
## Implementation Decisions

### Palette Sidebar (CANV-03)
- **D-01:** Categorized sidebar palette replaces the current WorkflowRail content when in editing mode. Categories: Agents (Agent, Execution Unit), Logic (Condition/Branch, Approval Gate), Data (Memory Resource, Data Mapping), Lifecycle (Start, End).
- **D-02:** Drag-to-add interaction — user drags a node type from the palette onto the canvas. Consistent with existing Vue Flow drag behavior in StudioCanvas.vue.
- **D-03:** Search/filter input at top of palette for quick node type lookup. Filters across all categories.
- **D-04:** Collapsible categories with node type icons. All expanded by default.

### Inspector Panel (CANV-04)
- **D-05:** Right-side inspector panel — shows properties for the selected node. Collapses/hides when nothing is selected.
- **D-06:** Form fields auto-generated from node type schema in NODE_TYPE_REGISTRY (apps/studio/src/types/nodes.ts). Each node type defines its editable fields.
- **D-07:** Real-time property updates — changes in inspector immediately reflected on the canvas node display.
- **D-08:** Inspector is the third panel in the three-panel layout (rail | canvas | inspector). Uses the existing CanvasArea.vue shell slot.

### Undo/Redo (CANV-07)
- **D-09:** Command pattern implementation in the canvas Pinia store. Each mutation (addNode, removeNode, addEdge, removeEdge, moveNode, updateNodeProperties) produces an undoable command.
- **D-10:** Scope: all canvas mutations are undoable — node add/move/delete, edge add/remove, property changes.
- **D-11:** Standard shortcuts: Ctrl+Z undo, Ctrl+Shift+Z redo. Undo/redo buttons also available in canvas controls (CanvasControls.vue).
- **D-12:** History limit: 50 operations (reasonable for workflow editing, prevents memory bloat).

### Keyboard Shortcuts (CANV-08)
- **D-13:** Standard editor shortcuts: Delete/Backspace (remove selected), Ctrl+A (select all), Ctrl+C (copy), Ctrl+V (paste), Ctrl+D (duplicate).
- **D-14:** No single-key shortcuts — all shortcuts require modifier keys. Prevents accidental operations.
- **D-15:** Shortcut discovery via tooltip or help modal (e.g., "?" key shows shortcut reference).

### Auto-Layout (CANV-05)
- **D-16:** dagre-based DAG layout (already a project dependency from Phase 22). One-click "Auto Layout" button in CanvasControls.vue.
- **D-17:** Layout direction: top-to-bottom (TB) as default, consistent with workflow execution flow direction.
- **D-18:** Layout preserves existing node selection. Animated transition to new positions for visual continuity.

### Validation Indicators (AUTH-06)
- **D-19:** Red border and warning icon on nodes with validation issues (missing required fields, invalid port connections, type mismatches).
- **D-20:** Validation details shown in inspector panel when an invalid node is selected — lists specific issues.
- **D-21:** Real-time validation — runs as user edits properties or modifies connections. Uses existing usePortValidation composable as foundation.

### Claude's Discretion
- Exact palette category icons and visual styling
- Inspector panel field widget types (text input, dropdown, toggle, etc.) per node property type
- Undo/redo stack data structure details
- Animation timing and easing for auto-layout transitions
- Keyboard shortcut help modal design
- Validation rule specifics per node type (which fields required, which connection types valid)
- Whether copy/paste uses clipboard API or internal buffer

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product & UX Direction
- `docs/superpowers/specs/2026-03-29-zeroth-studio-design.md` -- Validated Studio UX decisions (canvas-first, n8n-style editor posture)

### Phase 22 Foundation (must understand before extending)
- `.planning/phases/22-canvas-foundation-dev-infrastructure/22-CONTEXT.md` -- Prior decisions: Vue Flow, 8 node types, Pinia stores, three-panel layout
- `apps/studio/src/stores/canvas.ts` -- Canvas store to extend with undo/redo command pattern
- `apps/studio/src/types/nodes.ts` -- Node type registry to extend with editable field schemas
- `apps/studio/src/composables/usePortValidation.ts` -- Port validation composable to extend for validation indicators
- `apps/studio/src/composables/useCanvasActions.ts` -- Canvas actions composable (integration point for keyboard shortcuts)
- `apps/studio/src/components/canvas/CanvasControls.vue` -- Controls component to extend with undo/redo and auto-layout buttons
- `apps/studio/src/components/canvas/StudioCanvas.vue` -- Main canvas component (drag-to-add target)
- `apps/studio/src/components/shell/WorkflowRail.vue` -- Rail component to extend/replace with palette in editing mode
- `apps/studio/src/components/shell/CanvasArea.vue` -- Shell layout component (inspector panel slot)

### Existing Node Components
- `apps/studio/src/components/nodes/BaseNode.vue` -- Base node component (add validation indicator styling here)
- `apps/studio/src/components/nodes/AgentNode.vue` -- Example node to understand current node rendering

### Planning & Scope
- `.planning/REQUIREMENTS.md` -- Requirement IDs: CANV-03, CANV-04, CANV-05, CANV-07, CANV-08, AUTH-06
- `.planning/ROADMAP.md` -- Phase 23 scope, success criteria, dependencies

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `apps/studio/src/stores/canvas.ts` -- Canvas store with addNode, removeNode, addEdge, removeEdge, clearCanvas. Extend with command pattern for undo/redo.
- `apps/studio/src/composables/usePortValidation.ts` -- Port type validation logic. Extend for validation indicators.
- `apps/studio/src/composables/useCanvasActions.ts` -- Canvas action handlers. Extend for keyboard shortcut bindings.
- `apps/studio/src/components/canvas/CanvasControls.vue` -- Existing zoom/fit controls. Add undo/redo and auto-layout buttons.
- `apps/studio/src/types/nodes.ts` -- NODE_TYPE_REGISTRY with 8 node type definitions. Extend with editable property schemas.
- dagre library already installed as project dependency (used for layout).

### Established Patterns
- Pinia stores with `defineStore` composable pattern
- Vue 3 Composition API with `<script setup>`
- Tailwind CSS for all styling
- Vue Flow `@vue-flow/core` for canvas interactions
- Component-per-node-type pattern (BaseNode + specific node components)

### Integration Points
- WorkflowRail.vue -- Currently shows workflow list. Needs palette mode for editing.
- CanvasArea.vue -- Shell layout. Inspector panel goes in right slot.
- StudioCanvas.vue -- Drop target for palette drag-to-add.
- BaseNode.vue -- Add validation indicator rendering (red border, icon).
- CanvasControls.vue -- Add auto-layout and undo/redo buttons.

</code_context>

<specifics>
## Specific Ideas

- n8n-style editor posture: canvas-first, palette as secondary sidebar (not dominant)
- dagre TB (top-to-bottom) layout matches typical workflow execution flow direction
- Command pattern undo/redo keeps canvas store clean — commands wrap mutations
- Inspector auto-generates fields from node schema — no per-node-type inspector components needed
- Validation runs in real-time using composable pattern, consistent with usePortValidation

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 23-canvas-editing-ux*
*Context gathered: 2026-04-10*
