---
phase: 19-agent-node-llm-api-parity
plan: 01
subsystem: api
tags: [litellm, pydantic, openai, function-calling, structured-output, model-params]

# Dependency graph
requires:
  - phase: 12-real-llm-providers-retry
    provides: LiteLLMProviderAdapter, ProviderRequest/Response, ChatLiteLLM integration
provides:
  - ModelParams class for per-node LLM parameter control
  - Extended ProviderRequest with tools, tool_choice, response_format, model_params
  - ToolAttachmentManifest.to_openai_tool() conversion method
  - AgentNodeData model_params and mcp_servers fields
  - LiteLLMProviderAdapter kwargs forwarding for all new fields
affects: [19-02, 19-03]

# Tech tracking
tech-stack:
  added: []
  patterns: [kwargs-forwarding-via-ainvoke, openai-tool-format-conversion]

key-files:
  created:
    - tests/test_provider_request_extensions.py
    - tests/test_litellm_adapter_forwarding.py
  modified:
    - src/zeroth/agent_runtime/provider.py
    - src/zeroth/agent_runtime/tools.py
    - src/zeroth/agent_runtime/models.py
    - src/zeroth/graph/models.py

key-decisions:
  - "ModelParams defined in models.py (not provider.py) to avoid circular import, re-exported via provider.py"

patterns-established:
  - "kwargs forwarding: build dict conditionally, pass via **kwargs to ainvoke, never via constructor"
  - "OpenAI tool format conversion: to_openai_tool() on manifest produces standard function-calling dict"

requirements-completed: [API-01, API-02, API-03]

# Metrics
duration: 8min
completed: 2026-04-08
---

# Phase 19 Plan 01: Core Models and Provider Adapter Extensions Summary

**ModelParams, ProviderRequest tool/format/params fields, ToolAttachmentManifest.to_openai_tool(), and LiteLLMProviderAdapter kwargs forwarding for native LLM API parity**

## Performance

- **Duration:** 8 min (490s)
- **Started:** 2026-04-08T10:48:46Z
- **Completed:** 2026-04-08T10:56:56Z
- **Tasks:** 2
- **Files modified:** 6

## Accomplishments
- Extended all core data models (ProviderRequest, AgentConfig, AgentNodeData, ToolAttachmentManifest) with native LLM API fields
- Added ModelParams class for per-node temperature, top_p, max_tokens, stop, seed control
- Implemented to_openai_tool() conversion on ToolAttachmentManifest for function-calling format
- Updated LiteLLMProviderAdapter.ainvoke() to forward tools, tool_choice, response_format, and model_params as kwargs
- All 593+ existing tests pass without modification (pre-existing failures in live_scenarios and postgres tests unrelated)

## Task Commits

Each task was committed atomically (TDD: RED then GREEN):

1. **Task 1: Extend data models** - `0177e84` (test: RED), `5fe9087` (feat: GREEN)
2. **Task 2: LiteLLM adapter kwargs forwarding** - `e0ddecf` (test: RED), `e3963fc` (feat: GREEN)

## Files Created/Modified
- `src/zeroth/agent_runtime/provider.py` - Added ModelParams import, ProviderRequest tools/tool_choice/response_format/model_params fields, kwargs forwarding in LiteLLMProviderAdapter
- `src/zeroth/agent_runtime/tools.py` - Added description, parameters_schema, to_openai_tool() to ToolAttachmentManifest; updated ToolAttachmentBinding
- `src/zeroth/agent_runtime/models.py` - Added ModelParams class, model_params field on AgentConfig
- `src/zeroth/graph/models.py` - Added model_params and mcp_servers to AgentNodeData
- `tests/test_provider_request_extensions.py` - 14 tests for data model extensions
- `tests/test_litellm_adapter_forwarding.py` - 7 tests for kwargs forwarding

## Decisions Made
- ModelParams defined in models.py to avoid circular import (provider.py imports from models.py); re-exported via provider.py import for external consumers

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Moved ModelParams to models.py to avoid circular import**
- **Found during:** Task 1 (data model extension)
- **Issue:** Plan specified ModelParams in provider.py, but provider.py imports from models.py, and models.py would need ModelParams for AgentConfig field -- circular import
- **Fix:** Defined ModelParams in models.py, imported into provider.py (which already imports PromptMessage from models.py)
- **Files modified:** src/zeroth/agent_runtime/models.py, src/zeroth/agent_runtime/provider.py
- **Verification:** All imports resolve, all tests pass
- **Committed in:** 5fe9087

---

**Total deviations:** 1 auto-fixed (1 blocking)
**Impact on plan:** Structural placement change only. ModelParams is still importable from provider.py. No functional difference.

## Issues Encountered
None beyond the circular import resolved above.

## User Setup Required
None - no external service configuration required.

## Known Stubs
None - all fields are fully wired with proper defaults and forwarding.

## Next Phase Readiness
- All data models extended and ready for Plan 02 (runner threading) to wire model_params and tools through AgentRunner
- to_openai_tool() ready for Plan 03 (MCP integration) to convert registered tools to OpenAI format
- LiteLLMProviderAdapter already forwards all new fields via kwargs

---
*Phase: 19-agent-node-llm-api-parity*
*Completed: 2026-04-08*
