# Phase 23: Canvas Editing UX - Research

**Researched:** 2026-04-09
**Domain:** Vue Flow canvas editing (palette, inspector, undo/redo, layout, validation)
**Confidence:** HIGH

## Summary

Phase 23 extends the Phase 22 canvas foundation with six editing features: a categorized drag-and-drop node palette in the left rail, a property inspector panel on the right, dagre-based auto-layout, undo/redo via command pattern in the Pinia canvas store, keyboard shortcuts, and validation indicators on nodes. All features build on existing code -- the canvas store, node type registry, port validation composable, and three-panel shell layout.

The implementation requires no new library dependencies. Vue Flow provides built-in drag-and-drop support via `screenToFlowCoordinate()` and `addNodes()`. The `@dagrejs/dagre` package is already installed for auto-layout. Undo/redo is best implemented as a custom command pattern wrapping the existing canvas store mutations, keeping the store clean and testable. The inspector panel auto-generates form fields from an extended `NodeTypeDefinition` schema rather than requiring per-node-type inspector components.

**Primary recommendation:** Implement the command pattern in the canvas store first (it underpins all other features), then palette + inspector in parallel, then auto-layout and keyboard shortcuts, and finally validation indicators.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Categorized sidebar palette replaces WorkflowRail content in editing mode. Categories: Agents (Agent, Execution Unit), Logic (Condition/Branch, Approval Gate), Data (Memory Resource, Data Mapping), Lifecycle (Start, End).
- D-02: Drag-to-add interaction from palette onto canvas.
- D-03: Search/filter input at top of palette for quick node type lookup.
- D-04: Collapsible categories with node type icons. All expanded by default.
- D-05: Right-side inspector panel shows properties for selected node. Collapses when nothing selected.
- D-06: Form fields auto-generated from node type schema in NODE_TYPE_REGISTRY.
- D-07: Real-time property updates -- inspector changes immediately reflected on canvas.
- D-08: Inspector is third panel in three-panel layout using existing CanvasArea.vue shell slot.
- D-09: Command pattern in canvas Pinia store. Each mutation produces an undoable command.
- D-10: All canvas mutations undoable (add/move/delete node, add/remove edge, property changes).
- D-11: Ctrl+Z undo, Ctrl+Shift+Z redo. Buttons in CanvasControls.vue.
- D-12: History limit: 50 operations.
- D-13: Standard shortcuts: Delete/Backspace, Ctrl+A, Ctrl+C, Ctrl+V, Ctrl+D.
- D-14: No single-key shortcuts -- all require modifier keys (except Delete/Backspace).
- D-15: Shortcut discovery via tooltip or help modal ("?" key shows reference).
- D-16: dagre-based DAG layout. One-click "Auto Layout" button in CanvasControls.vue.
- D-17: Layout direction: top-to-bottom (TB) default.
- D-18: Layout preserves selection. Animated transition to new positions.
- D-19: Red border and warning icon on invalid nodes.
- D-20: Validation details in inspector when invalid node selected.
- D-21: Real-time validation using usePortValidation as foundation.

### Claude's Discretion
- Palette category icons and visual styling
- Inspector field widget types per property type
- Undo/redo stack data structure details
- Animation timing/easing for auto-layout
- Keyboard shortcut help modal design
- Validation rule specifics per node type
- Copy/paste: clipboard API vs internal buffer

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CANV-03 | Browse/search node types in categorized sidebar palette | Drag-and-drop pattern from Vue Flow official example; NODE_TYPE_REGISTRY category mapping |
| CANV-04 | View/edit selected node properties in inspector panel | Schema-driven form generation from extended NodeTypeDefinition; reactive Pinia binding |
| CANV-05 | Auto-layout graph in readable DAG arrangement | dagre graphlib with TB direction; position offset correction for Vue Flow top-left anchor |
| CANV-07 | Undo/redo canvas operations | Command pattern wrapping canvas store mutations; 50-item history ring buffer |
| CANV-08 | Keyboard shortcuts for common operations | Window keydown listener pattern (already used for Ctrl+S in App.vue); modifier key requirement |
| AUTH-06 | Validation indicators on nodes | Extended usePortValidation + schema-based field validation; reactive computed validation state |
</phase_requirements>

## Standard Stack

