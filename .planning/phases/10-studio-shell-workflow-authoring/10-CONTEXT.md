# Phase 10: Studio Shell & Workflow Authoring - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning
**Source:** Brainstormed Studio design decisions + existing backend codebase map

<domain>
## Phase Boundary

Phase 10 establishes the first Studio authoring layer on top of the shipped backend runtime foundation. The scope is the Studio shell, workflow draft/revision foundations, canvas-first authoring posture, node-local contract/validation UX, and the backend/frontend boundaries needed to support those flows.

This phase does not deliver the full Studio product. It creates the first coherent authoring shell and the control-plane/draft model required to support later runtime, assets, and environment phases.

</domain>

<decisions>
## Implementation Decisions

### Product Shell
- Studio is canvas-first by default
- The left rail is workflows-only, organized under folders
- `Assets` appears as a secondary entry, not mixed into the workflow tree
- The center canvas is the dominant working surface
- A compact top mode switch exposes `Editor`, `Executions`, and `Tests`
- The right side is a contextual inspector, not a second app surface
- Header carries save state, environment selector, and publish/deploy controls

### Runtime And Governance UX
- Runtime/governance information must be progressively disclosed
- The editor should remain minimal by default
- Node selection shows config plus compact recent activity
- Deeper runtime records must be reachable by run and by node

### Authoring Model
- Agents, executable units, and memory resources are reusable assets instantiated as workflow nodes
- Contracts are authored in node-local/contextual flows rather than as a primary asset library
- Environments are first-class workspace resources but live in header/settings UX rather than in main nav

### Delivery Strategy
- Reuse existing backend runtime/admin/audit surfaces rather than rebuilding them
- Add a dedicated Studio/backend authoring layer instead of overloading the deployment-bound service wrapper
- Frontend should align with the design spec’s minimal, contextual-runtime shell

### the agent's Discretion
- Exact frontend repo structure so long as it supports a Vue 3 + Vite Studio app cleanly
- Exact API route/module breakdown inside the new Studio package
- Exact component/file boundaries for shell, rail, canvas, and inspector
- Exact validation presentation patterns provided they stay local, contextual, and minimal

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product And UX Direction
- `docs/superpowers/specs/2026-03-29-zeroth-studio-design.md` — validated Studio shell, navigation, asset, environment, and runtime UX decisions

### Existing Runtime And Service Foundation
- `src/zeroth/service/app.py` — deployment-bound FastAPI app factory and route registration
- `src/zeroth/service/bootstrap.py` — service wiring, durable worker, auth, metrics, repositories
- `src/zeroth/service/run_api.py` — run invocation/status API surface
- `src/zeroth/service/admin_api.py` — admin run operations and metrics surface
- `src/zeroth/orchestrator/runtime.py` — workflow execution orchestration

### Planning And Scope
- `.planning/PROJECT.md` — current product framing and project-level decisions
- `.planning/REQUIREMENTS.md` — mapped requirements for the Studio milestone
- `.planning/ROADMAP.md` — Phase 10 milestone scope and downstream dependencies
- `.planning/codebase/ARCHITECTURE.md` — current backend architecture map
- `.planning/codebase/CONCERNS.md` — portability and Studio-gap concerns

</canonical_refs>

<specifics>
## Specific Ideas

- The target feel is similar to n8n’s editor posture, but without code reuse
- The shell should be minimal by default, despite the product’s technical depth
- The future UI phase should lock spacing, typography, density, and interaction contracts before heavy frontend implementation
- Workflow drafts and revisions should be distinct from deployed runtime graph snapshots

</specifics>

<deferred>
## Deferred Ideas

- Full Studio runtime/test/governance views beyond the initial shell baseline
- Asset deep-edit workflows for agents, executable units, and memory resources
- Environment registry and deployment binding UX
- Broader workspace-level operational surfaces outside workflow-centric authoring

</deferred>

---

*Phase: 10-studio-shell-workflow-authoring*
*Context gathered: 2026-03-30 via existing Studio design decisions*
