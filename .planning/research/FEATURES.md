# Feature Landscape: Zeroth Studio Visual Workflow Editor

**Domain:** Visual workflow authoring UI for governed multi-agent AI platform
**Researched:** 2026-04-09
**Confidence:** HIGH (cross-referenced 6 competing platforms, n8n architecture deep dive, official docs)

---

## Context: What Already Exists (Backend)

Zeroth v1.1 shipped a complete governed backend: graph-based workflow modeling with validation and versioning, runtime orchestration with approvals/memory/durable dispatch, identity/RBAC/governance evidence/audit trails, 100+ LLM models via LiteLLM with token economics (Regulus), external memory connectors, container sandboxing, webhooks, and containerized deployment. The Studio milestone adds a visual authoring frontend on top of this existing backend.

**Chosen stack:** Vue 3 / Vite / Pinia / Vue Flow / dagre / CodeMirror 6 (already decided in PROJECT.md).

---

## Table Stakes

Features users expect from any visual workflow editor in 2025-2026. Missing any of these makes the editor feel broken or incomplete. Every platform in the space (n8n, Dify, Langflow, Flowise, Rivet, ComfyUI) ships all of these.

| Feature | Why Expected | Complexity | Backend Dependency |
|---------|--------------|------------|-------------------|
| **Drag-and-drop node placement on canvas** | Core interaction model for all node-graph editors. Users drag nodes from a palette onto a pannable, zoomable canvas. n8n, Dify, Langflow, Flowise, Rivet, ComfyUI all use this exact pattern. | MEDIUM | None (pure frontend) |
| **Edge drawing between node ports** | Connecting outputs to inputs by dragging handles is the universal paradigm. Vue Flow provides this natively. Must support typed ports (data flow vs control flow) and validate connection compatibility. | LOW | Graph model validation API |
| **Node palette / library sidebar** | Left-side panel listing available node types grouped by category (Agents, Execution Units, Approval Gates, Memory, Tools). n8n calls this the "Node Creator"; Dify and Langflow use categorized sidebars. Users expect to search, filter, and drag from this panel. | MEDIUM | Asset catalog API (list available node types) |
| **Inspector / properties panel** | Right-side panel showing configuration for the selected node. n8n's NDV (Node Detail View) is the gold standard: parameters, settings, and docs tabs. Every platform has this. Must update reactively on node selection. | HIGH | Node schema API (field definitions per node type) |
| **Canvas navigation (pan, zoom, fit-to-view)** | Zoom in/out (scroll wheel), pan (space+drag or middle-click), fit-all-nodes-to-viewport button, minimap for orientation. Vue Flow provides pan/zoom natively; minimap is a Vue Flow plugin. | LOW | None (Vue Flow built-in) |
| **Auto-layout / tidy-up** | One-click button to arrange nodes in a readable left-to-right (or top-to-bottom) DAG layout. ComfyUI's biggest UX complaint is manual layout drudgery. dagre (already in stack) handles this. n8n uses dagre for the same purpose. | LOW | None (dagre, frontend-only) |
| **Undo/redo** | n8n implements this via a command pattern (AddNodeCommand, MoveNodeCommand, RemoveNodeCommand, AddConnectionCommand). Users expect Ctrl+Z/Ctrl+Y to work for all canvas mutations. Non-negotiable for any editor. | MEDIUM | None (frontend command stack) |
| **Save / load workflows** | Persist the authored graph to the backend and reload it. Must handle dirty-state detection ("Unsaved Changes" indicator, like n8n's `markStateDirty()`). Needs the Graph Authoring API (REST). | LOW | Graph CRUD API (already exists in backend) |
| **Node validation indicators** | Visual badges/icons on nodes showing errors (missing required fields, invalid connections, type mismatches). n8n shows validation issues directly on node badges. Dify highlights broken nodes in red. | MEDIUM | Backend validation API (already exists) |
| **Workflow execution trigger + status** | "Run" button that triggers workflow execution and shows per-node execution status (pending, running, success, error). Dify and n8n both show execution flow visually on the canvas with colored borders/badges. | MEDIUM | Run creation API, WebSocket for status updates |
| **Per-node execution results viewer** | After a run completes, clicking any node shows its input/output data, execution time, and token usage. Rivet and Dify both highlight this as a core debugging feature. Essential for understanding agent behavior. | MEDIUM | Run result API (per-node audit records) |
| **Keyboard shortcuts** | Delete (backspace/delete), select all (Ctrl+A), copy/paste nodes (Ctrl+C/V), duplicate (Ctrl+D). All canvas editors support these. Vue Flow has built-in key handling that needs to be wired. | LOW | None (frontend-only) |
| **Responsive three-panel layout** | Left rail (node palette/workflow list), center canvas, right inspector. Collapsible panels. n8n's `useSidebarLayout` composable manages this. This is the universal layout for workflow editors. | MEDIUM | None (frontend-only) |

### Table Stakes Specific to AI Agent Workflows

These go beyond generic workflow editors. Platforms focused on AI agents (Langflow, Dify, Rivet) treat these as baseline.

| Feature | Why Expected | Complexity | Backend Dependency |
|---------|--------------|------------|-------------------|
| **Model/provider selector per agent node** | Every AI workflow editor lets users pick which LLM to use per node. Langflow, Dify, and Rivet all have model dropdowns. Zeroth has 100+ models via LiteLLM; the inspector must expose this. | LOW | LiteLLM model list API |
| **Prompt/system-message editor** | Inline code editor for prompt templates in agent nodes. CodeMirror 6 (already in stack) provides syntax highlighting, variable interpolation display, and multi-line editing. Dify's Prompt IDE is a standout here. | MEDIUM | None (CodeMirror 6, frontend) |
| **Variable/context passing visualization** | Show how data flows between nodes -- what outputs feed into which inputs. Dify highlights connected nodes on Shift+click. Rivet shows real-time data on edges during execution. At minimum: typed port labels and hover tooltips. | MEDIUM | Graph schema (port type definitions) |
| **Tool/function attachment to agents** | Agent nodes need a way to attach tools (execution units, external APIs). Langflow's "Tool Mode" toggle on components is elegant. Rivet wires tools as connected nodes. Zeroth should show tool connections as distinct edge types. | MEDIUM | Agent tool binding API |

---

## Differentiators

Features that set Zeroth Studio apart. These exploit the governed backend that no competitor has. This is where Zeroth's unique value proposition lives.

| Feature | Value Proposition | Complexity | Backend Dependency |
|---------|-------------------|------------|-------------------|
| **Approval gate visualization** | No competing visual editor shows human-in-the-loop approval gates as first-class visual nodes with SLA timers, escalation indicators, and approval status badges. Zeroth has native `HumanApprovalNode` with SLA timeouts and escalation policies -- visualizing this is a unique capability. Show: pending/approved/rejected/escalated states, countdown timer for SLA, who-approved attribution. | HIGH | Approval status API, WebSocket for live updates |
| **Audit trail overlay per node** | Click any node to see its governance evidence: digest-chained audit records, who modified it, when it last ran, compliance status. No other visual editor surfaces audit data at the node level. Dify shows execution logs; Zeroth shows tamper-evident governance evidence. | MEDIUM | Audit trail API (already exists) |
| **Sandbox indicator badges** | Visual indicator on execution unit nodes showing sandbox mode (Docker/local/untrusted), resource constraints (CPU/memory limits), and network isolation status. No competitor visualizes sandbox posture. Users need to see at a glance which nodes run in constrained environments. | LOW | Sandbox config from node schema |
| **RBAC-aware canvas (read-only mode, restricted editing)** | Canvas respects the user's role: viewers see but cannot edit; operators can run but not modify graphs; authors have full edit access. n8n has basic RBAC (read-only from source control + collaboration locks), but Zeroth's per-resource RBAC is more granular. Disable drag/connect/delete interactions based on permissions. | MEDIUM | RBAC permission check API (already exists) |
| **Token cost overlay per node** | After execution, show token usage and cost (USD) as badges or tooltips on each agent node. Dify shows token counts in its debugger; Zeroth adds cost attribution via Regulus. Visualizing spend per node is unique in the space. | LOW | Regulus cost data from run results |
| **Budget gauge / spend dashboard** | Show remaining budget for the current tenant/deployment as a gauge or progress bar in the Studio header. Warn when approaching limits. No competitor has this because no competitor has budget enforcement. | LOW | Budget API (Regulus, already exists) |
| **Governance evidence bundle export** | One-click export of a workflow's complete governance evidence: graph version, audit trail, approval records, cost attribution, sandbox configurations. Packaged as a compliance artifact (JSON/PDF). Unique to governed platforms. | MEDIUM | Governance evidence API (already exists in backend) |
| **Environment-aware canvas** | Show which environment (dev/staging/prod) a workflow targets. Visual differentiation (color-coded header, environment badge). Prevent accidental edits to production workflows. Tied to deployment-time bindings. | LOW | Environment management API |
| **Workflow versioning with diff view** | Side-by-side or inline diff of graph versions showing added/removed/modified nodes and edges. Zeroth already has graph versioning in the backend. Visual diff is high-value for governance (who changed what, when). | HIGH | Graph version diff API |
| **Collaborative presence indicators** | Show which users are viewing/editing the same workflow (avatar pills on canvas, similar to n8n's collaboration feature). Combined with RBAC, this prevents conflicting edits. | MEDIUM | WebSocket presence API |

---

## Anti-Features

Features to explicitly NOT build. These are tempting but wrong for Zeroth Studio.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| **No-code / code-free promise** | Zeroth is a "medium-code" platform per PROJECT.md. Trying to hide all code behind a GUI leads to lowest-common-denominator expressiveness (ComfyUI's UX trap). Langflow and Flowise target non-developers; Zeroth targets teams with developers. | Expose CodeMirror editors for prompts, Python snippets in execution units, and JSON schema editing. "Medium-code" means visual structure with code where it matters. |
| **500+ integration nodes (n8n-style)** | n8n's value is breadth of integrations (400+ nodes). Zeroth's value is governed AI workflows. Building Slack/Gmail/Sheets nodes is a distraction that competitors do better. | Focus on AI-specific node types: Agent, ExecutionUnit, ApprovalGate, MemoryResource, Tool, Webhook. External integrations happen via HTTP/webhook nodes or execution unit code. |
| **Visual RAG pipeline builder** | Dify is building this (visual chunking/embedding/indexing). It is a separate product concern. Zeroth's memory connectors handle retrieval; the chunking/indexing pipeline is out of scope. | Memory resource nodes connect to existing external memory backends (Redis, pgvector, ChromaDB, Elasticsearch). Ingestion pipelines are the user's responsibility. |
| **Real-time LLM streaming on canvas** | Showing tokens appearing in real-time on the canvas (like ChatGPT typing) is flashy but architecturally complex. Zeroth uses async step-completion execution, not streaming. PROJECT.md explicitly marks streaming as out of scope. | Show execution status badges (pending/running/complete/error) and final results after completion. Sufficient for workflow authoring context. |
| **Marketplace / community node sharing** | Langflow has community components; n8n has a node marketplace. Building a marketplace is a product in itself -- discovery, quality control, versioning, security review. | Ship a well-designed core node library. Custom nodes via execution unit code. Marketplace can be a future milestone if demand materializes. |
| **Mobile-responsive canvas** | Node-graph editors are inherently desktop experiences. Responsive canvas for mobile is enormous effort with minimal payoff. n8n explicitly handles mobile as panning-only mode. | Desktop/tablet only. Set a minimum viewport width. Mobile users get a read-only workflow list view at most. |
| **AI-generated workflow suggestions** | "AI builds your workflow" is a hype feature. The graph authoring UX should be fast enough that users don't need AI to assemble nodes. Copilot-for-workflows is a research project, not a v2.0 feature. | Focus on fast node search, smart defaults, and template workflows instead. |
| **Embedded chat/test panel** | Dify and Langflow embed a chat panel for testing chatbot workflows. Zeroth is not a chatbot builder -- it orchestrates multi-agent workflows. | Test via the "Run" button with configurable inputs. Results appear in per-node result viewers. |

---

## Feature Dependencies

```
Node Palette / Library Sidebar
    +-- requires --> Asset Catalog API (list node types with schemas)

Inspector / Properties Panel
    +-- requires --> Node Schema API (field definitions, validation rules)
    +-- requires --> Model List API (for agent node model selector)

Canvas Save/Load
    +-- requires --> Graph Authoring API (CRUD for workflow graphs)
    +-- requires --> WebSocket connection (dirty state, collaboration)

Workflow Execution + Status
    +-- requires --> Run Creation API
    +-- requires --> WebSocket for real-time status updates
    +-- enables --> Per-node execution results viewer

Approval Gate Visualization
    +-- requires --> Approval Status API
    +-- requires --> WebSocket for live approval state changes
    +-- depends-on --> Base canvas (node rendering, edge drawing)

Audit Trail Overlay
    +-- requires --> Audit Trail API (already exists in backend)
    +-- depends-on --> Inspector panel (displays in inspector tab)

RBAC-Aware Canvas
    +-- requires --> Permission Check API (already exists)
    +-- affects --> All canvas interactions (drag, connect, delete, save)

Token Cost Overlay
    +-- requires --> Run Result API with cost data (Regulus)
    +-- depends-on --> Workflow execution feature

Workflow Versioning Diff
    +-- requires --> Graph Version Diff API (new backend endpoint)
    +-- depends-on --> Save/load workflows

Environment-Aware Canvas
    +-- requires --> Environment Management API (new backend work)
    +-- depends-on --> Canvas shell (header area for environment selector)
```

### Critical Path

```
Graph Authoring API --> Canvas Save/Load --> Everything else
     |
     +--> Node Schema API --> Inspector Panel --> Full editing UX
     |
     +--> WebSocket layer --> Execution status, approvals, collaboration
```

The Graph Authoring API (REST + WebSocket) is the single highest-priority backend dependency. Without it, the canvas cannot persist or load workflows.

---

## MVP Recommendation

### Phase 1: Canvas Foundation (must ship first)

1. **Three-panel shell layout** (left rail, canvas, right inspector)
2. **Drag-and-drop node placement** with Vue Flow canvas
3. **Edge drawing** between typed ports
4. **Node palette** with Zeroth's node types (Agent, ExecutionUnit, ApprovalGate, MemoryResource)
5. **Inspector panel** with node configuration forms
6. **Auto-layout** via dagre
7. **Save/load** via Graph Authoring API
8. **Undo/redo** via command pattern
9. **Keyboard shortcuts** (delete, select-all, copy/paste)
10. **Canvas navigation** (pan, zoom, fit-to-view, minimap)

### Phase 2: Execution and Governance Visualization

1. **Workflow execution trigger** with per-node status badges
2. **Per-node execution results viewer** (input/output/tokens/cost)
3. **Approval gate visualization** (status, SLA timer, attribution)
4. **Node validation indicators**
5. **RBAC-aware canvas** (read-only mode for viewers)
6. **Audit trail overlay** in inspector
7. **Sandbox indicator badges**

### Phase 3: Advanced Authoring

1. **Prompt/system-message editor** (CodeMirror 6)
2. **Model/provider selector** per agent node
3. **Token cost overlay** per node
4. **Budget gauge** in header
5. **Environment-aware canvas** with environment selector
6. **Variable/context passing visualization**
7. **Tool/function attachment** to agent nodes

### Defer to v2.1+

- **Workflow versioning diff view** -- HIGH complexity, high value but not launch-critical
- **Collaborative presence indicators** -- requires WebSocket presence infrastructure
- **Governance evidence bundle export** -- backend already supports it; UI is low priority vs core editing
- **Template workflow library** -- content creation effort, not engineering effort

---

## Competitor Feature Matrix

| Feature | n8n | Dify | Langflow | Flowise | Rivet | ComfyUI | Zeroth Studio |
|---------|-----|------|----------|---------|-------|---------|---------------|
| Drag-drop canvas | Yes | Yes | Yes | Yes | Yes | Yes | Yes (table stakes) |
| Typed port connections | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Inspector/properties panel | Yes (NDV) | Yes | Yes | Yes | Yes | Inline only | Yes |
| Auto-layout | Yes (dagre) | Partial | No | No | No | No | Yes (dagre) |
| Undo/redo | Yes (commands) | Limited | No | No | Yes | No | Yes (commands) |
| Per-node execution results | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Visual debugging | Basic | Strong | Strong | Basic | Strong | Basic | Strong (planned) |
| Human approval gates | Via integration | No | No | No | No | No | **Native, first-class** |
| Audit trail per node | No | No | No | No | No | No | **Native, governance** |
| RBAC-aware editing | Basic | Basic | No | No | No | No | **Granular, per-resource** |
| Sandbox visualization | No | No | No | No | No | No | **Unique** |
| Cost attribution per node | No | Token count only | No | No | No | No | **USD cost via Regulus** |
| Budget enforcement UI | No | No | No | No | No | No | **Unique** |
| Governance evidence export | No | No | No | No | No | No | **Unique** |
| Environment management | No | No | No | No | No | No | **Deployment-time bindings** |
| Graph versioning/diff | Version history | No | No | No | No | No | **Visual diff (planned)** |
| 400+ integrations | Yes | Partial | Via LangChain | Partial | No | Custom nodes | No (by design) |
| Embedded chat test | No | Yes | Yes | Yes | No | No | No (by design) |

**Key insight:** Zeroth Studio's differentiators cluster around governance visualization (approvals, audit, RBAC, sandbox, cost). No competitor in the AI workflow editor space offers any of these. This is the defensible moat -- not breadth of integrations or no-code simplicity.

---

## Sources

- [n8n Workflow Canvas Architecture (DeepWiki)](https://deepwiki.com/n8n-io/n8n/6.2-workflow-canvas-and-node-management) -- HIGH confidence (source code analysis)
- [n8n Editor UI Documentation](https://docs.n8n.io/courses/level-one/chapter-1/) -- HIGH confidence (official docs)
- [n8n Human-in-the-Loop](https://blog.n8n.io/human-in-the-loop-automation/) -- HIGH confidence (official blog)
- [Langflow Documentation](https://docs.langflow.org/) -- HIGH confidence (official docs)
- [Langflow GitHub](https://github.com/langflow-ai/langflow) -- HIGH confidence
- [Flowise AgentFlow V2](https://docs.flowiseai.com/using-flowise/agentflowv2) -- HIGH confidence (official docs)
- [Dify GitHub](https://github.com/langgenius/dify) -- HIGH confidence
- [Dify 2025 Summer Highlights](https://dify.ai/blog/2025-dify-summer-highlights) -- HIGH confidence (official blog)
- [Rivet Documentation](https://rivet.ironcladapp.com/docs) -- HIGH confidence (official docs)
- [Rivet GitHub](https://github.com/Ironclad/rivet) -- HIGH confidence
- [ComfyUI Workflow Docs](https://docs.comfy.org/development/core-concepts/workflow) -- HIGH confidence (official docs)
- [Open Source AI Agent Platform Comparison](https://jimmysong.io/blog/open-source-ai-agent-workflow-comparison/) -- MEDIUM confidence
- [React Flow / Vue Flow patterns](https://reactflow.dev/ui/templates/workflow-editor) -- HIGH confidence (framework docs)
- [Awesome Node-Based UIs](https://github.com/xyflow/awesome-node-based-uis) -- MEDIUM confidence (curated list)
- Zeroth PROJECT.md and codebase -- HIGH confidence (direct source)

---

*Feature research for: Zeroth v2.0 Studio Visual Workflow Editor*
*Researched: 2026-04-09*