### Core (already installed)
| Library | Version (package.json) | Purpose | Why Standard |
|---------|----------------------|---------|--------------|
| @vue-flow/core | ^1.48 | Canvas rendering, drag-drop, node selection | Foundation from Phase 22 |
| @dagrejs/dagre | ^3.0 | Directed graph auto-layout | Already a dependency; standard for DAG layout |
| pinia | ^3.0 | State management (canvas store, ui store) | Already in use; command pattern extends existing store |
| vue | ^3.5 | Composition API, reactivity | Project framework |
| tailwindcss | ^4.2 | Styling | Project standard |

### Supporting (no new dependencies needed)
No new npm packages required. All features are implementable with existing dependencies.

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Custom command pattern | pinia-undo plugin | Plugin uses snapshot-based approach (stores full state on each change), not suitable for fine-grained canvas ops; command pattern gives explicit undo semantics |
| Custom dagre wrapper | @vue-flow/layout | Not an official package; custom wrapper is 30 lines and gives full control |
| Internal copy buffer | Clipboard API | Clipboard API requires HTTPS and user permission; internal buffer is simpler and sufficient for same-session copy/paste |

## Architecture Patterns

### Recommended Project Structure
```
apps/studio/src/
  stores/
    canvas.ts              # Extended with command pattern (undo/redo)
  composables/
    useDragAndDrop.ts      # NEW: drag-to-add from palette to canvas
    useCanvasActions.ts    # Extended with keyboard shortcut handlers
    usePortValidation.ts   # Extended with schema-based field validation
    useAutoLayout.ts       # NEW: dagre layout composable
    useKeyboardShortcuts.ts # NEW: keyboard shortcut bindings
    useNodeValidation.ts   # NEW: per-node validation state
  components/
    palette/
      NodePalette.vue      # NEW: categorized palette sidebar
      PaletteCategory.vue  # NEW: collapsible category group
      PaletteItem.vue      # NEW: draggable node type item
    inspector/
      NodeInspector.vue    # NEW: property inspector panel
      InspectorField.vue   # NEW: auto-generated form field
    canvas/
      CanvasControls.vue   # Extended with undo/redo + auto-layout buttons
      StudioCanvas.vue     # Extended with drop handlers
    nodes/
      BaseNode.vue         # Extended with validation indicator styling
    shell/
      WorkflowRail.vue     # Extended with palette mode toggle
      CanvasArea.vue       # Extended with inspector panel slot
  types/
    nodes.ts               # Extended with editable property schemas
```

### Pattern 1: Command Pattern for Undo/Redo
**What:** Each canvas mutation is wrapped in a command object with `execute()` and `undo()` methods. The canvas store maintains a history stack (max 50) and pointer.
**When to use:** All canvas mutations (addNode, removeNode, addEdge, removeEdge, moveNode, updateNodeProperties).
**Example:**
```typescript
// Source: Standard command pattern adapted for Pinia
interface CanvasCommand {
  execute(): void
  undo(): void
  description: string
}

// Inside canvas store
const history = ref<CanvasCommand[]>([])
const historyIndex = ref(-1)
const MAX_HISTORY = 50

function executeCommand(command: CanvasCommand) {
  // Truncate any redo history
  history.value = history.value.slice(0, historyIndex.value + 1)
  // Execute and push
  command.execute()
  history.value.push(command)
  // Enforce limit
  if (history.value.length > MAX_HISTORY) {
    history.value.shift()
  } else {
    historyIndex.value++
  }
}

function undo() {
  if (historyIndex.value < 0) return
  history.value[historyIndex.value].undo()
  historyIndex.value--
}

function redo() {
  if (historyIndex.value >= history.value.length - 1) return
  historyIndex.value++
  history.value[historyIndex.value].execute()
}

// Example command factory
function createAddNodeCommand(type: string, position: XYPosition): CanvasCommand {
  let nodeId: string
  return {
    description: `Add ${type} node`,
    execute() {
      // Store created ID for undo
      const node = createNode(type, position)
      nodeId = node.id
      nodes.value.push(node)
    },
    undo() {
      nodes.value = nodes.value.filter(n => n.id !== nodeId)
      // Also remove any connected edges
      edges.value = edges.value.filter(e => e.source !== nodeId && e.target !== nodeId)
    },
  }
}
```

