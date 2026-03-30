# Zeroth Codebase Integrations

## Overview

The current codebase is mostly self-contained, but it already integrates with several external-facing concerns: a local GovernAI runtime dependency, HTTP auth infrastructure, SQLite and Redis storage, and secret-bearing execution contexts.

## Core Runtime Dependency

- GovernAI is consumed via the local path dependency in [`pyproject.toml`](`pyproject.toml`)
- This is the most important external code dependency in the current system
- The Zeroth codebase appears to wrap or align with GovernAI abstractions rather than re-implementing all runtime behavior from scratch

## HTTP Service Surface

The service layer exposes deployment-bound HTTP APIs through FastAPI:

- [`src/zeroth/service/run_api.py`](`src/zeroth/service/run_api.py`) — public run submission and status
- [`src/zeroth/service/approval_api.py`](`src/zeroth/service/approval_api.py`) — human approval interactions
- [`src/zeroth/service/contracts_api.py`](`src/zeroth/service/contracts_api.py`) — contract metadata surfaces
- [`src/zeroth/service/audit_api.py`](`src/zeroth/service/audit_api.py`) — audit, timeline, evidence, attestation surfaces
- [`src/zeroth/service/admin_api.py`](`src/zeroth/service/admin_api.py`) — admin controls and metrics

These APIs are internally integrated through the deployment bootstrap in [`src/zeroth/service/bootstrap.py`](`src/zeroth/service/bootstrap.py`).

## Authentication And Authorization

External auth-related integrations include:

- Static API key credentials via [`src/zeroth/service/auth.py`](`src/zeroth/service/auth.py`)
- JWT/JWKS bearer verification support in [`src/zeroth/service/auth.py`](`src/zeroth/service/auth.py`)
- Role and scope enforcement via [`src/zeroth/service/authorization.py`](`src/zeroth/service/authorization.py`)

Potential external providers or integration points:

- JWKS URL for bearer auth
- signed JWT issuers
- tenant/workspace identity boundaries

## Storage Integrations

- SQLite is the main persisted storage system via [`src/zeroth/storage/sqlite.py`](`src/zeroth/storage/sqlite.py`)
- Redis-related storage hooks exist in [`src/zeroth/storage/redis.py`](`src/zeroth/storage/redis.py`)
- JSON-based storage helper exists in [`src/zeroth/storage/json.py`](`src/zeroth/storage/json.py`)

Phase 9 durable control work suggests Redis may matter for dispatch/control-plane scenarios even though SQLite remains the evident default persistence layer.

## Secret And Credential Handling

The codebase expects secret-bearing configurations to exist in runtime contexts:

- secret provider interfaces in [`src/zeroth/secrets/provider.py`](`src/zeroth/secrets/provider.py`)
- redaction helpers in [`src/zeroth/secrets/redaction.py`](`src/zeroth/secrets/redaction.py`)
- encrypted-field helper in [`src/zeroth/storage/sqlite.py`](`src/zeroth/storage/sqlite.py`)

This matters for future Studio work because environment management will likely sit above these lower-level primitives.

## Deployment And Environment-Like Integration Points

Deployment snapshots and runtime binding information live in:

- [`src/zeroth/deployments/models.py`](`src/zeroth/deployments/models.py`)
- [`src/zeroth/deployments/service.py`](`src/zeroth/deployments/service.py`)
- [`src/zeroth/deployments/provenance.py`](`src/zeroth/deployments/provenance.py`)

Current deployment references already carry tenant/workspace metadata and pinned graph/contract versions, which will be important integration points for a future Studio gateway.

## Human Workflow Integrations

The codebase supports human-in-the-loop operations through:

- approval service and repository in `src/zeroth/approvals/`
- audit/evidence/attestation in `src/zeroth/audit/`
- admin operations such as cancel, replay, and interrupt in [`src/zeroth/service/admin_api.py`](`src/zeroth/service/admin_api.py`)

These are not third-party SaaS integrations by themselves, but they are external-facing control surfaces meant to be consumed by an operator UI or client.

## Test And Scenario Integrations

- Live scenario coverage exists under [`tests/live_scenarios/test_research_audit.py`](`tests/live_scenarios/test_research_audit.py`)
- Service integration helpers live in [`tests/service/helpers.py`](`tests/service/helpers.py`)
- The repo includes `live_scenarios/README.md`, suggesting local research-audit scenario setup

## Not Found During Mapping

- No frontend analytics, Sentry, Stripe, or cloud SDK integrations
- No Docker Compose or Kubernetes manifests in the root files inspected
- No explicit external database services besides Redis support
- No checked-in OpenAPI client generation or frontend API client package

## Practical Summary

Today’s integrations are mostly backend-runtime and auth/storage oriented. The next major integration surface will likely be a Studio UI and authoring gateway layered over the existing deployment, runtime, audit, approval, and admin APIs rather than replacing them.
