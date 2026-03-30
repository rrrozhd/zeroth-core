# Zeroth

## What This Is

Zeroth is a governed medium-code platform for building, running, and deploying production-grade multi-agent systems as standalone API services. The current repository contains the backend/runtime foundation through durable execution, governance, security, and operations, and the next major product step is Zeroth Studio: a canvas-first authoring and control-plane UI layered on top of those runtime surfaces.

## Core Value

Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## Requirements

### Validated

- ✓ Governed workflow graph modeling, validation, and versioning exist — phases 1-1F
- ✓ Runtime orchestration, approvals, memory, and deployment-bound service APIs exist — phases 2-5
- ✓ Identity, governance evidence, runtime hardening, and durable control-plane foundations exist — phases 6-9

### Active

- [ ] Deliver a Studio authoring layer for workflows, assets, and environments
- [ ] Deliver a frontend shell that is canvas-first, minimal by default, and operationally navigable
- [ ] Expose runtime, audit, approval, evidence, and admin data through a Studio-oriented control-plane/gateway surface
- [ ] Preserve separation between reusable assets, node configuration, and environment-bound execution context

### Out of Scope

- Mobile apps — the immediate product gap is a web-based Studio, not native/mobile delivery
- Replacing the existing runtime/control-plane internals — Studio should layer on top of them, not fork them
- Forking or transplanting n8n editor code — only product patterns should be reused

## Context

The repository is a mature Python backend with domain packages under `src/zeroth/`, broad pytest coverage under `tests/`, and phase-oriented planning/evidence documents in `phases/` and `PROGRESS.md`. There is no existing frontend app in the repo yet. A Studio design spec already exists at `docs/superpowers/specs/2026-03-29-zeroth-studio-design.md`, defining the intended UX model: canvas-first authoring, quiet workflow rail, top mode switch, contextual runtime surfaces, assets as reusable building blocks, and environment management in the header/settings layer.

## Constraints

- **Tech stack**: Existing backend is Python/FastAPI/Pydantic — new Studio work should integrate with, not replace, this foundation
- **Architecture**: Studio is a gateway + authoring layer — existing deployment/service runtime remains source of truth for execution
- **UX**: Minimal by default — runtime/governance depth must be progressively disclosed rather than always visible
- **Product semantics**: Zeroth-specific concepts must remain intact — agent, executable unit, memory resource, approvals, evidence, attestation, environments
- **Dependency portability**: Local GovernAI path dependency currently exists — environment assumptions should be treated carefully during planning

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Studio is canvas-first | Authoring is the product’s primary interaction, not operations monitoring | — Pending |
| Left rail is workflows-only with `Assets` as a secondary entry | Keeps the default shell minimal and focused | — Pending |
| Runtime records must be reachable by run and by node | Governance data is meaningful in both scopes | — Pending |
| Contracts are authored in node context, not as a primary asset library | They are tightly coupled to authoring and connection logic | — Pending |
| Environments live in the header/settings layer | They are cross-cutting operational context, not everyday navigation items | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

After each phase transition:
1. Requirements invalidated move to Out of Scope with reason
2. Requirements validated move to Validated with phase reference
3. New requirements are added to Active
4. Significant decisions are logged in the table above
5. The "What This Is" section is updated if product reality drifts

---
*Last updated: 2026-03-30 after GSD initialization*