### Pattern 2: Vue Flow Drag and Drop
**What:** Palette items set dataTransfer data on dragstart; canvas container handles dragover/drop events; `screenToFlowCoordinate()` converts screen position to flow coordinates.
**When to use:** Dragging node types from palette onto canvas.
**Example:**
```typescript
// Source: Vue Flow official drag-and-drop example
// In palette item:
function onDragStart(event: DragEvent, nodeType: string) {
  if (event.dataTransfer) {
    event.dataTransfer.setData('application/vueflow', nodeType)
    event.dataTransfer.effectAllowed = 'move'
  }
}

// In StudioCanvas.vue:
const { screenToFlowCoordinate, addNodes } = useVueFlow()

function onDragOver(event: DragEvent) {
  event.preventDefault()
  if (event.dataTransfer) {
    event.dataTransfer.dropEffect = 'move'
  }
}

function onDrop(event: DragEvent) {
  const type = event.dataTransfer?.getData('application/vueflow')
  if (!type) return
  const position = screenToFlowCoordinate({
    x: event.clientX,
    y: event.clientY,
  })
  // Use canvas store's command-based addNode
  canvasStore.addNodeWithUndo(type, position)
}
```

### Pattern 3: Dagre Auto-Layout
**What:** Creates a dagre graph from current nodes/edges, runs layout algorithm, applies calculated positions to Vue Flow nodes with animation.
**When to use:** One-click "Auto Layout" button in CanvasControls.
**Example:**
```typescript
// Source: dagre + Vue Flow layout pattern
import dagre from '@dagrejs/dagre'

const NODE_WIDTH = 160  // matches BaseNode 10rem
const NODE_HEIGHT = 100 // approximate node height

function getLayoutedPositions(
  nodes: CanvasNode[],
  edges: CanvasEdge[],
  direction: 'TB' | 'LR' = 'TB'
): Map<string, XYPosition> {
  const g = new dagre.graphlib.Graph().setDefaultEdgeLabel(() => ({}))
  g.setGraph({ rankdir: direction, nodesep: 60, ranksep: 80 })

  nodes.forEach(node => {
    g.setNode(node.id, { width: NODE_WIDTH, height: NODE_HEIGHT })
  })
  edges.forEach(edge => {
    g.setEdge(edge.source, edge.target)
  })

  dagre.layout(g)

  const positions = new Map<string, XYPosition>()
  nodes.forEach(node => {
    const pos = g.node(node.id)
    // dagre returns center coordinates; Vue Flow uses top-left
    positions.set(node.id, {
      x: pos.x - NODE_WIDTH / 2,
      y: pos.y - NODE_HEIGHT / 2,
    })
  })
  return positions
}
```

### Pattern 4: Schema-Driven Inspector
**What:** Extend NodeTypeDefinition with a `properties` array defining editable fields. Inspector reads schema and renders appropriate form widgets.
**When to use:** Auto-generating inspector forms without per-node-type components.
**Example:**
```typescript
// Extended NodeTypeDefinition in types/nodes.ts
export type PropertyFieldType = 'text' | 'textarea' | 'number' | 'select' | 'toggle'

export interface PropertyDefinition {
  key: string
  label: string
  type: PropertyFieldType
  required?: boolean
  default?: unknown
  options?: { label: string; value: string }[]  // for select type
  placeholder?: string
}

export interface NodeTypeDefinition {
  type: string
  label: string
  category: 'core' | 'flow' | 'data'
  ports: PortDefinition[]
  icon: string
  properties?: PropertyDefinition[]  // NEW
}

// Example: agent node with editable properties
agent: {
  type: 'agent',
  label: 'Agent',
  category: 'core',
  icon: 'sparkle',
  ports: [...],
  properties: [
    { key: 'name', label: 'Name', type: 'text', required: true, placeholder: 'Agent name' },
    { key: 'description', label: 'Description', type: 'textarea' },
    { key: 'model', label: 'Model', type: 'select', options: [
      { label: 'GPT-4o', value: 'gpt-4o' },
      { label: 'Claude 3.5', value: 'claude-3.5-sonnet' },
    ]},
    { key: 'temperature', label: 'Temperature', type: 'number', default: 0.7 },
  ],
}
```

