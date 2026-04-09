# Requirements: Zeroth Studio

**Defined:** 2026-04-09
**Core Value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## v2.0 Requirements

Requirements for Zeroth Studio visual workflow authoring UI. Each maps to roadmap phases.

### Canvas Foundation

- [ ] **CANV-01**: User can drag nodes from palette onto a pannable, zoomable canvas
- [ ] **CANV-02**: User can draw edges between typed node ports to create connections
- [ ] **CANV-03**: User can browse and search available node types in a categorized sidebar palette
- [ ] **CANV-04**: User can view and edit selected node properties in an inspector panel
- [ ] **CANV-05**: User can auto-layout the graph in a readable DAG arrangement
- [x] **CANV-06**: User can save and load workflow graphs via the authoring API
- [ ] **CANV-07**: User can undo and redo canvas operations (node add/move/delete, edge add/remove)
- [ ] **CANV-08**: User can use keyboard shortcuts for common operations (delete, select-all, copy/paste, duplicate)
- [ ] **CANV-09**: User can navigate the canvas with pan, zoom, fit-to-view, and minimap
- [x] **CANV-10**: User can work in a responsive three-panel layout (rail, canvas, inspector) with collapsible panels

### Graph Authoring API

- [x] **API-01**: Studio can create, read, update, and delete workflow graphs via REST endpoints
- [ ] **API-02**: Studio can receive real-time updates via WebSocket (execution status, validation, presence)
- [ ] **API-03**: Studio can retrieve available node type schemas with field definitions and validation rules
- [ ] **API-04**: Studio can trigger workflow execution and receive per-node status updates

### Governance Visualization

- [ ] **GOV-01**: User can see approval gate nodes with live status (pending/approved/rejected/escalated), SLA countdown, and who-approved attribution
- [ ] **GOV-02**: User can view per-node audit trail (governance evidence, modification history, compliance status) in the inspector
- [ ] **GOV-03**: User can see sandbox indicator badges on execution unit nodes showing isolation mode and resource constraints
- [ ] **GOV-04**: Canvas enforces RBAC — viewers see read-only, operators can run but not edit, authors have full access
- [ ] **GOV-05**: User can see token cost and usage as badges/tooltips on agent nodes after execution
- [ ] **GOV-06**: User can see remaining tenant budget as a gauge in the Studio header
- [ ] **GOV-07**: User can see and switch the target environment (dev/staging/prod) with visual differentiation

### AI Authoring

- [ ] **AUTH-01**: User can select LLM model/provider per agent node from available models
- [ ] **AUTH-02**: User can edit prompts and system messages in a CodeMirror editor with syntax highlighting
- [ ] **AUTH-03**: User can attach tools and execution units to agent nodes as distinct connection types
- [ ] **AUTH-04**: User can see data flow between nodes via typed port labels and hover tooltips
- [ ] **AUTH-05**: User can trigger workflow execution and view per-node results (input/output/tokens/cost)
- [ ] **AUTH-06**: User can see node validation indicators (missing fields, invalid connections, type mismatches)

### Versioning & Collaboration

- [ ] **COLLAB-01**: User can view side-by-side diff of graph versions showing added/removed/modified nodes and edges
- [ ] **COLLAB-02**: User can see which other users are viewing/editing the same workflow (presence indicators)

### Deployment & Infrastructure

- [x] **INFRA-01**: Studio frontend is served via Nginx alongside the existing FastAPI backend in the Docker deployment
- [ ] **INFRA-02**: Frontend types are generated from the backend OpenAPI spec to prevent type drift

## v2.1 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Export & Templates

- **FUTURE-01**: Governance evidence bundle export (one-click compliance artifact)
- **FUTURE-02**: Template workflow library
- **FUTURE-03**: Workflow import/export (JSON)

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| 500+ integration nodes (n8n-style) | Zeroth's value is governed AI workflows, not integration breadth |
| Visual RAG pipeline builder | Memory connectors handle retrieval; ingestion is user's responsibility |
| Real-time LLM streaming on canvas | Async step-completion model; PROJECT.md explicitly excludes streaming |
| Marketplace / community nodes | Product in itself — discovery, quality control, versioning, security |
| Mobile-responsive canvas | Graph editors are desktop experiences; min viewport width enforced |
| AI-generated workflow suggestions | Research project, not v2.0 feature |
| Embedded chat/test panel | Zeroth is not a chatbot builder; test via Run button with configurable inputs |
| No-code promise | Zeroth is medium-code; CodeMirror for prompts and code where it matters |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| CANV-01 | Phase 22 | Pending |
| CANV-02 | Phase 22 | Pending |
| CANV-03 | Phase 23 | Pending |
| CANV-04 | Phase 23 | Pending |
| CANV-05 | Phase 23 | Pending |
| CANV-06 | Phase 22 | Complete |
| CANV-07 | Phase 23 | Pending |
| CANV-08 | Phase 23 | Pending |
| CANV-09 | Phase 22 | Pending |
| CANV-10 | Phase 22 | Complete |
| API-01 | Phase 22 | Complete |
| API-02 | Phase 24 | Pending |
| API-03 | Phase 22 | Pending |
| API-04 | Phase 24 | Pending |
| GOV-01 | Phase 25 | Pending |
| GOV-02 | Phase 25 | Pending |
| GOV-03 | Phase 25 | Pending |
| GOV-04 | Phase 25 | Pending |
| GOV-05 | Phase 25 | Pending |
| GOV-06 | Phase 25 | Pending |
| GOV-07 | Phase 25 | Pending |
| AUTH-01 | Phase 24 | Pending |
| AUTH-02 | Phase 24 | Pending |
| AUTH-03 | Phase 24 | Pending |
| AUTH-04 | Phase 24 | Pending |
| AUTH-05 | Phase 24 | Pending |
| AUTH-06 | Phase 23 | Pending |
| COLLAB-01 | Phase 26 | Pending |
| COLLAB-02 | Phase 26 | Pending |
| INFRA-01 | Phase 22 | Complete |
| INFRA-02 | Phase 22 | Pending |

**Coverage:**
- v2.0 requirements: 31 total
- Mapped to phases: 31
- Unmapped: 0

---
*Requirements defined: 2026-04-09*
*Last updated: 2026-04-09 after roadmap creation*
