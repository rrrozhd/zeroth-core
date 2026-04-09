# Pitfalls Research

**Domain:** Visual workflow editor (Zeroth Studio) added to existing governed multi-agent platform
**Researched:** 2026-04-09
**Confidence:** HIGH (verified across Vue Flow docs, React Flow perf guides, n8n license docs, graph editor UX research)

---

## Critical Pitfalls

### Pitfall 1: Accidental n8n Pattern Copying Under SUL License

**What goes wrong:**
Developers study n8n's open source code as a "design reference" and inadvertently reproduce substantial portions of its UI structure, component hierarchy, CSS patterns, or interaction logic. The Sustainable Use License (SUL) is not OSI-approved open source -- it explicitly prohibits white-labeling, embedding in SaaS, and redistribution. Even if the final code is "rewritten," if the structure is demonstrably derived from n8n source, it creates legal exposure during audits, fundraising, or acquisition due diligence.

**Why it happens:**
n8n's source is browsable on GitHub, making it tempting to copy-reference specific components. The line between "inspired by" and "derived from" is blurry in practice. Developers who read n8n source code may unconsciously replicate its architecture even when writing "from scratch."

**How to avoid:**
- Treat n8n as a UX reference (screenshots, user flows, interaction videos) not a code reference. Never open n8n source files during implementation.
- Document design decisions with independent rationale. Every component should trace to Zeroth's governance domain model, not to "n8n does it this way."
- Use MIT-licensed libraries directly (Vue Flow, dagre, CodeMirror 6) -- these are safe.
- Maintain clean-room discipline: designers sketch UX from requirements, developers implement from those sketches, nobody references n8n TypeScript during coding.

**Warning signs:**
- PRs that reference n8n file paths or component names
- Component hierarchy that mirrors n8n's package structure (e.g., `editor-ui/src/components/canvas/`)
- CSS class names or variable names suspiciously similar to n8n's

**Phase to address:**
Phase 1 (Foundation) -- establish clean-room policy before any canvas code is written.

---

### Pitfall 2: Vue Flow Container Sizing and Initialization Failures

**What goes wrong:**
The Vue Flow canvas renders as a blank white rectangle or fails to display nodes. This is the single most common Vue Flow issue per official troubleshooting docs: the parent container must have explicit width and height CSS properties. Additionally, nodes appear but are positioned at (0,0) in a pile because auto-layout was not applied after initial data load.

**Why it happens:**
Vue Flow uses an SVG/canvas rendering approach that requires explicit dimensions -- it cannot infer size from flex layouts or percentage-based parents without explicit constraints. Developers assume it will fill available space like a regular div. The dagre auto-layout pass must be triggered explicitly after nodes are added; it is not automatic.

**How to avoid:**
- Set the Vue Flow container to `width: 100%; height: 100%` and ensure ALL ancestor elements up to the root also have explicit dimensions (common CSS: `html, body, #app { height: 100%; margin: 0; }`).
- Run dagre layout computation after every graph data load, then call `fitView()` to center the result.
- Create a reusable `useGraphLayout` composable that handles the dagre computation + fitView sequence so it is never forgotten.

**Warning signs:**
- Blank canvas area with no console errors (the container has 0 height)
- All nodes stacked at top-left corner
- Graph looks correct on one screen size but breaks on resize

**Phase to address:**
Phase 1 (Foundation) -- the very first canvas milestone must nail container setup with responsive resize handling.

---

### Pitfall 3: Over-Engineering Before Validating Core Canvas Experience

**What goes wrong:**
The team builds real-time collaboration, advanced undo/redo with operational transforms, plugin systems, or complex inspector panels before verifying that the basic canvas interaction -- drag a node, draw an edge, configure a property, save, reload -- works well and feels good. The result is months of infrastructure work on features nobody has validated, while the core editing loop remains buggy or awkward.

**Why it happens:**
Graph editors are technically fascinating. Engineers gravitate toward hard problems (CRDTs, conflict resolution, plugin architectures) instead of the mundane but critical UX work of making node placement, edge drawing, and property editing feel smooth. The existing Zeroth backend is sophisticated, which creates pressure to match that sophistication in the frontend immediately.

