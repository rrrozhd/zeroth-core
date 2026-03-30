# Zeroth Codebase Concerns

## 1. Local GovernAI Path Dependency

The most obvious portability concern is the local path dependency in [`pyproject.toml`](`pyproject.toml`):

- `governai @ file:///Users/dondoe/coding/governai`

This ties environment setup to one machine layout and can break onboarding, CI, or verification on another machine unless the dependency path is recreated or replaced.

## 2. Missing Top-Level Product Documentation

[`README.md`](`README.md`) is effectively empty. That increases onboarding cost for both humans and tooling because the repo has rich internal phase documents but little top-level product/setup orientation.

## 3. Deployment-Bound Service Shape

The existing FastAPI app in [`src/zeroth/service/app.py`](`src/zeroth/service/app.py`) is deployment-scoped. That is appropriate for the current runtime wrapper, but it is narrower than what a future Studio control plane needs.

Likely gap areas:

- workspace-wide authoring APIs
- draft/revision lifecycle APIs
- edit leases and concurrency control
- environment registry and authoring-time asset management

This is not a bug, but it is a major architectural expansion point.

## 4. No Frontend Or Studio Foundation Yet

The repo has no frontend package, no Node/Vite workspace, and no `src/zeroth/studio/` package yet. The planned Studio effort is therefore not an incremental polish task; it is a substantial new surface area.

## 5. Bootstrap Surface Growth

[`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`) already wires many dependencies:

- deployment service
- run/thread repositories
- approvals
- audit
- contract registry
- orchestrator
- auth
- dispatch
- guardrails
- metrics

As new control-plane or Studio concerns are added, this file could become a pressure point unless new bounded bootstrap layers are introduced.

## 6. Direct Repository Internals In Admin API

[`src/zeroth/service/admin_api.py`](`src/zeroth/service/admin_api.py`) reaches into repository internals (`bootstrap.run_repository._store.database.transaction()`) during replay. That works, but it weakens encapsulation and suggests some admin operations may need a more explicit service abstraction.

## 7. Environment And Secret UX Gap

Backend primitives for auth, secret handling, deployment metadata, and encrypted fields exist, but there is no user-facing environment management surface yet. This will matter for Studio because environment-bound secrets and bindings are central to the intended product model.

## 8. Docs And Planning Systems Are Split

The repo already uses:

- [`PROGRESS.md`](`PROGRESS.md`)
- `phases/phase-*/PLAN.md`
- `docs/specs/`
- `docs/superpowers/specs/`

Adding GSD `.planning/` introduces a second planning/documentation system. This may be fine, but it creates a process concern: future contributors need clarity on which planning layer is authoritative for which purpose.

## 9. Limited External Runtime Reproducibility Signals

The files inspected do not show:

- container/devcontainer setup
- CI workflows in `.github/`
- explicit production packaging/deployment manifests

The backend may still be runnable locally, but reproducibility and CI visibility are not obvious from repo root.

## 10. Frontend Planning Risk

Because there is no frontend code yet, there is risk of jumping directly into implementation without:

- workspace/package structure decisions
- UI testing strategy
- API client conventions
- auth/session model for Studio

The new Studio design spec helps, but the actual plan phase still needs to translate those decisions into concrete repo structure before coding begins.

## Practical Summary

The current backend is well-factored and well-tested, but the biggest risks are around portability, missing top-level onboarding docs, and the size of the upcoming jump from deployment-bound backend to a full authoring/control-plane plus frontend product.
