# Requirements: Zeroth

**Defined:** 2026-03-30
**Core Value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## v1 Requirements

### Runtime Foundation

- [ ] **RUN-01**: User can execute deployed governed workflows through stable service APIs
- [ ] **RUN-02**: User can review run status, approvals, audits, evidence, and attestation for deployed workflows
- [ ] **RUN-03**: Operator can interrupt, cancel, and replay runs through authorized admin controls

### Authoring Studio

- [ ] **STU-01**: User can open a canvas-first Studio interface for authoring workflows
- [x] **STU-02**: User can create and edit workflow drafts separately from deployed runtime graphs
- [ ] **STU-03**: User can publish and deploy authored workflows to named environments
- [ ] **STU-04**: User can inspect workflow execution in both run-level and node-level contexts

### Assets And Configuration

- [ ] **AST-01**: User can manage reusable agent definitions as assets referenced by workflow nodes
- [ ] **AST-02**: User can manage reusable executable units as assets referenced by workflow nodes
- [ ] **AST-03**: User can manage reusable memory resources as assets referenced by workflow nodes
- [ ] **AST-04**: User can configure contracts in node-local authoring flows rather than through a detached contract library
- [ ] **AST-05**: User can manage environment-bound secrets and bindings separately from workflow structure

### UX And Governance

- [ ] **UX-01**: Studio is minimal by default, with progressive disclosure for governance and runtime detail
- [ ] **UX-02**: Workflows are navigable through a foldered left rail and contextual editor shell
- [ ] **UX-03**: Runtime and governance data are accessible without turning the editor into a heavy operations dashboard

## v2 Requirements

### Expansion

- **EXP-01**: User can collaborate on the same workflow draft with richer multi-session editing ergonomics
- **EXP-02**: User can use project-unit authoring flows on top of the same Studio asset foundation
- **EXP-03**: User can access broader workspace-level analytics and management views beyond workflow-centric authoring

## Out of Scope

| Feature | Reason |
|---------|--------|
| Mobile-native Studio clients | Web Studio is the immediate gap and highest-leverage surface |
| n8n code reuse/forking | Product patterns can be borrowed, implementation should remain Zeroth-native |
| Rebuilding runtime/admin backends from scratch for Studio | Existing runtime/control-plane surfaces already exist and should be reused |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| RUN-01 | Phase 1-9 | Complete |
| RUN-02 | Phase 1-9 | Complete |
| RUN-03 | Phase 9 | Complete |
| STU-01 | Phase 10 | Pending |
| STU-02 | Phase 10 | Complete |
| STU-03 | Phase 11 | Pending |
| STU-04 | Phase 11 | Pending |
| AST-01 | Phase 12 | Pending |
| AST-02 | Phase 12 | Pending |
| AST-03 | Phase 12 | Pending |
| AST-04 | Phase 10 | Pending |
| AST-05 | Phase 13 | Pending |
| UX-01 | Phase 10 | Pending |
| UX-02 | Phase 10 | Pending |
| UX-03 | Phase 11 | Pending |

**Coverage:**
- v1 requirements: 15 total
- Mapped to phases: 15
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-30*
*Last updated: 2026-03-30 after GSD initialization*
