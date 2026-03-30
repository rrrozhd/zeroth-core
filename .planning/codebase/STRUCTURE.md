# Zeroth Codebase Structure

## Root Layout

Important top-level paths:

- [`src/`](`src/`) — main Python package source
- [`tests/`](`tests/`) — pytest suites mirroring backend subsystems
- [`docs/`](`docs/`) — spec and planning-oriented documents
- [`phases/`](`phases/`) — execution-phase plans and artifacts
- [`live_scenarios/`](`live_scenarios/`) — local scenario support/readme
- [`pyproject.toml`](`pyproject.toml`) — package/tooling definition
- [`uv.lock`](`uv.lock`) — locked dependency set for `uv`
- [`PROGRESS.md`](`PROGRESS.md`) — implementation progress log
- [`AGENTS.md`](`AGENTS.md`) — repo-specific agent instructions

## Main Source Tree

All application code currently lives under [`src/zeroth/`](`src/zeroth/`).

Top-level package subdirectories:

- [`src/zeroth/graph/`](`src/zeroth/graph/`)
- [`src/zeroth/contracts/`](`src/zeroth/contracts/`)
- [`src/zeroth/mappings/`](`src/zeroth/mappings/`)
- [`src/zeroth/runs/`](`src/zeroth/runs/`)
- [`src/zeroth/agent_runtime/`](`src/zeroth/agent_runtime/`)
- [`src/zeroth/execution_units/`](`src/zeroth/execution_units/`)
- [`src/zeroth/conditions/`](`src/zeroth/conditions/`)
- [`src/zeroth/memory/`](`src/zeroth/memory/`)
- [`src/zeroth/approvals/`](`src/zeroth/approvals/`)
- [`src/zeroth/audit/`](`src/zeroth/audit/`)
- [`src/zeroth/deployments/`](`src/zeroth/deployments/`)
- [`src/zeroth/dispatch/`](`src/zeroth/dispatch/`)
- [`src/zeroth/guardrails/`](`src/zeroth/guardrails/`)
- [`src/zeroth/observability/`](`src/zeroth/observability/`)
- [`src/zeroth/service/`](`src/zeroth/service/`)
- [`src/zeroth/storage/`](`src/zeroth/storage/`)
- [`src/zeroth/secrets/`](`src/zeroth/secrets/`)
- [`src/zeroth/identity/`](`src/zeroth/identity/`)
- [`src/zeroth/policy/`](`src/zeroth/policy/`)
- [`src/zeroth/orchestrator/`](`src/zeroth/orchestrator/`)

## Service Layer Structure

The service package is split by HTTP concern:

- [`src/zeroth/service/app.py`](`src/zeroth/service/app.py`) — app factory and route registration
- [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`) — dependency wiring
- [`src/zeroth/service/run_api.py`](`src/zeroth/service/run_api.py`) — run routes
- [`src/zeroth/service/approval_api.py`](`src/zeroth/service/approval_api.py`) — approval routes
- [`src/zeroth/service/audit_api.py`](`src/zeroth/service/audit_api.py`) — audit/evidence routes
- [`src/zeroth/service/contracts_api.py`](`src/zeroth/service/contracts_api.py`) — contract routes
- [`src/zeroth/service/admin_api.py`](`src/zeroth/service/admin_api.py`) — admin/metrics routes
- [`src/zeroth/service/auth.py`](`src/zeroth/service/auth.py`) — authentication logic
- [`src/zeroth/service/authorization.py`](`src/zeroth/service/authorization.py`) — permission/scoping helpers

## Test Structure

Tests largely mirror the source layout by subsystem:

- [`tests/graph/`](`tests/graph/`)
- [`tests/contracts/`](`tests/contracts/`)
- [`tests/execution_units/`](`tests/execution_units/`)
- [`tests/agent_runtime/`](`tests/agent_runtime/`)
- [`tests/service/`](`tests/service/`)
- [`tests/dispatch/`](`tests/dispatch/`)
- [`tests/audit/`](`tests/audit/`)
- [`tests/policy/`](`tests/policy/`)

There are also cross-cutting or special suites:

- [`tests/live_scenarios/`](`tests/live_scenarios/`)
- [`tests/test_smoke.py`](`tests/test_smoke.py`)

## Documentation And Planning Structure

- [`docs/specs/`](`docs/specs/`) contains implementation-facing specs for completed phases
- [`docs/superpowers/specs/`](`docs/superpowers/specs/`) contains the new Studio design spec
- [`docs/superpowers/plans/`](`docs/superpowers/plans/`) exists as a planning bucket
- [`phases/phase-*/PLAN.md`](`phases/`) contains execution-phase plans
- [`phases/phase-*/artifacts/`](`phases/`) stores verification outputs

## Naming Patterns

- backend modules are generally singular by concern (`registry.py`, `service.py`, `models.py`, `repository.py`)
- tests typically use `test_<subsystem>.py`
- package names reflect bounded domains instead of generic `utils`

## Missing Structural Areas

Not present today:

- no `.planning/` project scaffolding before this mapping run
- no frontend app or `apps/` directory
- no dedicated `studio` package under `src/zeroth/`
- no top-level deployment/dev server scripts beyond Python toolchain usage

## Practical Summary

The structure is disciplined and backend-centric. It is already segmented well enough that a future Studio authoring layer can likely be added as a new bounded area rather than forcing a broad repo reorganization.
