# Project Research Summary

**Project:** Zeroth Studio -- Visual Workflow Editor
**Domain:** Visual workflow authoring UI for governed multi-agent AI platform
**Researched:** 2026-04-09
**Confidence:** HIGH

## Executive Summary

Zeroth Studio is a visual graph editor frontend (Vue 3 SPA) layered on top of an already-complete governed multi-agent backend (v1.1). The domain is well-understood: six competing platforms (n8n, Dify, Langflow, Flowise, Rivet, ComfyUI) establish a clear table-stakes feature set -- drag-and-drop canvas, edge drawing, node inspector, save/load, undo/redo, execution visualization. The recommended stack (Vue 3, Vue Flow, Pinia, Tailwind CSS 4, Reka UI headless components) is mature, version-verified, and follows the same patterns n8n uses without copying its code. No new backend dependencies are needed; the existing FastAPI/Postgres/Redis stack supports all Studio requirements through new REST routes and WebSocket endpoints.

The recommended approach is phased delivery with strict scope discipline. Phase 1 delivers a single-user canvas that can place nodes, draw edges, configure properties, and save/load graphs via REST -- nothing more. WebSocket, collaboration, and advanced undo/redo are deferred to later phases. This ordering is driven by the single most dangerous pitfall identified: over-engineering infrastructure (real-time sync, CRDTs, plugin systems) before validating that the core editing loop feels good. The existing backend sophistication creates pressure to match it immediately in the frontend; resisting that pressure is critical.

Zeroth Studio's defensible differentiation is governance visualization -- approval gate status, audit trails, sandbox indicators, RBAC-aware editing, per-node cost attribution. No competitor surfaces any of these. However, the same governance data creates the biggest UX risk: cluttering the canvas with badges and indicators until the workflow graph is unreadable. The research strongly recommends a three-level progressive disclosure model (single icon on canvas, details in inspector, deep dives in dedicated panels) established in Phase 1 and enforced throughout.

## Key Findings

### Recommended Stack

The frontend stack is fully decided and version-verified. Vue 3.5.x with Pinia 3 for state, Vue Flow 1.48.x for the graph canvas, dagre 3.0.0 for auto-layout, Reka UI 2.9.x (headless) with Tailwind CSS 4 for UI components, CodeMirror 6 for code editing, and VueUse 14.x for composable utilities including WebSocket. The backend requires no new packages -- FastAPI's native WebSocket, existing Redis pub/sub, and existing GraphRepository cover all needs.

**Core technologies:**
- **Vue Flow 1.48.x**: Interactive flow canvas with pan/zoom/drag/edge-drawing -- same library n8n uses, MIT licensed
- **Reka UI 2.9.x + Tailwind CSS 4**: Headless accessible components with full visual control -- pre-styled libraries (Element Plus, Vuetify) fight custom graph editor UIs
- **Pinia 3**: Single source of truth for graph state; Vue Flow derives its rendering from the Pinia store via a useCanvasMapping composable
- **VueUse useWebSocket**: Reactive WebSocket client with auto-reconnect, replacing unmaintained alternatives
- **ky**: 2KB HTTP client wrapping native Fetch -- replaces Axios at 15% of the bundle size
- **openapi-typescript**: Generates frontend types from FastAPI's OpenAPI spec, keeping Pydantic models and TypeScript types in sync automatically

### Expected Features

**Must have (table stakes):**
- Drag-and-drop node placement on pannable/zoomable canvas
- Edge drawing between typed ports with connection validation
- Node palette sidebar with search/filter by category
- Inspector panel for node configuration (properties, settings)
- Auto-layout via dagre, canvas navigation (pan/zoom/fit/minimap)
- Save/load workflows via REST API with dirty-state indicator
- Keyboard shortcuts (delete, select-all, copy/paste, undo)
- Workflow execution trigger with per-node status badges
- Per-node execution results viewer (input/output/tokens/cost)
- Model/provider selector per agent node (100+ models via LiteLLM)
- Prompt editor with CodeMirror 6