### Anti-Patterns to Avoid
- **Snapshot-based undo:** Storing full canvas state on each operation wastes memory and makes 50-item history expensive. Use command pattern instead.
- **Per-node-type inspector components:** Creating a separate inspector for each of 8 node types creates maintenance burden. Schema-driven generation is the correct approach.
- **Global keydown without cleanup:** Always use `onMounted`/`onUnmounted` lifecycle hooks for keyboard event listeners to prevent memory leaks.
- **Mutating node positions directly during animation:** Use Vue's transition system or requestAnimationFrame to animate position changes -- do not mutate the store in a tight loop.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Graph layout algorithm | Custom node positioning | `@dagrejs/dagre` (already installed) | Graph layout is a solved problem; dagre handles rank assignment, crossing minimization |
| Coordinate conversion | Manual zoom/pan offset math | `useVueFlow().screenToFlowCoordinate()` | Vue Flow accounts for all transform state internally |
| Connection validation | Custom port compatibility checks | Extend existing `usePortValidation.ts` | Already handles port type matching; just needs schema field validation added |
| Drag-and-drop plumbing | Custom mouse event tracking | HTML5 Drag and Drop API via `dataTransfer` | Standard browser API; Vue Flow examples demonstrate this exact pattern |

**Key insight:** Every feature in this phase extends existing code rather than introducing new dependencies. The architecture is already in place from Phase 22.

## Common Pitfalls

### Pitfall 1: Undo/Redo with Edge Cascading
**What goes wrong:** Removing a node must also remove connected edges, but undo must restore both node AND edges. If edge removal isn't captured in the undo command, redo/undo cycles leave orphaned edges or lose edges permanently.
**Why it happens:** Node deletion is a compound operation.
**How to avoid:** The `removeNode` command must snapshot connected edges before deletion, and restore them on undo.
**Warning signs:** After undo of node deletion, edges to/from the restored node are missing.

### Pitfall 2: Drag-Drop Coordinate Offset
**What goes wrong:** Nodes dropped from palette appear at wrong position (offset by toolbar height, sidebar width, or zoom level).
**Why it happens:** Using `event.clientX/Y` without converting through Vue Flow's coordinate system.
**How to avoid:** Always use `screenToFlowCoordinate({ x: event.clientX, y: event.clientY })` -- it handles all transforms including zoom, pan, and container offset.
**Warning signs:** Nodes appear shifted from drop point, especially when zoomed in/out.

### Pitfall 3: Inspector Reactivity Breaking Store
**What goes wrong:** Two-way binding in inspector form fields directly mutates node data in the store, bypassing the command pattern and breaking undo.
**Why it happens:** v-model on store properties creates direct mutations.
**How to avoid:** Inspector edits should go through a command: collect changes, on blur/commit create an `updateNodeProperties` command, execute it through the command system.
**Warning signs:** Property edits can't be undone; undo stack doesn't grow when editing inspector fields.

### Pitfall 4: Keyboard Shortcuts Firing in Text Inputs
**What goes wrong:** Delete key fires while user is typing in inspector text field, accidentally deleting the selected node.
**Why it happens:** Global keydown listener doesn't check `event.target`.
**How to avoid:** Check if `event.target` is an input/textarea/contenteditable element and skip shortcut handling if so.
**Warning signs:** Pressing Delete or Backspace in inspector form deletes the selected node.

### Pitfall 5: Layout Animation Conflicting with Vue Flow State
**What goes wrong:** Animated position transitions cause Vue Flow to emit change events on each frame, triggering dirty state, autosave, or undo history entries.
**Why it happens:** Vue Flow reactively watches node positions.
**How to avoid:** Set a `isLayouting` flag during animation, skip dirty-marking and command creation while it's true. Apply final positions as a single batch command.
**Warning signs:** Auto-layout creates 50+ undo entries (one per animation frame), or triggers save on every frame.

### Pitfall 6: Dagre Center vs Top-Left Anchor
**What goes wrong:** After auto-layout, nodes appear offset from their intended positions.
**Why it happens:** Dagre returns center coordinates; Vue Flow positions nodes from top-left corner.
**How to avoid:** Subtract half node width/height from dagre results: `x: pos.x - NODE_WIDTH / 2, y: pos.y - NODE_HEIGHT / 2`.
**Warning signs:** Nodes overlap or have uneven spacing after layout.

