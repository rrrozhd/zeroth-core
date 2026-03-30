# Roadmap: Zeroth

## Milestones

- ✅ **v1.0 Runtime Foundation** - Phases 1-9 (backend/runtime foundation complete)
- 🚧 **v2.0 Zeroth Studio** - Phases 10-13 (planned)

## Phases

<details>
<summary>✅ v1.0 Runtime Foundation (Phases 1-9) - SHIPPED 2026-03-27</summary>

### Phase 1: Core Foundation
**Goal**: Establish graph, contract, mapping, run, validation, and versioning foundations.
**Depends on**: Nothing (first phase)
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Graphs, contracts, mappings, and runs have typed persisted foundations
  2. Validation and versioning support safe authoring/runtime progression
**Plans**: Complete

Plans:
- [x] 01-01: Domain models and graph schema
- [x] 01-02: Contract registry, mappings, validation, and versioning

### Phase 2: Execution Core
**Goal**: Deliver orchestration, agent runtime, execution units, conditions, and thread persistence.
**Depends on**: Phase 1
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Governed workflows execute end-to-end with agents, units, and branching
  2. Thread continuity and checkpoints persist across runs
**Plans**: Complete

Plans:
- [x] 02-01: Execution unit and agent runtime foundations
- [x] 02-02: Orchestration, conditions, tools, and thread handling

### Phase 3: Platform Control
**Goal**: Add memory connectors, approvals, and audit-aware control surfaces.
**Depends on**: Phase 2
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. Memory, approvals, and control interactions work through persisted platform models
  2. Governance-relevant actions are captured in audit flows
**Plans**: Complete

Plans:
- [x] 03-01: Memory and approval lifecycle
- [x] 03-02: Approval API and audit integration

### Phase 4: Deployment Surface
**Goal**: Expose published deployments through a service wrapper.
**Depends on**: Phase 3
**Requirements**: RUN-01
**Success Criteria** (what must be TRUE):
  1. Published graphs can be deployed and invoked through HTTP service APIs
  2. Deployment bootstrap and service routes work against pinned snapshots
**Plans**: Complete

Plans:
- [x] 04-01: Deployment and service wrapper

### Phase 5: Integration & Polish
**Goal**: Verify end-to-end behavior and document implementation-facing specs.
**Depends on**: Phase 4
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. End-to-end service flows are validated
  2. MVP runtime foundation is documented and shippable
**Plans**: Complete

Plans:
- [x] 05-01: Integration verification and specs

### Phase 6: Identity & Tenant Governance
**Goal**: Add service authentication, RBAC, and tenant/workspace scoping.
**Depends on**: Phase 5
**Requirements**: RUN-02
**Success Criteria** (what must be TRUE):
  1. Authenticated access and scope enforcement protect service surfaces
  2. Identity lineage is visible in runtime/audit flows
**Plans**: Complete

Plans:
- [x] 06-01: Authentication, RBAC, and tenant isolation

### Phase 7: Transparent Governance & Provenance
**Goal**: Expose audit, evidence, and attestation surfaces.
**Depends on**: Phase 6
**Requirements**: RUN-02
**Success Criteria** (what must be TRUE):
  1. Users can inspect public audit and evidence surfaces
  2. Provenance and attestation are verifiable
**Plans**: Complete

Plans:
- [x] 07-01: Audit, evidence, and attestation surfaces

### Phase 8: Runtime Security Hardening
**Goal**: Harden sandboxing, policy enforcement, and secret protection.
**Depends on**: Phase 7
**Requirements**: RUN-01, RUN-02
**Success Criteria** (what must be TRUE):
  1. Runtime execution is hardened against unsafe execution paths
  2. Secret handling and policy enforcement are production-oriented
**Plans**: Complete

Plans:
- [x] 08-01: Runtime security hardening