**Should have (differentiators -- Zeroth's governance moat):**
- Approval gate visualization with SLA timers and status badges
- Audit trail overlay per node (tamper-evident governance evidence)
- RBAC-aware canvas (read-only for viewers, restricted for operators)
- Sandbox indicator badges on execution unit nodes
- Token cost overlay and budget gauge (Regulus cost attribution)
- Environment-aware canvas with deployment-target indicators

**Defer (v2.1+):**
- Workflow versioning diff view (HIGH complexity)
- Collaborative presence indicators (requires WebSocket presence infra)
- Governance evidence bundle export (backend exists; UI is low priority)
- Template workflow library (content effort, not engineering)

### Architecture Approach

The architecture uses Nginx as the single entry point -- serving the Vue SPA as static files and reverse-proxying API/WebSocket traffic to FastAPI. The frontend follows a feature-sliced directory structure (canvas, inspector, workflow-rail, validation, execution, environments) with a shared layer for API client, auth, and reusable UI components. The critical architectural pattern is the useCanvasMapping composable that bridges Zeroth's Graph Pydantic model and Vue Flow's internal node/edge format bidirectionally. All canvas mutations flow through useCanvasOperations which updates the Pinia graphStore (single source of truth) and optionally syncs to the backend.

**Major components:**
1. **Nginx** -- TLS termination, static file serving, reverse proxy for /v2/ REST and /ws/ WebSocket
2. **FastAPI v2 Studio API** -- Thin REST wrapper around existing GraphRepository for graph/node/edge CRUD, validation, asset catalog
3. **WebSocket Hub** -- Connection manager grouped by graph_id, topic-based message routing, Redis pub/sub for multi-worker broadcast
4. **Vue SPA (studio/)** -- Feature-sliced Vue 3 app: canvas (Vue Flow), inspector, workflow rail, validation panel, execution panel
5. **graphStore (Pinia)** -- Single source of truth for the Graph document; canvas and inspector derive state from it reactively

### Critical Pitfalls

1. **n8n license contamination** -- n8n's SUL license prohibits code derivation. Use n8n as UX reference (screenshots only), never open its source during implementation. Establish clean-room policy before Phase 1.
2. **Over-engineering before core validation** -- Do not build WebSocket sync, CRDTs, or plugin systems before the basic edit-save-reload loop works and feels good. Phase 1 must be REST-only, single-user, zero-collaboration.
3. **Performance collapse at 50+ nodes** -- Custom governance decorators on every node compound rendering cost. Use shallowRef for node arrays, memoize node components, keep governance badges to single icons (no rich sub-components), profile at 50/100/200 nodes as gating criteria.
4. **Graph state divergence / data loss** -- Optimistic canvas edits can diverge from backend-validated state. Implement client-side validation mirroring governance rules, draft vs. published state model, debounced auto-save with conflict detection.
5. **Governance UI clutter** -- Surfacing all governance data on the canvas destroys readability. Enforce three-level progressive disclosure: one icon on canvas, details in inspector, deep dives in dedicated panels. Test the canvas with governance decorators disabled -- if it is not clean without them, the base UX is broken.
6. **Node type explosion** -- Build ONE generic StudioNode component driven by type configuration data, not one custom component per domain type. Maximum 2-3 custom node components total.

## Implications for Roadmap

Based on research, the project naturally divides into 4 phases driven by dependency ordering and risk mitigation.

### Phase 1: Canvas Foundation and Dev Infrastructure
**Rationale:** Everything depends on a working canvas with save/load. The dev workflow (Vite + FastAPI proxy + Docker) must be established before any feature work. This phase also establishes the clean-room policy, generic StudioNode pattern, and progressive disclosure hierarchy -- decisions that are expensive to change later.
**Delivers:** Single-user visual editor that can create, edit, and persist workflow graphs. Three-panel layout. Basic node palette with all Zeroth node types. Inspector panel with property editing. Auto-layout. Keyboard shortcuts.
**Addresses:** All 13 table-stakes features from FEATURES.md (canvas interaction, save/load, node palette, inspector, navigation, auto-layout, keyboard shortcuts)
**Avoids:** Over-engineering (Pitfall 3), Vue Flow container sizing (Pitfall 2), n8n license contamination (Pitfall 1), node type explosion (Pitfall 10), inspector scope creep (Pitfall 11), deployment friction (Pitfall 8)
**Backend work:** v2 Studio REST API (thin wrapper around GraphRepository), Nginx config updates, Docker multi-stage build with Node.js frontend stage

### Phase 2: Execution Visualization and Governance Layer
**Rationale:** With the core editing loop validated, add the execution feedback loop (run workflows, see per-node status) and Zeroth's governance differentiators. WebSocket is introduced here for execution status push (read-only, server-to-client) -- not for graph editing.
**Delivers:** Run button with per-node execution status. Approval gate visualization with SLA timers. Audit trail overlay in inspector. Sandbox badges. RBAC-aware canvas (read-only mode). Node validation indicators. Governance overlay toggle.
**Addresses:** All Phase 2 features from FEATURES.md (execution, approval gates, audit trail, RBAC, sandbox indicators, validation)
**Avoids:** WebSocket complexity creep (Pitfall 9 -- WS is read-only push only), governance UI clutter (Pitfall 6 -- progressive disclosure enforced), graph state divergence (Pitfall 5 -- inline validation added)

### Phase 3: Advanced Authoring and Economics
**Rationale:** With editing and execution working, add the authoring power features (CodeMirror prompt editor, model selector, tool attachment) and Zeroth's economic differentiators (token cost overlay, budget gauge). These features have no blocking dependencies on earlier phases beyond a working canvas and execution pipeline.
**Delivers:** CodeMirror-powered prompt/system-message editor. Model/provider selector per agent node. Token cost overlay per node. Budget gauge in header. Environment-aware canvas with environment selector. Variable/context passing visualization. Tool/function attachment to agent nodes.
**Addresses:** All Phase 3 features from FEATURES.md (prompt editor, model selector, cost overlay, budget gauge, environments, variable visualization, tool attachment)
**Avoids:** Performance collapse (Pitfall 4 -- stress test at 200 nodes as exit criterion)

### Phase 4: Collaboration and Advanced Governance
**Rationale:** Deferred features that are high-value but not launch-critical. Each requires significant infrastructure (WebSocket presence, graph diff computation, evidence bundling) that should not delay the core product.
**Delivers:** Workflow versioning with visual diff view. Collaborative presence indicators. Governance evidence bundle export. Undo/redo (snapshot-based, not command-pattern). Template workflow library.
**Addresses:** All deferred features from FEATURES.md
**Avoids:** Undo/redo corruption (Pitfall 7 -- use snapshot approach, not command pattern)

### Phase Ordering Rationale

- **Phase 1 before everything:** The Graph Authoring API and canvas are the critical path. Every other feature depends on a working canvas with save/load. Dev infrastructure (Vite proxy, Docker build) must exist before any feature work.
- **Phase 2 before Phase 3:** Execution visualization validates the end-to-end loop (author -> run -> inspect results). Governance features are Zeroth's differentiator and should ship before authoring refinements.
- **Phase 3 before Phase 4:** Authoring power features (prompt editor, model selector) serve individual users. Collaboration features serve teams -- smaller audience initially.
- **WebSocket phasing:** REST-only in Phase 1. Read-only push in Phase 2. Bidirectional (if needed) in Phase 4. This avoids WebSocket complexity creep (Pitfall 9).

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Vue Flow integration patterns -- container sizing, custom node rendering, and the useCanvasMapping bridge between Zeroth Graph models and Vue Flow elements need careful prototyping
- **Phase 2:** WebSocket message protocol design -- topic-based routing, authentication on connect, reconnection semantics need a focused design spike
- **Phase 4:** Undo/redo implementation strategy -- snapshot vs. command pattern tradeoffs, undo boundaries, memory management need dedicated research

Phases with standard patterns (skip research-phase):
- **Phase 3:** All technologies are well-documented (CodeMirror 6, model selector dropdowns, budget display components). Standard CRUD + display patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified via npm April 2026. Compatibility matrix validated. No speculative choices. |
| Features | HIGH | Cross-referenced 6 competing platforms. Table stakes verified against n8n, Dify, Langflow, Flowise, Rivet, ComfyUI. Differentiators grounded in existing backend capabilities. |
| Architecture | HIGH | Feature-sliced Vue structure, Nginx reverse proxy, REST + WebSocket split all follow established patterns. n8n's architecture validates the approach without copying code. |
| Pitfalls | HIGH | Vue Flow container sizing, performance at scale, and undo/redo corruption verified against official docs and community reports. License risk verified against n8n SUL text. |

**Overall confidence:** HIGH

### Gaps to Address

- **OpenAPI-to-TypeScript generation pipeline:** The openapi-typescript workflow needs validation during Phase 1 setup. Verify it handles Zeroth's discriminated union types correctly.
- **Vue Flow custom node performance at scale:** The 50/100/200 node performance targets are based on React Flow benchmarks. Vue Flow may differ. Prototype and benchmark early in Phase 1.
- **Graph document size for WebSocket:** Large graphs (100+ nodes) transmitted as full JSON over WebSocket may need delta/patch compression. Assess during Phase 2.
- **RBAC integration with canvas interactions:** How granular should permission checks be? Per-node? Per-operation? Design work needed in Phase 2.
- **ky version pinning:** ky was recommended as "latest" without a pinned version. Pin at install time.

## Sources

### Primary (HIGH confidence)
- [npm package registry](https://www.npmjs.com/) -- version verification for all frontend packages
- [Vue Flow documentation](https://vueflow.dev/) -- container sizing, custom nodes, troubleshooting
- [React Flow performance guide](https://reactflow.dev/learn/advanced-use/performance) -- shared architecture performance patterns
- [FastAPI WebSocket docs](https://fastapi.tiangolo.com/advanced/websockets/) -- native WebSocket support
- [Tailwind CSS v4 docs](https://tailwindcss.com/) -- Vite plugin integration
- [Reka UI GitHub](https://github.com/unovue/reka-ui) -- headless component verification
- [n8n SUL license text](https://docs.n8n.io/sustainable-use-license/) -- license restriction verification
- Zeroth codebase and PROJECT.md -- backend capabilities verification

### Secondary (MEDIUM confidence)
- [n8n Architecture (DeepWiki)](https://deepwiki.com/n8n-io/n8n/) -- canvas architecture, command pattern, design system patterns
- [Feature-Sliced Design](https://feature-sliced.design/) -- frontend directory structure patterns
- [npm-compare component libraries](https://npm-compare.com/) -- download comparisons for UI library selection
- [Graph visualization UX guide (Cambridge Intelligence)](https://cambridge-intelligence.com/) -- progressive disclosure patterns

### Tertiary (LOW confidence)
- Community blog posts on FastAPI + Vue deployment -- patterns validated against official docs but specifics may vary

---
*Research completed: 2026-04-09*
*Ready for roadmap: yes*
