---
phase: 13-regulus-economics-integration
plan: 01
subsystem: econ
tags: [regulus, cost-tracking, litellm, instrumentation, pydantic]

# Dependency graph
requires:
  - phase: 12-real-llm-providers-retry
    provides: ProviderAdapter protocol, ProviderResponse with token_usage, NodeAuditRecord with TokenUsage
provides:
  - src/zeroth/econ/ module with RegulusClient, CostEstimator, InstrumentedProviderAdapter
  - RegulusSettings integrated into ZerothSettings
  - ProviderResponse and NodeAuditRecord cost_usd/cost_event_id fields
  - Alembic migration 002 for cost columns on node_audits
affects: [13-02, 13-03, budget-enforcement, audit-pipeline]

# Tech tracking
tech-stack:
  added: [econ-instrumentation-sdk, cachetools]
  patterns: [decorator-adapter for cross-cutting concerns, fire-and-forget telemetry, Decimal-based cost tracking]

key-files:
  created:
    - src/zeroth/econ/__init__.py
    - src/zeroth/econ/models.py
    - src/zeroth/econ/client.py
    - src/zeroth/econ/cost.py
    - src/zeroth/econ/adapter.py
    - src/zeroth/migrations/versions/002_add_cost_fields.py
    - tests/test_econ_models.py
    - tests/test_econ_adapter.py
  modified:
    - pyproject.toml
    - src/zeroth/config/settings.py
    - src/zeroth/agent_runtime/provider.py
    - src/zeroth/audit/models.py

key-decisions:
  - "Lazy import for InstrumentedProviderAdapter in __init__.py to avoid circular imports"
  - "CostEstimator wraps litellm.cost_per_token with try/except returning Decimal(0) for unknown models"
  - "RegulusClient.stop() calls flush_once() then stop() on transport for clean shutdown"

patterns-established:
  - "Decorator adapter pattern: InstrumentedProviderAdapter wraps ProviderAdapter without modifying protocol"
  - "Cost attribution flow: adapter estimates cost, emits event, enriches response for downstream audit"

requirements-completed: [ECON-01, ECON-02]

# Metrics
duration: 6min
completed: 2026-04-07
---

# Phase 13 Plan 01: Regulus Economics Foundation Summary

**Regulus SDK integration with InstrumentedProviderAdapter emitting cost events per LLM call, litellm-based cost estimation, and cost attribution fields on ProviderResponse/NodeAuditRecord**

## Performance

- **Duration:** 6 min (346s)
- **Started:** 2026-04-07T07:35:15Z
- **Completed:** 2026-04-07T07:41:01Z
- **Tasks:** 2
- **Files modified:** 13

## Accomplishments
- Created src/zeroth/econ/ module with full public API: RegulusClient, CostEstimator, InstrumentedProviderAdapter
- InstrumentedProviderAdapter wraps any ProviderAdapter, measures latency, estimates USD cost via litellm, emits Regulus ExecutionEvent, and enriches response
- Added cost_usd and cost_event_id fields to both ProviderResponse and NodeAuditRecord for end-to-end cost attribution
- Integrated RegulusSettings into ZerothSettings with env-var support (ZEROTH_REGULUS__ prefix)
- Created Alembic migration 002 adding cost columns to node_audits table
- 13 unit tests passing (7 for models/config, 6 for adapter)

## Task Commits

Each task was committed atomically (TDD: test -> feat):

1. **Task 1: SDK dependency, econ module with models, client, cost estimator, config** - `9b37a59` (test) -> `1d3e806` (feat)
2. **Task 2: InstrumentedProviderAdapter with unit tests** - `fe4a2cc` (test) -> `5389a3e` (feat)

## Files Created/Modified
- `src/zeroth/econ/__init__.py` - Public API exports for econ module
- `src/zeroth/econ/models.py` - RegulusSettings and CostAttribution pydantic models
- `src/zeroth/econ/client.py` - RegulusClient wrapping InstrumentationClient
- `src/zeroth/econ/cost.py` - CostEstimator using litellm.cost_per_token
- `src/zeroth/econ/adapter.py` - InstrumentedProviderAdapter decorator
- `src/zeroth/migrations/versions/002_add_cost_fields.py` - Alembic migration for cost columns
- `src/zeroth/config/settings.py` - Added regulus sub-model to ZerothSettings
- `src/zeroth/agent_runtime/provider.py` - Added cost_usd, cost_event_id to ProviderResponse
- `src/zeroth/audit/models.py` - Added cost_usd, cost_event_id to NodeAuditRecord
- `pyproject.toml` - Added econ-instrumentation-sdk and cachetools dependencies
- `tests/test_econ_models.py` - 7 tests for models, config, client, cost estimator
- `tests/test_econ_adapter.py` - 6 tests for InstrumentedProviderAdapter

## Decisions Made
- Lazy import for InstrumentedProviderAdapter in econ/__init__.py to avoid circular imports (adapter imports from provider.py)
- CostEstimator wraps litellm.cost_per_token in try/except, returns Decimal("0") for unknown models
- RegulusClient.stop() calls flush_once() then stop() for clean transport shutdown (per Regulus SDK pitfall)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- Pre-existing test failure in tests/live_scenarios/test_research_audit.py (timeout waiting for run status) -- unrelated to this plan's changes, all 126 other tests pass.

## User Setup Required

None - econ-instrumentation-sdk is referenced as a local path dependency. Regulus backend connection is disabled by default (enabled=False).

## Next Phase Readiness
- InstrumentedProviderAdapter ready to be wired into orchestrator node execution (Plan 02)
- Budget enforcement can use RegulusSettings.budget_cache_ttl and cachetools (installed here)
- Cost attribution fields available on ProviderResponse and NodeAuditRecord for audit pipeline

## Self-Check: PASSED

All 8 created files verified. All 4 commit hashes verified.

---
*Phase: 13-regulus-economics-integration*
*Completed: 2026-04-07*
