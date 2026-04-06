---
phase: 12-real-llm-providers-retry
plan: 03
subsystem: agent-runtime, audit
tags: [token-usage, audit-trail, unit-tests, integration-tests, litellm, retry]
dependency_graph:
  requires: [12-01, 12-02]
  provides: [token-audit-trail, adapter-test-suite, retry-test-suite, live-test-suite]
  affects: [audit, agent-runtime]
tech_stack:
  added: []
  patterns: [pytest-mark-live, mock-based-adapter-testing]
key_files:
  created:
    - tests/test_litellm_adapter.py
    - tests/test_retry_backoff.py
    - tests/test_live_llm.py
  modified:
    - src/zeroth/audit/models.py
    - src/zeroth/agent_runtime/runner.py
    - pyproject.toml
decisions:
  - Added addopts to pyproject.toml to exclude live tests by default
  - Registered live pytest marker in pyproject.toml markers config
metrics:
  duration: 296s
  completed: "2026-04-06T21:53:45Z"
  tasks: 3
  files: 6
---

# Phase 12 Plan 03: Token Audit Trail & Test Suite Summary

NodeAuditRecord wired with token_usage field populated from ProviderResponse; 19 unit tests for LiteLLM adapter and retry/backoff; 3 live integration tests gated behind @pytest.mark.live marker.

## What Was Done

### Task 1: Wire token_usage into NodeAuditRecord and AgentRunner audit flow
- Added `token_usage: TokenUsage | None = None` field to `NodeAuditRecord` in `src/zeroth/audit/models.py`, placed between `execution_metadata` and `error`
- Added token_usage copy logic in `AgentRunner.run()` after `serialize_record()` call: when `response.token_usage` is not None, serializes it to the audit record dict via `model_dump(mode="json")`
- **Commit:** b6e2e8d

### Task 2: Create unit tests for adapter, retry, and token flow
- Created `tests/test_litellm_adapter.py` with 6 tests: content return, token extraction, no-usage handling, PromptMessage conversion, metadata provider info, fallback response_metadata extraction
- Created `tests/test_retry_backoff.py` with 13 tests: 7 error classification tests (rate limit, service unavailable, timeout, internal server error retryable; auth error, bad request, generic exception not retryable) and 6 backoff computation tests (base delay, doubling, quadrupling, max cap, jitter range, attempt-1 zero)
- All 19 tests pass in 0.02s with no network calls
- **Commit:** 1e05a92

### Task 3: Create live integration tests and register pytest marker
- Created `tests/test_live_llm.py` with 3 tests: `test_openai_live_call`, `test_anthropic_live_call`, `test_token_usage_flows_to_response`
- All gated with `pytestmark = pytest.mark.live` and individual `@pytest.mark.skipif` for API key presence
- Registered `live` marker in `pyproject.toml` `[tool.pytest.ini_options]` to suppress UnknownMarkWarning
- Added `addopts = "-m 'not live'"` so default `uv run pytest` excludes live tests
- **Commit:** e703a89

## Deviations from Plan

None - plan executed exactly as written.

## Decisions Made

1. **Live marker in pyproject.toml**: Registered the `live` marker and `addopts` filter in `pyproject.toml` rather than `conftest.py` since pytest config was already centralized there.
2. **Extra test added**: Added `test_ainvoke_fallback_response_metadata` beyond the plan's specified tests to cover the OpenAI-style `response_metadata.token_usage` fallback path.

## Known Stubs

None - all data flows are wired end-to-end.

## Verification Results

- `uv run pytest tests/test_litellm_adapter.py tests/test_retry_backoff.py -v`: 19 passed
- `uv run pytest tests/test_live_llm.py --collect-only -o "addopts="`: 3 tests collected
- `uv run python -c "from zeroth.audit.models import NodeAuditRecord; assert 'token_usage' in NodeAuditRecord.model_fields"`: OK
- `uv run ruff check src/ tests/test_litellm_adapter.py tests/test_retry_backoff.py tests/test_live_llm.py`: All checks passed

## Self-Check: PASSED

All 6 files verified present. All 3 commits verified in git log.
