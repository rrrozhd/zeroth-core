# Roadmap: Zeroth

## Milestones

- ✅ **v1.0 Runtime Foundation** — Phases 1-9 (shipped 2026-03-27)
- ✅ **v1.1 Production Readiness** — Phases 11-21 (shipped 2026-04-09)
- 📋 **v2.0 Zeroth Studio** — Phases 22-25 (planned)

## Phases

<details>
<summary>✅ v1.0 Runtime Foundation (Phases 1-9) — SHIPPED 2026-03-27</summary>

- [x] Phase 1: Core Foundation (2/2 plans) — completed 2026-03-19
- [x] Phase 2: Execution Core (2/2 plans) — completed 2026-03-19
- [x] Phase 3: Platform Control (2/2 plans) — completed 2026-03-19
- [x] Phase 4: Deployment Surface (1/1 plan) — completed 2026-03-20
- [x] Phase 5: Integration & Polish (1/1 plan) — completed 2026-03-26
- [x] Phase 6: Identity & Tenant Governance (1/1 plan) — completed 2026-03-27
- [x] Phase 7: Transparent Governance & Provenance (1/1 plan) — completed 2026-03-27
- [x] Phase 8: Runtime Security Hardening (1/1 plan) — completed 2026-03-27
- [x] Phase 9: Durable Control Plane & Production Operations (1/1 plan) — completed 2026-03-27

</details>

<details>
<summary>✅ v1.1 Production Readiness (Phases 11-21) — SHIPPED 2026-04-09</summary>

- [x] Phase 11: Config & Postgres Storage (3/3 plans) — completed 2026-04-06
- [x] Phase 12: Real LLM Providers & Retry (3/3 plans) — completed 2026-04-06
- [x] Phase 13: Regulus Economics Integration (3/3 plans) — completed 2026-04-07
- [x] Phase 14: Memory Connectors & Container Sandbox (5/5 plans) — completed 2026-04-07
- [x] Phase 15: Webhooks & Approval SLA (3/3 plans) — completed 2026-04-07
- [x] Phase 16: Distributed Dispatch & Horizontal Scaling (3/3 plans) — completed 2026-04-07
- [x] Phase 17: Deployment Packaging & Operations (3/3 plans) — completed 2026-04-07
- [x] Phase 18: Cross-Phase Integration Wiring (2/2 plans) — completed 2026-04-08
- [x] Phase 19: Agent Node LLM API Parity (3/3 plans) — completed 2026-04-08
- [x] Phase 20: Bootstrap Integration Wiring (1/1 plan) — completed 2026-04-09
- [x] Phase 21: Health Probe Fix & Tech Debt (1/1 plan) — completed 2026-04-09

</details>

### 📋 v2.0 Zeroth Studio (Planned)

**Milestone Goal:** Deliver the authoring, asset, environment, and execution UX needed to turn the backend runtime foundation into a full Studio product.

### Phase 22: Studio Shell & Workflow Authoring
**Goal**: Establish the Studio shell, canvas-first navigation, workflow drafts, and authoring-time contracts/validation UX.
**Depends on**: Phase 21
**Requirements**: STU-01, STU-02, AST-04, UX-01, UX-02
**Success Criteria** (what must be TRUE):
  1. User can open a Studio shell with workflow rail, canvas, inspector, and mode switch
  2. User can manage workflow drafts separately from deployed runtime graphs
  3. Authoring validation and contract configuration work in node-local flows
**Plans**: 3 plans
**UI hint**: yes

Plans:
- [ ] 22-01: Studio backend session, draft, revision, and lease foundations
- [ ] 22-02: Frontend shell, routing, and canvas/inspector baseline
- [ ] 22-03: Validation, contract-authoring UX, and autosave boundaries

### Phase 23: Studio Runtime, Executions, And Testing
**Goal**: Add execution timelines, test runs, and runtime/gateway views to Studio.
**Depends on**: Phase 22
**Requirements**: STU-03, STU-04, UX-03
**Success Criteria** (what must be TRUE):
  1. User can run draft tests from Studio against persisted authoring snapshots
  2. User can inspect runtime data by run and by node
  3. Studio reuses existing runtime, audit, approval, and admin surfaces through a gateway layer
**Plans**: 3 plans
**UI hint**: yes

Plans:
- [ ] 23-01: Studio runtime gateway and query normalization
- [ ] 23-02: Executions and tests views
- [ ] 23-03: Node-scoped and run-scoped governance UX

### Phase 24: Studio Assets
**Goal**: Add reusable asset authoring for agents, executable units, and memory resources.
**Depends on**: Phase 22
**Requirements**: AST-01, AST-02, AST-03
**Success Criteria** (what must be TRUE):
  1. User can browse and select reusable assets from Studio
  2. Asset definitions can be edited separately from workflow node instances
  3. Asset workflows preserve canvas context by default
**Plans**: 2 plans
**UI hint**: yes

Plans:
- [ ] 24-01: Asset models and backend persistence
- [ ] 24-02: Asset slide-over UX and deep-edit flows

### Phase 25: Environments & Deployment UX
**Goal**: Add environment management and deployment-time bindings for Studio.
**Depends on**: Phase 23, Phase 24
**Requirements**: AST-05
**Success Criteria** (what must be TRUE):
  1. User can switch current environment from the Studio header
  2. User can manage environment-bound secrets and bindings safely
  3. Publish/deploy flows use named environments as first-class configuration
**Plans**: 2 plans
**UI hint**: yes

Plans:
- [ ] 25-01: Environment registry and secret/binding management
- [ ] 25-02: Header environment UX and deploy integration

## Progress

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
| 22. Studio Shell & Workflow Authoring | v2.0 | 0/3 | Not started | - |
| 23. Studio Runtime, Executions, And Testing | v2.0 | 0/3 | Not started | - |
| 24. Studio Assets | v2.0 | 0/2 | Not started | - |
| 25. Environments & Deployment UX | v2.0 | 0/2 | Not started | - |
