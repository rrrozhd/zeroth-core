# Roadmap: Zeroth

## Milestones

- v1.0 Runtime Foundation -- Phases 1-9 (shipped 2026-03-27)
- v1.1 Production Readiness -- Phases 11-21 (shipped 2026-04-09)
- v2.0 Zeroth Studio -- Phases 22-26 (in progress)

## Phases

<details>
<summary>v1.0 Runtime Foundation (Phases 1-9) -- SHIPPED 2026-03-27</summary>

- [x] Phase 1: Core Foundation (2/2 plans) -- completed 2026-03-19
- [x] Phase 2: Execution Core (2/2 plans) -- completed 2026-03-19
- [x] Phase 3: Platform Control (2/2 plans) -- completed 2026-03-19
- [x] Phase 4: Deployment Surface (1/1 plan) -- completed 2026-03-20
- [x] Phase 5: Integration & Polish (1/1 plan) -- completed 2026-03-26
- [x] Phase 6: Identity & Tenant Governance (1/1 plan) -- completed 2026-03-27
- [x] Phase 7: Transparent Governance & Provenance (1/1 plan) -- completed 2026-03-27
- [x] Phase 8: Runtime Security Hardening (1/1 plan) -- completed 2026-03-27
- [x] Phase 9: Durable Control Plane & Production Operations (1/1 plan) -- completed 2026-03-27

</details>

<details>
<summary>v1.1 Production Readiness (Phases 11-21) -- SHIPPED 2026-04-09</summary>

- [x] Phase 11: Config & Postgres Storage (3/3 plans) -- completed 2026-04-06
- [x] Phase 12: Real LLM Providers & Retry (3/3 plans) -- completed 2026-04-06
- [x] Phase 13: Regulus Economics Integration (3/3 plans) -- completed 2026-04-07
- [x] Phase 14: Memory Connectors & Container Sandbox (5/5 plans) -- completed 2026-04-07
- [x] Phase 15: Webhooks & Approval SLA (3/3 plans) -- completed 2026-04-07
- [x] Phase 16: Distributed Dispatch & Horizontal Scaling (3/3 plans) -- completed 2026-04-07
- [x] Phase 17: Deployment Packaging & Operations (3/3 plans) -- completed 2026-04-07
- [x] Phase 18: Cross-Phase Integration Wiring (2/2 plans) -- completed 2026-04-08
- [x] Phase 19: Agent Node LLM API Parity (3/3 plans) -- completed 2026-04-08
- [x] Phase 20: Bootstrap Integration Wiring (1/1 plan) -- completed 2026-04-09
- [x] Phase 21: Health Probe Fix & Tech Debt (1/1 plan) -- completed 2026-04-09

</details>

### v2.0 Zeroth Studio (In Progress)

**Milestone Goal:** Build a visual workflow authoring UI for governed multi-agent systems, using Vue 3 + Vue Flow, with governance-first canvas patterns reimplemented for Zeroth's domain model.

- [ ] **Phase 22: Canvas Foundation & Dev Infrastructure** - Vue app scaffold, basic canvas with node/edge editing, graph authoring REST API, Docker/Nginx serving
- [ ] **Phase 23: Canvas Editing UX** - Palette sidebar, inspector panel, auto-layout, undo/redo, keyboard shortcuts, validation indicators
- [ ] **Phase 24: Execution & AI Authoring** - WebSocket real-time updates, workflow execution trigger, per-node results, model selector, prompt editor, tool attachment
- [ ] **Phase 25: Governance Visualization** - Approval gates, audit trail, sandbox badges, RBAC-aware canvas, cost/budget display, environment switching
- [ ] **Phase 26: Versioning & Collaboration** - Graph version diff view, collaborative presence indicators

## Phase Details

### Phase 22: Canvas Foundation & Dev Infrastructure
**Goal**: Users can create workflow graphs by placing nodes and drawing edges on an interactive canvas, save and load them via the API, and work within a responsive three-panel Studio layout served from Docker
**Depends on**: Phase 21 (v1.1 shipped)
**Requirements**: CANV-01, CANV-02, CANV-06, CANV-09, CANV-10, API-01, API-03, INFRA-01, INFRA-02
**Success Criteria** (what must be TRUE):
  1. User can drag nodes onto a pannable, zoomable canvas and draw edges between typed node ports
  2. User can save a workflow graph and reload it in a new browser session with all nodes and edges intact
  3. User can navigate the canvas with pan, zoom, fit-to-view, and minimap
  4. User sees a responsive three-panel layout (workflow rail, canvas, inspector) with collapsible panels
  5. Studio frontend is served via Nginx alongside FastAPI in Docker, and frontend types are generated from the backend OpenAPI spec
**Plans**: TBD
**UI hint**: yes