### Phase 9: Durable Control Plane & Production Operations
**Goal**: Make dispatch, recovery, guardrails, metrics, and admin controls durable.
**Depends on**: Phase 8
**Requirements**: RUN-01, RUN-02, RUN-03
**Success Criteria** (what must be TRUE):
  1. Run dispatch and recovery survive process turnover
  2. Operators can observe and control runs safely
**Plans**: Complete

Plans:
- [x] 09-01: Durable dispatch, guardrails, observability, and admin controls

</details>

### 🚧 v2.0 Zeroth Studio (Planned)

**Milestone Goal:** Deliver the authoring, asset, environment, and execution UX needed to turn the backend runtime foundation into a full Studio product.

### Phase 10: Studio Shell & Workflow Authoring
**Goal**: Establish the Studio shell, canvas-first navigation, workflow drafts, and authoring-time contracts/validation UX.
**Depends on**: Phase 9
**Requirements**: STU-01, STU-02, AST-04, UX-01, UX-02
**Success Criteria** (what must be TRUE):
  1. User can open a Studio shell with workflow rail, canvas, inspector, and mode switch
  2. User can manage workflow drafts separately from deployed runtime graphs
  3. Authoring validation and contract configuration work in node-local flows
**Plans**: 6 plans

Plans:
- [ ] 10-01-PLAN.md — Studio workflow and lease persistence foundations
- [ ] 10-02-PLAN.md — Studio backend bootstrap, auth scope enforcement, and workflow or lease APIs
- [ ] 10-03-PLAN.md — Frontend Studio workspace scaffold and typed API contracts
- [ ] 10-04-PLAN.md — Canvas-first shell composition and workflow navigation baseline
- [ ] 10-05-PLAN.md — Scope-aware draft save, validation, and slash-safe contract lookup APIs
- [ ] 10-06-PLAN.md — Lease-aware frontend autosave and node-local validation or contract UX

### Phase 11: Studio Runtime, Executions, And Testing
**Goal**: Add execution timelines, test runs, and runtime/gateway views to Studio.
**Depends on**: Phase 10
**Requirements**: STU-03, STU-04, UX-03
**Success Criteria** (what must be TRUE):
  1. User can run draft tests from Studio against persisted authoring snapshots
  2. User can inspect runtime data by run and by node
  3. Studio reuses existing runtime, audit, approval, and admin surfaces through a gateway layer
**Plans**: 3 plans

Plans:
- [ ] 11-01: Studio runtime gateway and query normalization
- [ ] 11-02: Executions and tests views
- [ ] 11-03: Node-scoped and run-scoped governance UX

### Phase 12: Studio Assets
**Goal**: Add reusable asset authoring for agents, executable units, and memory resources.
**Depends on**: Phase 10
**Requirements**: AST-01, AST-02, AST-03
**Success Criteria** (what must be TRUE):
  1. User can browse and select reusable assets from Studio
  2. Asset definitions can be edited separately from workflow node instances
  3. Asset workflows preserve canvas context by default
**Plans**: 2 plans

Plans:
- [ ] 12-01: Asset models and backend persistence
- [ ] 12-02: Asset slide-over UX and deep-edit flows

### Phase 13: Environments & Deployment UX
**Goal**: Add environment management and deployment-time bindings for Studio.
**Depends on**: Phase 11, Phase 12
**Requirements**: AST-05
**Success Criteria** (what must be TRUE):
  1. User can switch current environment from the Studio header
  2. User can manage environment-bound secrets and bindings safely
  3. Publish/deploy flows use named environments as first-class configuration
**Plans**: 2 plans

Plans:
- [ ] 13-01: Environment registry and secret/binding management
- [ ] 13-02: Header environment UX and deploy integration

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
| 10. Studio Shell & Workflow Authoring | v2.0 | 0/4 | Not started | - |
| 11. Studio Runtime, Executions, And Testing | v2.0 | 0/3 | Not started | - |
| 12. Studio Assets | v2.0 | 0/2 | Not started | - |
| 13. Environments & Deployment UX | v2.0 | 0/2 | Not started | - |