**How to avoid:**
- Phase 1 deliverable is a single-user canvas that can: place nodes, draw edges, configure node properties in an inspector, save to backend, reload from backend. Nothing else.
- No WebSocket sync, no collaboration, no undo/redo beyond browser-native in Phase 1.
- Validate with real users (or internal team) before building any "advanced" features.
- Apply the "demo to stakeholders" test: if you cannot demo a satisfying editing flow in Phase 1, no amount of infrastructure will save the product.

**Warning signs:**
- Sprint planning includes "real-time sync" or "collaboration" before basic save/load works
- More time spent on WebSocket infrastructure than on node interaction polish
- Backend API design dominates discussions while canvas UX gets handwaved

**Phase to address:**
Phase 1 (Foundation) -- explicitly scope to single-user, zero-collaboration, save/load only.

---

### Pitfall 4: Performance Collapse with Large Graphs (50+ Nodes)

**What goes wrong:**
The editor becomes sluggish or unusable when workflows exceed ~50 nodes. Node dragging lags, edge re-routing stutters, and the inspector panel updates with visible delay. This happens even though Vue Flow has built-in viewport virtualization, because custom node components, edge styles, and governance decorators (approval badges, audit indicators) add rendering overhead that compounds.

**Why it happens:**
Every custom node component re-renders when the nodes array changes (a common pitfall documented in React Flow's performance guide, which shares the same architecture). Complex CSS (shadows, gradients, animations on governance badges) multiplies per-node rendering cost. Accessing the full nodes/edges arrays inside custom components causes cascade re-renders. Governance-specific decorators (approval status badges, sandbox indicators, RBAC locks) add DOM elements per node that generic flow editors do not have.

**How to avoid:**
- Use `shallowRef` for the nodes and edges arrays in Pinia stores. Never pass the full reactive array into custom node components.
- Wrap all custom node/edge components with Vue's equivalent of memoization (careful prop comparison, `v-memo` where applicable).
- Keep governance decorators as simple as possible: single icon + tooltip, not rich components with their own reactive state.
- Offload dagre layout computation to a Web Worker for graphs with 100+ nodes.
- Profile with Vue DevTools performance tab at 50, 100, and 200 nodes during Phase 1 as a gating criterion.
- Simplify CSS on nodes: no box-shadow, no animations, no gradients in the default theme. Add visual richness only on hover/selection.

**Warning signs:**
- Node drag FPS drops below 30 at 50 nodes
- Inspector panel update latency exceeds 100ms
- Layout computation (dagre) blocks the main thread for more than 500ms

**Phase to address:**
Phase 1 (Foundation) must set performance baselines. Phase 2 (Advanced Canvas) must include 200-node stress testing.

---

### Pitfall 5: Graph State vs. Backend State Divergence

**What goes wrong:**
The frontend canvas state (node positions, edges, properties) drifts from what is persisted in the backend. Users make edits, assume they are saved, close the tab, and lose work. Or worse, the backend has validation rules (governance constraints, type checking) that reject a graph the frontend allowed the user to build, creating a "save failed" error after significant editing effort.

**Why it happens:**
The canvas is inherently optimistic -- users drag and connect freely. The backend enforces governance rules (valid node types, required approval gates, edge type constraints). If validation only happens on save, users discover errors late. Auto-save adds complexity: partial saves can persist invalid intermediate states.

**How to avoid:**
- Implement client-side validation that mirrors backend governance rules. Every edge connection must validate source/target port compatibility before the connection is established (Vue Flow supports connection validation callbacks).
- Design a "draft" vs. "validated" state model: drafts save freely (positions, partial configs), validation runs explicitly before "publish" or "deploy."
- Show validation errors inline on the canvas (red node borders, warning edges) not just in a toast notification.
- Implement optimistic save with conflict detection: save on every meaningful change (debounced), but show a "saved" / "unsaved changes" indicator prominently.

**Warning signs:**
- No auto-save indicator visible in the UI
- Backend rejects graphs that the frontend allowed users to create
- Users must click "Save" and wait for a spinner

**Phase to address:**
Phase 1 (Foundation) for basic save/load with draft model. Phase 2 for inline validation and governance constraint checking.

---

### Pitfall 6: Governance UI Clutter Destroying Canvas Usability

**What goes wrong:**
Every node gets decorated with approval status badges, audit trail links, sandbox indicators, RBAC permission locks, cost attribution data, and execution status -- turning the clean visual graph into an unreadable mess of icons and labels. The governance information that makes Zeroth valuable becomes the thing that makes the editor unusable.

**Why it happens:**
Zeroth's core value proposition is governance. There is organizational pressure to surface ALL governance data on the canvas to differentiate from ungoverned competitors. Each governance feature (approvals, audit, RBAC, economics, sandbox) has its own champion who wants visibility. Without a clear information hierarchy, every indicator gets equal visual weight.

**How to avoid:**
- Adopt a progressive disclosure model with three levels:
  1. **Canvas level:** One subtle icon per node indicating governance status (green check = all clear, yellow = pending approval, red = violation). That is it.
  2. **Selection level:** When a node is selected, the inspector panel shows governance details (approval history, audit log, RBAC permissions, cost data).
  3. **Deep dive level:** Dedicated governance views (audit log panel, approval queue panel) accessible from the shell rail, not from the canvas.
- Never put text labels for governance on canvas nodes. Icons only, with tooltips on hover.
- Provide a "governance overlay" toggle that users can enable/disable to see more detail when needed, rather than always showing it.
- Test the canvas with governance decorators disabled. If the editor is not clean and usable without them, the base UX is broken.

**Warning signs:**
- Node components have more than 2 icon badges visible at default zoom
- Canvas screenshots look "busy" compared to n8n/Retool/Pipedream at the same graph complexity
- Users cannot identify the workflow logic (what connects to what) at a glance because governance chrome dominates

**Phase to address:**
Phase 1 (Foundation) must establish the 3-level information hierarchy. Phase 2 must implement the governance overlay toggle.

---

### Pitfall 7: Undo/Redo That Corrupts Graph State

**What goes wrong:**
Undo/redo appears to work for simple operations (move a node, delete an edge) but corrupts state when operations interact: undo a node deletion that also had edges, undo a property change that triggered validation, undo past an auto-save boundary. Users lose trust in the editor and stop using undo, resorting to manual "save as version X" workflows.

**Why it happens:**
Undo/redo in graph editors is fundamentally harder than in text editors because operations are multi-entity (deleting a node also removes its edges, changes layout, may invalidate governance constraints). Naive implementations use snapshot-based undo (save entire graph state on each change), which is memory-expensive and does not compose well. Command-pattern undo requires every mutation to have a precise inverse, which is difficult when mutations trigger side effects (auto-layout, validation).

**How to avoid:**
- Start with NO custom undo/redo in Phase 1. Ship without it. Let users rely on "revert to last save" instead. This is honest and reliable.
- When implementing undo (Phase 2+), use a snapshot-based approach with structural diff compression, not command-pattern. Snapshots are simpler to reason about and debug.
- Define undo boundaries: an undo step is "the last user-initiated action" not "the last internal state change." Moving a node = 1 undo step, even though it generated dozens of position updates.
- Never undo past a save boundary. Saved state is sacred.
- Test undo sequences that span: node create + edge create + node delete + undo + undo + undo. This is the minimum test matrix.

**Warning signs:**
- Undo sometimes produces a graph state that never existed (phantom nodes, orphan edges)
- Redo after undo does not produce the exact same state
- Memory usage grows linearly with edit count (snapshot leak)

**Phase to address:**
Phase 2 (Advanced Canvas) at earliest. Do NOT attempt in Phase 1.

---

### Pitfall 8: FastAPI + Vue Deployment and Dev Workflow Friction

**What goes wrong:**
The development experience degrades: hot-reload breaks because Vite's dev server and FastAPI's uvicorn conflict on ports or CORS. Production builds require manual coordination of Vue build output and FastAPI static file serving. Docker builds are slow because the Node.js build layer and Python layer are not properly cached. API routes clash with Vue Router's history mode fallback.

**Why it happens:**
FastAPI and Vue are separate ecosystems with no built-in integration story. Every team reinvents the glue: CORS config, static file serving, dev proxy, Docker multi-stage builds. The "it works in dev" to "it works in production" gap is wide because dev uses two servers (Vite + uvicorn) while production typically serves everything from one.

**How to avoid:**
- Dev setup: Vite dev server on port 5173, FastAPI on port 8000. Vite proxies `/api/*` to FastAPI. No CORS needed in dev because same-origin via proxy.
- Production: Vite builds to `dist/`, Nginx serves static files and proxies `/api/*` to FastAPI (Zeroth already has Nginx in its Docker stack from v1.1).
- Docker: Multi-stage build. Stage 1: Node.js builds Vue. Stage 2: Python copies dist/ and installs backend. Layer caching means Vue rebuilds only when frontend source changes.
- Monorepo structure: `studio/` for Vue app at repo root, `src/zeroth/` for Python. Separate `package.json` and `pyproject.toml`. Do NOT nest Vue inside the Python package.
- CI: Run frontend lint/test and backend lint/test as parallel jobs.

**Warning signs:**
- Developers must restart two servers manually after config changes
- CORS errors in the browser console during development
- Production Docker image exceeds 2GB (Node.js runtime not removed after build)
- Vue Router returns 404 on page refresh in production

**Phase to address:**
Phase 1 (Foundation) -- dev workflow and build pipeline must be established before any feature work.

---

### Pitfall 9: WebSocket Complexity Creep Before Core REST API Is Solid

**What goes wrong:**
The team introduces WebSocket connections early for "real-time canvas updates" or "live collaboration" before the basic REST API for graph CRUD is complete and tested. WebSocket code introduces connection lifecycle management (auth, reconnect, heartbeat), message serialization/deserialization, and state synchronization -- tripling the API surface area. Bugs in the WebSocket layer are harder to debug than REST endpoint bugs because state is stateful and ephemeral.

**Why it happens:**
"Real-time" sounds impressive and modern. The existing Zeroth backend already has async infrastructure, making WebSocket support feel like a small addition. But WebSocket endpoints require fundamentally different error handling (no HTTP status codes), testing approaches (no curl/httpie), and state management (connection lifecycle vs. request/response).

**How to avoid:**
- Phase 1 is REST-only. Save graphs with `PUT /api/v1/workflows/{id}`. Load with `GET`. No WebSocket.
- Phase 2 introduces WebSocket for live execution status updates only (read-only push from server to client). Not for graph editing.
- Phase 3 (if ever needed) introduces collaborative editing via WebSocket. This requires CRDT or OT -- do not attempt without dedicated research.
- When WebSocket is introduced: require JWT auth on connect, implement ping/pong heartbeat at 30s intervals, auto-reconnect with exponential backoff on the client, and topic-based message channels (not a single noisy pipe).

**Warning signs:**
- WebSocket endpoint code appears before all REST CRUD endpoints are tested
- No reconnection logic in the client WebSocket handler
- WebSocket messages sent without schema validation (raw JSON.parse)
- More than one WebSocket connection per browser tab

**Phase to address:**
Phase 1 uses REST only. Phase 2 adds read-only WebSocket for execution status. Phase 3+ for collaborative editing.

---

### Pitfall 10: Vue Flow Node Type Explosion and Unmaintainable Custom Nodes

**What goes wrong:**
Every Zeroth concept gets its own custom Vue Flow node type: AgentNode, ExecutionUnitNode, ApprovalGateNode, MemoryResourceNode, WebhookTriggerNode, ToolNode, GuardrailNode, etc. Each custom node has its own template, styles, port layout, and governance decorators. Within a few phases, there are 10+ node type components with heavily duplicated logic, inconsistent styling, and port handle positioning that breaks when one type is updated but others are not.

**Why it happens:**
The domain model has many distinct concepts. The natural instinct is to create a custom node component per domain type. Vue Flow makes custom nodes easy -- too easy. Each node type starts small but accumulates governance badges, status indicators, configuration previews, and port variations.

**How to avoid:**
- Build ONE generic `StudioNode` component with slots for: header (icon + title), body (configurable content area), footer (status indicators), and dynamic port generation based on a port schema.
- Node types are DATA, not components. A node type definition is a JSON/TypeScript config object specifying: icon, color, available ports, governance requirements, inspector schema. The `StudioNode` component renders any type.
- Only create a truly custom node component when the generic one genuinely cannot accommodate the layout (this should be rare -- approval gates may warrant one because they have a fundamentally different interaction model).
- Enforce this in code review: any PR adding a new Vue component in `nodes/` must justify why `StudioNode` is insufficient.

**Warning signs:**
- More than 3 custom node component files in the codebase
- Copy-pasted port handle positioning code across node types
- Governance badge rendering duplicated in multiple node components
- Node type visual inconsistency (different padding, font sizes, border radius)

**Phase to address:**
Phase 1 (Foundation) -- design the generic StudioNode before implementing any specific node type.

---

### Pitfall 11: Inspector Panel Becoming a Second Application

**What goes wrong:**
The right-rail inspector panel grows to include: property editor forms, CodeMirror script editors, governance status panels, approval history timelines, execution logs, cost breakdowns, environment variable editors, and RBAC permission managers. The inspector becomes a full application embedded in a sidebar, with its own routing, state management, and loading states. It competes with the canvas for user attention and developer effort.

**Why it happens:**
The inspector is the "catch-all" for any information that does not fit on the canvas node. Each feature added to Zeroth Studio gets an inspector tab. Without discipline, the inspector accumulates tabs until it requires its own navigation system.

**How to avoid:**
- Inspector shows ONLY properties of the currently selected node/edge. Maximum 3 sections: Properties (editable fields), Governance (read-only status), and Code (CodeMirror editor, only for nodes that have scripts).
- Audit history, execution logs, cost breakdowns, and approval queues belong in dedicated PANELS accessible from the shell rail (left sidebar), not in the inspector.
- Inspector must load instantly. No API calls on selection change -- use data already fetched with the graph. Deep data (audit logs) loads on explicit user action (click "View audit trail" link that opens the dedicated panel).
- Test: select a node, inspect, select a different node. The inspector must update in under 50ms with no loading spinner.

**Warning signs:**
- Inspector has more than 3 tabs
- Inspector makes API calls on node selection
- Inspector has its own Pinia store separate from the graph store
- Developers spend more time on inspector features than canvas features

**Phase to address:**
Phase 1 (Foundation) -- define inspector scope and boundaries. Phase 2 adds governance tab.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Storing node positions in the same API as workflow logic | Simpler API surface | Every logic save triggers position data transfer; position changes pollute version history | MVP only -- split position storage from logic by Phase 2 |
| Inline styles on custom Vue Flow nodes | Fast iteration during prototyping | Inconsistent theming, impossible to skin/rebrand, performance cost of style recalculation | Never -- use CSS classes and CSS variables from day 1 |
| Snapshot-based undo (full graph copy per action) | Simple to implement | Memory grows with edit count; slow for large graphs | Acceptable if snapshots are compressed and capped (50 undo levels max) |
| Single WebSocket for all Studio events | Simple connection management | Noisy channel, no backpressure per event type, hard to debug | Phase 2 only -- use topic-based channels when adding execution status |
| Bundling governance validation only in the save endpoint | Single validation point | Slow saves, no inline feedback, users build invalid graphs for 30 minutes then get rejected | Never -- validate client-side first, server-side as backstop |
| Hardcoded node type registry instead of dynamic loading | Faster to ship | Every new node type requires a code change and redeploy | Phase 1 only -- make registry data-driven by Phase 2 |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Vue Flow + Pinia | Storing Vue Flow's internal state (viewport, selection) in Pinia, causing double-reactivity and stale state bugs | Let Vue Flow own its internal state via `useVueFlow()`. Only store graph data (nodes, edges, metadata) in Pinia. Sync on explicit save events. |
| FastAPI + Vue Router | Both try to handle 404s; FastAPI returns JSON errors for routes Vue should handle | FastAPI catch-all route serves `index.html` for non-API paths. API routes always prefixed with `/api/v1/`. Nginx handles this cleanly in production. |
| dagre + Vue Flow | Running dagre synchronously on the main thread, blocking UI during layout | Use `requestAnimationFrame` or Web Worker for dagre computation. Apply positions in a single batch update to avoid per-node re-render. |
| CodeMirror 6 + Vue 3 | Creating new CodeMirror instances on every inspector panel open, causing memory leaks | Reuse CodeMirror instances. Create once per editor slot, update document content via transactions. Destroy only on component unmount. |
| WebSocket + FastAPI | Using FastAPI's built-in WebSocket with no authentication, no heartbeat, no reconnection | Add JWT auth on WebSocket connect. Implement ping/pong heartbeat (30s interval). Client-side auto-reconnect with exponential backoff. |
| Vue Flow + TypeScript | Typing custom node/edge props loosely (using `any`), losing type safety for governance-specific node data | Define strict TypeScript interfaces for each node type (AgentNode, ApprovalGateNode, ExecutionUnitNode) extending Vue Flow's base Node type. |
| Zeroth Graph API + Studio | Assuming the existing `WorkflowGraph` Pydantic model maps 1:1 to Vue Flow's node/edge format | Create explicit adapter functions: `zerothGraphToVueFlow()` and `vueFlowToZerothGraph()`. The backend model has governance metadata, execution semantics, and contract bindings that are not Vue Flow concepts. |
| Vite proxy + FastAPI auth | Vite proxy strips or fails to forward auth headers/cookies to FastAPI | Configure Vite proxy with `changeOrigin: true` and explicit header forwarding. Test authenticated endpoints through the proxy during dev. |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Re-rendering all nodes on any nodes array change | Sluggish drag, high CPU during interaction | Use `shallowRef` for nodes/edges, memoize custom node components, subscribe to individual node changes not the full array | 30+ nodes |
| Complex CSS on node components (shadows, gradients, animations) | Janky scrolling and zooming, paint storms in DevTools | Flat design for canvas nodes. Shadows/gradients only on hover state. GPU-composited transforms only (`transform`, `opacity`). | 50+ nodes |
| Synchronous dagre layout on every graph change | UI freezes for 200ms+ when adding/removing nodes | Debounce layout computation (300ms). Use Web Worker for graphs > 50 nodes. Only re-layout affected subgraph, not entire graph. | 100+ nodes |
| Unbounded WebSocket message queue | Browser tab memory grows, eventual OOM crash | Implement backpressure: drop stale position updates, batch property updates, cap message queue depth | 10+ concurrent editors or high-frequency updates |
| Storing full graph snapshots for undo history | Memory usage grows 1-5MB per undo step for large graphs | Structural diff compression (store delta, not full snapshot). Cap undo history at 50 steps. Flush on save. | 200+ edits in a session |
| Inspector panel re-rendering on every node drag | Visible jank in the property editor while dragging nodes | Decouple inspector updates from position changes. Inspector reacts to selection and property changes only, not position changes during drag. | Any graph with inspector open during node manipulation |
| Governance badge tooltip rendering for off-screen nodes | Tooltips computed for nodes outside viewport despite Vue Flow virtualization | Tooltips rendered on hover only, not pre-computed. Ensure governance data fetch is lazy per-node. | 100+ nodes with governance decorators |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Executing user-provided code in CodeMirror previews client-side | XSS via crafted agent prompts or tool configurations | CodeMirror is display-only in the browser. Never eval() user content. Server-side sandbox (already exists in Zeroth) handles all execution. |
| WebSocket connections without authentication | Unauthenticated users can read/modify workflow state in real-time | Require JWT token on WebSocket handshake. Validate on connect, reject unauthorized. Re-validate on token expiry. |
| Frontend stores sensitive agent configuration (API keys, secrets) in Pinia state | Keys visible in Vue DevTools, browser memory dumps, localStorage | Never send secrets to the frontend. Display masked values (last 4 chars). Edit via dedicated secure endpoint. Backend resolves secrets at execution time only. |
| Canvas export (screenshot/JSON) includes governance metadata | Audit trail data, approver identities, and cost data leak when users share workflow screenshots | Export function strips governance metadata by default. Provide explicit "include governance data" checkbox for authorized roles only. |
| Workflow JSON import without validation | Malicious workflow JSON could contain script injection in node labels/descriptions, or reference unauthorized node types | Validate all imported JSON against the Zeroth graph schema server-side. Sanitize all string fields. Reject unknown node types. |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Showing all governance data on every node at all times | Canvas is unreadable; users cannot see the workflow for the governance chrome | Progressive disclosure: 1 status icon on canvas, details in inspector, deep dive in dedicated panels |
| Modal dialogs for node configuration | Blocks canvas interaction; users cannot reference other nodes while editing | Side panel inspector (right rail) that stays open while canvas remains interactive |
| Requiring manual save with no auto-save | Users lose work on tab close, browser crash, or navigation | Auto-save drafts every 5 seconds (debounced). Explicit "Publish" for validated/governed state. |
| No visual distinction between node types | Users cannot scan the graph to find approval gates vs. agent nodes vs. execution units | Distinct color-coded headers per node type. Governance nodes (approval gates) visually distinct from compute nodes via shape or accent color. |
| Tiny connection handles on nodes | Edge drawing requires pixel-perfect mouse targeting, frustrating on laptops/trackpads | Large hit targets (16x16px minimum). Magnetic snap-to when cursor is within 20px of a compatible port. |
| No keyboard shortcuts | Power users bottlenecked by mouse-only interaction | Cmd+Z undo, Cmd+S save, Delete to remove, Cmd+A select all, spacebar for pan mode. Document shortcuts in a help overlay (Cmd+/). |
| Zoom level resets on graph reload | Users lose their viewport context when switching between workflows | Persist viewport (zoom, pan) per workflow in localStorage. Restore on reopen. |
| Empty canvas with no guidance | New users stare at blank white space with no idea how to start | Show onboarding prompt: "Drag a node from the sidebar or double-click to add." Pre-populate with a starter template option. |
| Approval gate nodes that look identical to compute nodes | Users forget to add required approval gates; governance violations discovered only at deploy time | Approval gates have a distinct visual language (different shape, mandatory yellow/orange accent). Governance rules surface "missing required approval" as inline canvas warnings. |

---

## "Looks Done But Isn't" Checklist

- [ ] **Canvas resize:** Handles browser window resize, sidebar collapse/expand, and inspector panel toggle without blank areas or cut-off nodes
- [ ] **Edge routing:** Edges re-route when nodes are moved; no edges permanently passing through other nodes (dagre re-layout on significant moves)
- [ ] **Empty state:** New workflow shows helpful onboarding (drag hint, "add your first node" prompt) not a blank canvas
- [ ] **Error recovery:** If the backend is unreachable, the canvas shows a clear offline indicator and queues saves for retry -- does not silently lose edits
- [ ] **Browser back button:** Navigating away from the editor with unsaved changes shows a "discard changes?" prompt (beforeunload handler)
- [ ] **Touch/trackpad:** Pinch-to-zoom and two-finger pan work correctly on trackpads; not just mouse wheel zoom
- [ ] **Governance node validation:** Approval gate nodes without assigned approvers show a validation warning, not just silently invalid
- [ ] **Edge deletion governance:** Deleting an edge between governance-required nodes (e.g., removing the only path through an approval gate) shows a governance violation warning
- [ ] **Multiple selection:** Cmd+click multi-select, drag-to-select rectangle, and group operations (move, delete, copy) all work consistently
- [ ] **Node overflow:** Nodes with long names or many ports handle overflow gracefully (truncation with tooltip, scrollable port list) not layout breakage
- [ ] **Graph-to-backend round-trip:** Create graph in Studio, save, reload page, verify identical graph. Test with 20+ nodes and governance decorators.
- [ ] **Concurrent browser tabs:** Two tabs editing the same workflow do not silently overwrite each other. At minimum: last-write-wins with conflict detection and warning.
- [ ] **CodeMirror cleanup:** Opening and closing 20 nodes with code editors does not leak CodeMirror instances (check memory in DevTools).

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| n8n license contamination | HIGH | Legal audit of all frontend code. Potentially rewrite contaminated components from clean-room specs. Document provenance of every component. |
| Vue Flow container sizing failures | LOW | Add CSS reset and explicit dimensions. 10-minute fix once understood. |
| Over-engineering before validation | HIGH | Scrap unvalidated infrastructure. Return to basic canvas. Lost time is unrecoverable. |
| Performance collapse at scale | MEDIUM | Audit custom node components for reactivity leaks. Add shallowRef, memoization. May require custom node component rewrites. 1-2 sprint effort. |
| Graph state divergence (data loss) | HIGH | Implement auto-save with revision history. Add "version revert" UI. Cannot recover already-lost user data. |
| Governance UI clutter | MEDIUM | Redesign information hierarchy. Move decorators to inspector panel. Requires UX redesign but not architectural change. 1 sprint. |
| Undo/redo corruption | MEDIUM | Replace command-pattern with snapshot-based approach. Clear undo history. Disable undo until fixed. 1-2 sprint effort. |
| Deployment/dev workflow friction | LOW | Fix once with proper Vite proxy config, Docker multi-stage build, and Nginx routing. Template-able for future projects. |
| WebSocket complexity creep | MEDIUM | Revert to REST-only for graph CRUD. Keep WebSocket only for execution status push. Remove collaborative editing. |
| Node type explosion | MEDIUM | Refactor to generic StudioNode. Extract type-specific logic to config objects. 1-2 sprint effort but touches every node type. |
| Inspector becoming second app | MEDIUM | Audit inspector tabs. Move non-essential content to dedicated panels. Enforce 3-section max rule. |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| n8n license contamination | Phase 1 (Foundation) | PR review checklist item: "No n8n source code referenced." Clean-room policy document exists. |
| Vue Flow container sizing | Phase 1 (Foundation) | Canvas renders correctly at 3 viewport sizes (1280, 1920, 4K). Resize handler tested. |
| Over-engineering before validation | Phase 1 (Foundation) | Phase 1 exit criteria: working single-user save/load demo with 5+ node types. No WebSocket code exists. |
| Performance at scale | Phase 1 baseline, Phase 2 stress test | 50-node drag test at 60fps in Phase 1. 200-node layout under 500ms in Phase 2. Profiled in CI. |
| Graph state divergence | Phase 1 (save/load), Phase 2 (auto-save + validation) | Round-trip test: create graph, save, reload, compare. Auto-save indicator visible. |
| Governance UI clutter | Phase 1 (hierarchy design), Phase 2 (overlay toggle) | Canvas screenshot test: governance-decorated graph readable at 20 nodes. |
| Undo/redo corruption | Phase 2 (Advanced Canvas) | Undo sequence: create 5 nodes, 4 edges, delete 2 nodes, undo 3x, verify graph integrity. |
| FastAPI + Vue deployment | Phase 1 (Foundation) | `make dev` starts both servers with hot-reload. `make build` produces working Docker image under 1GB. |
| WebSocket complexity creep | Phase 2 (execution status only) | No WebSocket code in Phase 1. Phase 2 WebSocket is read-only push. |
| Node type explosion | Phase 1 (Foundation) | Single StudioNode component handles all node types. Max 2 custom node components (StudioNode + ApprovalGateNode). |
| Inspector scope creep | Phase 1 (Foundation) | Inspector has max 3 sections. No API calls on selection. Updates in under 50ms. |

---

## Sources

- [Vue Flow Troubleshooting Guide](https://vueflow.dev/guide/troubleshooting.html) -- official container sizing, node/edge validation errors (HIGH confidence)
- [React Flow Performance Guide](https://reactflow.dev/learn/advanced-use/performance) -- memoization, re-render pitfalls, shared architecture patterns (HIGH confidence)
- [n8n Sustainable Use License](https://docs.n8n.io/sustainable-use-license/) -- license restrictions, redistribution prohibitions (HIGH confidence)
- [The Real Limits of n8n Free Automation](https://dev.to/alifar/the-real-limits-of-n8n-free-automation-what-you-need-to-know-before-shipping-to-production-59o6) -- SUL legal exposure scenarios (MEDIUM confidence)
- [Supercharging Vue Flow with Web Workers](https://medium.com/@talmogendorff/supercharging-your-vue-flows-workflows-with-web-workers-32b1703fdf6e) -- offloading computation from main thread (MEDIUM confidence)
- [You Don't Know Undo/Redo](https://dev.to/isaachagoel/you-dont-know-undoredo-4hol) -- undo complexity in graph/collaborative editors (MEDIUM confidence)
- [Graph Visualization UX Guide](https://cambridge-intelligence.com/a-guide-to-graph-ux-or-how-to-avoid-wrecking-your-graph-visualization/) -- progressive disclosure, overcrowding prevention (MEDIUM confidence)
- [Complex Approvals App Design (UXPin)](https://www.uxpin.com/studio/blog/complex-approvals-app-design/) -- approval gate UX patterns (MEDIUM confidence)
- [FastAPI + Vue Project Structure Discussion](https://github.com/fastapi/fastapi/discussions/4344) -- monorepo layout patterns (MEDIUM confidence)
- [Rerun Blog: Graphs, Drag & Drop and Undo](https://rerun.io/blog/graphs) -- graph editor undo implementation insights (MEDIUM confidence)
- [FastAPI + Vue Deployment Guide](https://imadsaddik.com/blogs/how-to-manually-deploy-a-vuejs-and-fastapi-application) -- deployment patterns (MEDIUM confidence)
- [How to Design Real-Time Collaboration with FastAPI and WebSockets](https://hexshift.medium.com/how-to-design-real-time-collaboration-tools-with-fastapi-and-websockets-baa711557039) -- WebSocket architecture patterns (MEDIUM confidence)
- [Vue Flow DeepWiki](https://deepwiki.com/bcakmakoglu/vue-flow) -- architecture and performance internals (MEDIUM confidence)

---

*Pitfalls research for: Zeroth Studio visual workflow editor*
*Researched: 2026-04-09*
