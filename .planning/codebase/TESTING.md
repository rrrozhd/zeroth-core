# Zeroth Codebase Testing

## Test Framework

- pytest is the main test framework
- async tests use `pytest-asyncio`
- pytest configuration is in [`pyproject.toml`](`pyproject.toml`) with `testpaths = ["tests"]` and `asyncio_mode = "auto"`

## Test Layout

Tests mirror the backend subsystem layout closely.

Examples:

- graph: [`tests/graph/`](`tests/graph/`)
- contracts: [`tests/contracts/`](`tests/contracts/`)
- execution units: [`tests/execution_units/`](`tests/execution_units/`)
- service APIs: [`tests/service/`](`tests/service/`)
- dispatch: [`tests/dispatch/`](`tests/dispatch/`)
- observability: [`tests/observability/`](`tests/observability/`)

This mirroring makes it easy to find verification for a package by name.

## Test Granularity

The repo contains a mix of:

- unit tests for model, repository, and utility behavior
- service/API tests for FastAPI routes and auth behavior
- integration-style tests covering phase-level and end-to-end flows
- live-scenario tests under [`tests/live_scenarios/`](`tests/live_scenarios/`)

Examples of broader suites:

- [`tests/service/test_e2e_phase4.py`](`tests/service/test_e2e_phase4.py`)
- [`tests/service/test_e2e_phase5.py`](`tests/service/test_e2e_phase5.py`)
- [`tests/live_scenarios/test_research_audit.py`](`tests/live_scenarios/test_research_audit.py`)

## Test Helpers And Fakes

Common service setup is centralized in [`tests/service/helpers.py`](`tests/service/helpers.py`).

Patterns used there:

- deterministic fake runners
- bootstrap helpers for deployment/service setup
- auth header helpers for operator/reviewer/admin roles
- local Pydantic test payload models

This suggests a convention of building small focused test doubles rather than a heavyweight mocking framework.

## Coverage Shape

Based on the file tree, the backend coverage is broad across:

- domain modeling
- storage backends
- service auth/RBAC
- audit/evidence
- durable dispatch
- runtime policy enforcement
- sandbox hardening

Areas not represented yet:

- frontend/UI tests
- browser E2E for a web app
- component-level or visual regression tests

## Verification Commands

Repo guidance from [`AGENTS.md`](`AGENTS.md`) points to:

- `uv sync`
- `uv run pytest -v`
- `uv run ruff check src/`
- `uv run ruff format src/`

This indicates an expected loop of install → test → lint → format.

## Phase-Based Evidence

The project also stores verification artifacts outside pytest itself:

- `phases/phase-*/artifacts/*.txt`

These act as durable evidence for phase completion and broader validation sweeps. This is a stronger discipline than relying on transient local terminal output only.

## Practical Summary

Testing is a strong part of the current backend culture. Any future Studio/frontend work will need to introduce a parallel testing stack for UI behavior, because the current repository has no browser/component test infrastructure yet.