## Code Examples

### Keyboard Shortcut Handler with Input Guard
```typescript
// Source: Standard pattern for canvas keyboard shortcuts
function useKeyboardShortcuts(canvasStore: ReturnType<typeof useCanvasStore>) {
  function isInputElement(target: EventTarget | null): boolean {
    if (!target || !(target instanceof HTMLElement)) return false
    const tag = target.tagName.toLowerCase()
    return tag === 'input' || tag === 'textarea' || target.isContentEditable
  }

  function handleKeydown(e: KeyboardEvent) {
    if (isInputElement(e.target)) return

    const isMod = e.ctrlKey || e.metaKey

    // Delete selected
    if (e.key === 'Delete' || e.key === 'Backspace') {
      e.preventDefault()
      canvasStore.deleteSelected()
    }
    // Undo
    else if (isMod && e.key === 'z' && !e.shiftKey) {
      e.preventDefault()
      canvasStore.undo()
    }
    // Redo
    else if (isMod && e.key === 'z' && e.shiftKey) {
      e.preventDefault()
      canvasStore.redo()
    }
    // Select all
    else if (isMod && e.key === 'a') {
      e.preventDefault()
      canvasStore.selectAll()
    }
    // Copy
    else if (isMod && e.key === 'c') {
      e.preventDefault()
      canvasStore.copySelected()
    }
    // Paste
    else if (isMod && e.key === 'v') {
      e.preventDefault()
      canvasStore.pasteClipboard()
    }
    // Duplicate
    else if (isMod && e.key === 'd') {
      e.preventDefault()
      canvasStore.duplicateSelected()
    }
  }

  onMounted(() => window.addEventListener('keydown', handleKeydown))
  onUnmounted(() => window.removeEventListener('keydown', handleKeydown))
}
```

### Validation Composable
```typescript
// Source: Extension of existing usePortValidation pattern
import { computed } from 'vue'
import type { CanvasNode, CanvasEdge } from '../stores/canvas'
import type { NodeTypeDefinition } from '../types/nodes'
import { NODE_TYPE_REGISTRY } from '../types/nodes'

export interface ValidationIssue {
  type: 'missing_field' | 'invalid_connection' | 'type_mismatch'
  message: string
  field?: string
}

export function useNodeValidation(
  nodes: () => CanvasNode[],
  edges: () => CanvasEdge[]
) {
  const validationMap = computed(() => {
    const map = new Map<string, ValidationIssue[]>()
    for (const node of nodes()) {
      const issues: ValidationIssue[] = []
      const typeDef = NODE_TYPE_REGISTRY[node.type]
      if (!typeDef) continue

      // Check required properties
      if (typeDef.properties) {
        for (const prop of typeDef.properties) {
          if (prop.required && !node.data[prop.key]) {
            issues.push({
              type: 'missing_field',
              message: `${prop.label} is required`,
              field: prop.key,
            })
          }
        }
      }

      // Check minimum connections (e.g., start must have output)
      const nodeEdges = edges().filter(
        e => e.source === node.id || e.target === node.id
      )
      const inputPorts = typeDef.ports.filter(p => p.direction === 'input')
      const outputPorts = typeDef.ports.filter(p => p.direction === 'output')
      
      if (inputPorts.length > 0 && !nodeEdges.some(e => e.target === node.id)) {
        issues.push({
          type: 'invalid_connection',
          message: 'Missing input connection',
        })
      }
      if (outputPorts.length > 0 && !nodeEdges.some(e => e.source === node.id)) {
        issues.push({
          type: 'invalid_connection',
          message: 'Missing output connection',
        })
      }

      if (issues.length > 0) {
        map.set(node.id, issues)
      }
    }
    return map
  })

  function getIssues(nodeId: string): ValidationIssue[] {
    return validationMap.value.get(nodeId) ?? []
  }

  function isValid(nodeId: string): boolean {
    return !validationMap.value.has(nodeId)
  }

  return { validationMap, getIssues, isValid }
}
```

