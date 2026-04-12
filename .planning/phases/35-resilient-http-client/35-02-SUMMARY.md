---
phase: 35-resilient-http-client
plan: 02
subsystem: http-client-governance
tags: [http, governance, capability-gating, audit, auth-resolution, bootstrap-wiring]
dependency_graph:
  requires: [35-01, zeroth.core.http, zeroth.core.secrets, zeroth.core.policy, zeroth.core.config]
  provides: [http-client-bootstrap-wiring, http-governance-integration-tests]
  affects: [zeroth.core.service.bootstrap, zeroth.core.orchestrator.runtime]
tech_stack:
  added: []
  patterns: [env-secret-provider-reuse, lazy-import-in-bootstrap, mock-transport-testing]
key_files:
  created:
    - tests/http/test_governance_integration.py
  modified:
    - src/zeroth/core/service/bootstrap.py
    - src/zeroth/core/orchestrator/runtime.py
decisions:
  - Reuse EnvSecretProvider with os.environ for HTTP auth resolution, matching existing SecretResolver pattern in execution_units runner
  - Lazy imports for ResilientHttpClient and EnvSecretProvider inside bootstrap_service() to avoid circular import chains
  - Place http_client field after artifact_store in both RuntimeOrchestrator and ServiceBootstrap dataclasses for chronological phase ordering
metrics:
  duration_seconds: 371
  completed: "2026-04-12T23:08:23Z"
  tasks_completed: 2
  tasks_total: 2
  tests_added: 14
  files_created: 1
  files_modified: 2
---

# Phase 35 Plan 02: Governance Integration & Bootstrap Wiring Summary

ResilientHttpClient wired into ServiceBootstrap and RuntimeOrchestrator with EnvSecretProvider for auth resolution, plus 14 integration tests verifying capability gating, audit record accumulation, auth header injection, rate limiting, and the full governance pipeline end-to-end.

## What Was Built

### Bootstrap Wiring (`src/zeroth/core/service/bootstrap.py`)
- Added `http_client` field to `ServiceBootstrap` dataclass after `artifact_store`
- Added Phase 35 construction block in `bootstrap_service()` that creates `ResilientHttpClient` with `HttpClientSettings` from `ZerothSettings.http_client` and `EnvSecretProvider(os.environ)` for runtime secret resolution
- Wires `http_client_instance` to `orchestrator.http_client` and includes it in the `ServiceBootstrap` return constructor
- Uses lazy imports (`import os`, `from zeroth.core.http import ResilientHttpClient`, `from zeroth.core.secrets import EnvSecretProvider`) to avoid circular import chains

### Orchestrator Field (`src/zeroth/core/orchestrator/runtime.py`)
- Added `http_client: Any | None = None` field to `RuntimeOrchestrator` dataclass after `artifact_store`, making the HTTP client available for downstream agent tool access

### Integration Tests (`tests/http/test_governance_integration.py`)
- **Capability gating (4 tests):** GET succeeds with NETWORK_READ+EXTERNAL_API_CALL; GET fails with only NETWORK_WRITE; POST succeeds with NETWORK_WRITE+EXTERNAL_API_CALL; None capabilities skip check (backward compat)
- **Audit record accumulation (3 tests):** URL redaction strips query params; records capture method/status/latency/size; drain clears records on call
- **Auth resolution (3 tests):** Bearer token injected as `Authorization: Bearer <value>`; API key injected on custom header name; no auth header when no secret_key
- **Rate limiting (1 test):** Token bucket rejects when burst exhausted with low refill rate
- **Multiple records (1 test):** Sequential requests accumulate independent HttpCallRecord entries
- **Full pipeline (1 test):** Auth + capability + audit all work together in single request
- **Bootstrap wiring (1 test):** ZerothSettings has http_client field of type HttpClientSettings

All tests use `httpx.MockTransport` for deterministic HTTP mocking with `MockSecretProvider` dataclass for auth resolution.

## Deviations from Plan

None - plan executed exactly as written.

## Task Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Bootstrap wiring and orchestrator field | b78436c | bootstrap.py, runtime.py |
| 2 | Governance integration tests | ee66c52 | test_governance_integration.py |

## Verification Results

- `uv run pytest tests/http/test_governance_integration.py -v` -- 14/14 passed
- `uv run pytest tests/http/ tests/artifacts/ tests/mappings/ tests/conditions/` -- 226/226 passed
- `uv run ruff check src/zeroth/core/http/ src/zeroth/core/service/ src/zeroth/core/orchestrator/` -- all clean
- Bootstrap constructs ResilientHttpClient and wires to orchestrator
- Integration tests prove capability gating, audit records, auth resolution, rate limiting

## Self-Check: PASSED
