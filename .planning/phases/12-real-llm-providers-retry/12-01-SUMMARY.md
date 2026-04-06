---
phase: 12-real-llm-providers-retry
plan: 01
subsystem: agent-runtime
tags: [litellm, langchain, chatllm, token-usage, pydantic, llm-adapter]

requires:
  - phase: 11-config-postgres-storage
    provides: async database, pydantic-settings config
provides:
  - LiteLLMProviderAdapter class implementing ProviderAdapter protocol via ChatLiteLLM
  - TokenUsage pydantic model for token consumption tracking
  - ProviderResponse.token_usage field for passing token data through the runtime
affects: [12-02 retry, 12-03 audit-integration, 13-regulus-instrumentation]

tech-stack:
  added: [litellm>=1.83, langchain-litellm>=0.3.4, tenacity>=8.2]
  patterns: [ChatLiteLLM wrapper per model string, client caching, dual token extraction (usage_metadata + response_metadata)]

key-files:
  created: []
  modified:
    - pyproject.toml
    - src/zeroth/audit/models.py
    - src/zeroth/agent_runtime/provider.py

key-decisions:
  - "Used ChatLiteLLM per D-01 rather than raw litellm.acompletion for GovernAI LangChain compatibility"
  - "Dual token extraction strategy: usage_metadata primary, response_metadata.token_usage fallback"
  - "Client caching per model string to reuse connections across calls"

patterns-established:
  - "LiteLLM model format strings (openai/gpt-4o, anthropic/claude-*) for provider routing"
  - "TokenUsage as standalone pydantic model reusable across ProviderResponse and NodeAuditRecord"

requirements-completed: [LLM-01, LLM-02, LLM-04]

duration: 3min
completed: 2026-04-07
---

# Phase 12 Plan 01: LiteLLM Provider Adapter & Token Usage Summary

**LiteLLMProviderAdapter routes to 100+ LLM providers via ChatLiteLLM with TokenUsage extraction from AIMessage metadata**

## Performance

- **Duration:** 3 min
- **Started:** 2026-04-06T21:40:40Z
- **Completed:** 2026-04-06T21:43:49Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Installed litellm (v1.83.0, verified not compromised), langchain-litellm, and tenacity dependencies
- Created TokenUsage pydantic model with input_tokens, output_tokens, total_tokens, model_name fields
- Built LiteLLMProviderAdapter with message conversion, token extraction, and tool call normalization
- Added token_usage field to ProviderResponse as non-breaking optional addition
- All 280 existing tests continue passing

## Task Commits

Each task was committed atomically:

1. **Task 1: Install dependencies and create TokenUsage model** - `d4b5d59` (feat)
2. **Task 2: Create LiteLLMProviderAdapter** - `c10eb96` (feat)

## Files Created/Modified
- `pyproject.toml` - Added litellm>=1.83,<2.0, langchain-litellm>=0.3.4, tenacity>=8.2
- `src/zeroth/audit/models.py` - Added TokenUsage pydantic model before NodeAuditRecord
- `src/zeroth/agent_runtime/provider.py` - Added LiteLLMProviderAdapter class, TokenUsage import, token_usage field on ProviderResponse

## Decisions Made
- Used ChatLiteLLM (LangChain wrapper) per D-01 user-locked decision, not raw litellm.acompletion
- Dual token extraction: usage_metadata (LangChain standard) with response_metadata.token_usage fallback
- Client instances cached per model string in _clients dict for connection reuse

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- litellm no longer exposes `__version__` attribute; used `importlib.metadata.version()` instead for version verification

## User Setup Required

None - no external service configuration required. API keys (OPENAI_API_KEY, ANTHROPIC_API_KEY) are needed at runtime but not for installation.

## Next Phase Readiness
- LiteLLMProviderAdapter ready for Plan 02 to wrap with retry logic (tenacity already installed)
- TokenUsage ready for Plan 03 to wire into NodeAuditRecord
- All imports clean, ruff checks pass, 280 tests green

---
*Phase: 12-real-llm-providers-retry*
*Completed: 2026-04-07*

## Self-Check: PASSED

- [x] pyproject.toml exists with new dependencies
- [x] src/zeroth/audit/models.py exists with TokenUsage class
- [x] src/zeroth/agent_runtime/provider.py exists with LiteLLMProviderAdapter
- [x] Commit d4b5d59 found (Task 1)
- [x] Commit c10eb96 found (Task 2)
- [x] 280 tests passing
- [x] ruff check clean