### Phase 23: Canvas Editing UX
**Goal**: Users have a complete editing experience with categorized node palette, property inspector, auto-layout, undo/redo, keyboard shortcuts, and validation feedback
**Depends on**: Phase 22
**Requirements**: CANV-03, CANV-04, CANV-05, CANV-07, CANV-08, AUTH-06
**Success Criteria** (what must be TRUE):
  1. User can browse and search node types in a categorized sidebar palette
  2. User can select a node and view/edit its properties in the inspector panel
  3. User can auto-layout the graph into a readable DAG arrangement
  4. User can undo and redo canvas operations, and use keyboard shortcuts for common actions (delete, select-all, copy/paste, duplicate)
  5. User can see validation indicators on nodes with missing fields, invalid connections, or type mismatches
**Plans**: TBD
**UI hint**: yes

### Phase 24: Execution & AI Authoring
**Goal**: Users can trigger workflow execution, see real-time per-node status updates via WebSocket, configure agent nodes with model/provider selection, prompt editing, tool attachment, and inspect data flow between nodes
**Depends on**: Phase 23
**Requirements**: API-02, API-04, AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. User can trigger workflow execution and see real-time per-node status badges (running, complete, failed) via WebSocket
  2. User can view per-node execution results including input, output, token count, and cost
  3. User can select an LLM model/provider per agent node and edit prompts in a syntax-highlighted CodeMirror editor
  4. User can attach tools and execution units to agent nodes as distinct connection types
  5. User can see data flow between nodes via typed port labels and hover tooltips
**Plans**: TBD
**UI hint**: yes

### Phase 25: Governance Visualization
**Goal**: Users can see Zeroth's governance layer on the canvas -- approval gate status with SLA timers, per-node audit trails, sandbox indicators, RBAC-enforced access levels, token cost attribution, budget gauges, and environment switching
**Depends on**: Phase 24
**Requirements**: GOV-01, GOV-02, GOV-03, GOV-04, GOV-05, GOV-06, GOV-07
**Success Criteria** (what must be TRUE):
  1. User can see approval gate nodes with live status (pending/approved/rejected/escalated), SLA countdown, and who-approved attribution
  2. User can view per-node audit trail and governance evidence in the inspector
  3. Canvas enforces RBAC -- viewers see read-only, operators can run but not edit, authors have full access
  4. User can see token cost badges on agent nodes after execution and a tenant budget gauge in the Studio header
  5. User can see and switch the target environment (dev/staging/prod) with visual differentiation, and see sandbox indicator badges on execution unit nodes
**Plans**: TBD
**UI hint**: yes

### Phase 26: Versioning & Collaboration
**Goal**: Users can compare graph versions side-by-side and see which other users are viewing or editing the same workflow
**Depends on**: Phase 24
**Requirements**: COLLAB-01, COLLAB-02
**Success Criteria** (what must be TRUE):
  1. User can view a side-by-side diff of graph versions showing added, removed, and modified nodes and edges
  2. User can see presence indicators showing which other users are viewing or editing the same workflow
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 22 -> 23 -> 24 -> 25 -> 26

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Core Foundation | v1.0 | 2/2 | Complete | 2026-03-19 |
| 2. Execution Core | v1.0 | 2/2 | Complete | 2026-03-19 |
| 3. Platform Control | v1.0 | 2/2 | Complete | 2026-03-19 |
| 4. Deployment Surface | v1.0 | 1/1 | Complete | 2026-03-20 |
| 5. Integration & Polish | v1.0 | 1/1 | Complete | 2026-03-26 |
| 6. Identity & Tenant Governance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 7. Transparent Governance & Provenance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 8. Runtime Security Hardening | v1.0 | 1/1 | Complete | 2026-03-27 |
| 9. Durable Control Plane & Production Operations | v1.0 | 1/1 | Complete | 2026-03-27 |
| 11. Config & Postgres Storage | v1.1 | 3/3 | Complete | 2026-04-06 |
| 12. Real LLM Providers & Retry | v1.1 | 3/3 | Complete | 2026-04-06 |
| 13. Regulus Economics Integration | v1.1 | 3/3 | Complete | 2026-04-07 |
| 14. Memory Connectors & Container Sandbox | v1.1 | 5/5 | Complete | 2026-04-07 |
| 15. Webhooks & Approval SLA | v1.1 | 3/3 | Complete | 2026-04-07 |
| 16. Distributed Dispatch & Horizontal Scaling | v1.1 | 3/3 | Complete | 2026-04-07 |
| 17. Deployment Packaging & Operations | v1.1 | 3/3 | Complete | 2026-04-07 |
| 18. Cross-Phase Integration Wiring | v1.1 | 2/2 | Complete | 2026-04-08 |
| 19. Agent Node LLM API Parity | v1.1 | 3/3 | Complete | 2026-04-08 |
| 20. Bootstrap Integration Wiring | v1.1 | 1/1 | Complete | 2026-04-09 |
| 21. Health Probe Fix & Tech Debt | v1.1 | 1/1 | Complete | 2026-04-09 |
| 22. Canvas Foundation & Dev Infrastructure | v2.0 | 0/? | Not started | - |
| 23. Canvas Editing UX | v2.0 | 0/? | Not started | - |
| 24. Execution & AI Authoring | v2.0 | 0/? | Not started | - |
| 25. Governance Visualization | v2.0 | 0/? | Not started | - |
| 26. Versioning & Collaboration | v2.0 | 0/? | Not started | - |
