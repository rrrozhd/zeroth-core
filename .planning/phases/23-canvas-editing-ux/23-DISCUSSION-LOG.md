# Phase 23: Canvas Editing UX - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-10
**Phase:** 23-canvas-editing-ux
**Areas discussed:** Palette Design, Inspector Panel, Undo/Redo Scope, Keyboard Shortcuts, Validation Feedback
**Mode:** auto (all decisions auto-selected with recommended defaults)

---

## Palette Design

| Option | Description | Selected |
|--------|-------------|----------|
| Categorized sidebar with search | Categories: Agents, Logic, Data, Lifecycle. Drag-to-add. Search filter. | ✓ |
| Floating palette popup | Modal or popover triggered by button click | |
| Toolbar strip | Horizontal node type bar above canvas | |

**User's choice:** Categorized sidebar with search (auto-selected)
**Notes:** Consistent with n8n-style editor posture from Phase 22 decisions. Reuses WorkflowRail slot.

---

## Inspector Panel

| Option | Description | Selected |
|--------|-------------|----------|
| Right-side inspector with auto-generated forms | Schema-driven fields from NODE_TYPE_REGISTRY | ✓ |
| Inline node editing | Edit properties directly on node cards | |
| Modal property editor | Open dialog for each node | |

**User's choice:** Right-side inspector with auto-generated forms (auto-selected)
**Notes:** Fits three-panel layout from Phase 22. Auto-generation from schema avoids per-node-type inspector components.

---

## Undo/Redo Scope

| Option | Description | Selected |
|--------|-------------|----------|
| All canvas mutations | Node CRUD, edge CRUD, moves, property changes. Command pattern. | ✓ |
| Structural only | Only node/edge add/remove, not moves or property changes | |

**User's choice:** All canvas mutations (auto-selected)
**Notes:** Command pattern in Pinia store. 50-operation history limit.

---

## Keyboard Shortcuts

| Option | Description | Selected |
|--------|-------------|----------|
| Standard editor shortcuts | Delete, Ctrl+A/C/V/D. All require modifier keys. | ✓ |
| Minimal shortcuts | Delete only, no copy/paste/duplicate | |
| Extended shortcuts | Include single-key shortcuts (N for new node, etc.) | |

**User's choice:** Standard editor shortcuts (auto-selected)
**Notes:** No single-key shortcuts to prevent accidental operations.

---

## Validation Feedback

| Option | Description | Selected |
|--------|-------------|----------|
| Inline node indicators + inspector details | Red border/icon on invalid nodes, details in inspector | ✓ |
| Toast notifications | Show validation errors as dismissible toasts | |
| Validation panel | Dedicated panel listing all validation issues | |

**User's choice:** Inline node indicators + inspector details (auto-selected)
**Notes:** Extends usePortValidation composable. Real-time validation as user edits.

---

## Claude's Discretion

- Palette category icons and visual styling
- Inspector field widget types per property type
- Undo/redo stack implementation details
- Auto-layout animation timing
- Keyboard shortcut help modal design
- Per-node-type validation rules
- Copy/paste clipboard strategy

## Deferred Ideas

None
