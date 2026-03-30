# Zeroth Codebase Conventions

## General Style

The codebase follows a clean, typed Python style with explicit module boundaries and a preference for small focused files per concern.

Common traits across `src/zeroth/`:

- `from __future__ import annotations` at the top of modules
- Pydantic models for API and domain schemas
- dataclasses for plain containers and helper structures
- explicit type annotations throughout
- concise module docstrings explaining responsibility

## File-Level Patterns

A recurring pattern is one concern per file:

- `models.py` for typed models
- `repository.py` for persistence access
- `service.py` for domain behavior
- `*_api.py` for HTTP route registration

This makes the code predictable to navigate.

## Validation And API Modeling

- FastAPI request/response payloads are typed with Pydantic models
- `ConfigDict(extra="forbid")` is commonly used to reject unexpected fields
- domain validation and HTTP validation are generally kept close to boundary code
- contract-driven validation is a first-class concept in [`src/zeroth/contracts/registry.py`](`src/zeroth/contracts/registry.py`) and [`src/zeroth/service/run_api.py`](`src/zeroth/service/run_api.py`)

## Error Handling

- boundary layers raise `HTTPException` for API failures
- lower layers often raise `ValueError` or domain-specific exceptions
- defensive `pragma: no cover` blocks exist around branches that are difficult or unnecessary to drive in tests
- comments tend to explain intent, especially around compatibility, guardrails, or lifecycle edges

## Dependency Wiring

- dependencies are composed explicitly in [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`)
- protocols are used where a narrow bootstrap contract is sufficient, e.g. in service APIs
- `app.state.bootstrap` is used instead of a larger inversion-of-control container

## Naming Conventions

- package names are domain-oriented: `runs`, `audit`, `conditions`, `guardrails`
- class names are descriptive and often suffixed by responsibility, e.g. `RunWorker`, `QuotaEnforcer`, `ApprovalService`
- enum-like HTTP statuses use `StrEnum`, e.g. `RunPublicStatus`
- test helper names are descriptive rather than abbreviated, e.g. `default_service_auth_config`

## Persistence And Repository Conventions

- repository code is separate from model definitions
- SQLite schema migration/version tracking is centralized in [`src/zeroth/storage/sqlite.py`](`src/zeroth/storage/sqlite.py`)
- direct SQL exists but is wrapped in structured helpers or repositories rather than spread across the app

## Testability Conventions

- lightweight fake or deterministic runners are built as simple dataclasses in [`tests/service/helpers.py`](`tests/service/helpers.py`)
- helpers centralize common setup for deployment/service test scenarios
- tests tend to verify behavior through the public API or repository surface instead of relying only on internal implementation details

## Documentation Conventions

- phase plans live in `phases/phase-*/PLAN.md`
- evidence files live in `phases/phase-*/artifacts/`
- implementation status is tracked in [`PROGRESS.md`](`PROGRESS.md`)

## Practical Summary

The codebase values explicitness, type safety, predictable file roles, and testable composition. New code should fit those patterns rather than introducing highly implicit frameworks or broad “misc utilities” modules.