### Inspector Panel Rendering Logic
```typescript
// Inspector auto-generates fields from schema
// In NodeInspector.vue <script setup>
const canvasStore = useCanvasStore()
const uiStore = useUiStore()

const selectedNode = computed(() =>
  canvasStore.nodes.find(n => n.id === uiStore.selectedNodeId)
)

const typeDef = computed(() =>
  selectedNode.value ? NODE_TYPE_REGISTRY[selectedNode.value.type] : null
)

const properties = computed(() => typeDef.value?.properties ?? [])

function updateProperty(key: string, value: unknown) {
  if (!selectedNode.value) return
  // Goes through command pattern for undo support
  canvasStore.updateNodePropertyWithUndo(selectedNode.value.id, key, value)
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `project()` for coordinate conversion | `screenToFlowCoordinate()` | Vue Flow 1.x | Clearer API name; same underlying function |
| dagre (original) | @dagrejs/dagre | 2024 | Moved to @dagrejs org scope; same API |
| Pinia v2 options API stores | Pinia v3 setup stores | 2025 | Project already uses setup store pattern |

**Deprecated/outdated:**
- `project()` in useVueFlow: Still works but `screenToFlowCoordinate()` is the preferred name
- dagre npm package (unscoped): Use `@dagrejs/dagre` instead

## Open Questions

1. **Category mapping alignment**
   - What we know: CONTEXT.md defines categories as Agents/Logic/Data/Lifecycle. Current NODE_TYPE_REGISTRY uses core/flow/data.
   - What's unclear: Whether to change the registry's `category` field or add a separate palette-category mapping.
   - Recommendation: Add a `paletteCategory` mapping constant rather than changing the existing `category` field, since `category` may be used elsewhere. Planner should decide.

2. **Node move tracking for undo**
   - What we know: Vue Flow internally handles node dragging. The store receives updated positions reactively.
   - What's unclear: How to intercept drag-start/drag-end to create a single move command (not one per pixel).
   - Recommendation: Use Vue Flow's `@node-drag-start` and `@node-drag-stop` events. Capture initial position on start, create command on stop with before/after positions.

3. **Copy/paste serialization**
   - What we know: D-13 requires copy/paste. Nodes have data objects and connections.
   - What's unclear: Whether to copy connected edges between copied nodes, and how to handle position offsets for pasted nodes.
   - Recommendation: Copy selected nodes + edges between them. Paste with 24px offset (matching snap grid). Use internal buffer (simpler than Clipboard API).

## Project Constraints (from CLAUDE.md)

- Build: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/` (backend only -- Studio uses `npm run build` and `vitest`)
- Testing: vitest for Studio frontend tests
- Progress logging: Must use `progress-logger` skill for every implementation session
- Layout: `src/zeroth/` for backend, `apps/studio/` for frontend
- Styling: Tailwind CSS (v4.2)
- Components: Vue 3 Composition API with `<script setup>`
- State: Pinia stores with `defineStore` composable pattern

## Sources

### Primary (HIGH confidence)
- Vue Flow official drag-and-drop documentation: [vueflow.dev/examples/dnd.html](https://vueflow.dev/examples/dnd.html) - DnD pattern with screenToFlowCoordinate
- Vue Flow official layout examples: [vueflow.dev/examples/layout/simple.html](https://vueflow.dev/examples/layout/simple.html) - dagre integration
- Existing codebase: `apps/studio/src/stores/canvas.ts`, `apps/studio/src/types/nodes.ts`, `apps/studio/src/composables/usePortValidation.ts` - current implementation

### Secondary (MEDIUM confidence)
- React Flow dagre example (same dagre API, different framework): [reactflow.dev/examples/layout/dagre](https://reactflow.dev/examples/layout/dagre) - getLayoutedElements pattern
- Vue Flow drag-and-drop guide: [mintlify.com/bcakmakoglu/vue-flow/examples/drag-and-drop](https://www.mintlify.com/bcakmakoglu/vue-flow/examples/drag-and-drop) - complete DnD code

### Tertiary (LOW confidence)
- pinia-undo plugin evaluation: [github.com/wobsoriano/pinia-undo](https://github.com/wobsoriano/pinia-undo) - considered and rejected in favor of custom command pattern

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries already installed, versions confirmed in package.json
- Architecture: HIGH - patterns verified against Vue Flow official docs and existing codebase
- Pitfalls: HIGH - based on known Vue Flow behavior and command pattern fundamentals
- Code examples: MEDIUM - adapted from official examples and standard patterns, not tested against this specific codebase

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable -- no library upgrades expected)
