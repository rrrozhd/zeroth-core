---
phase: 19-agent-node-llm-api-parity
plan: 02
subsystem: agent-runtime
tags: [openai, tools, response-format, model-params, pydantic, json-schema]

# Dependency graph
requires:
  - phase: 19-agent-node-llm-api-parity
    plan: 01
    provides: "ProviderRequest fields (tools, response_format, model_params), ToolAttachmentManifest.to_openai_tool(), ModelParams"
provides:
  - "_build_provider_request() centralizes ProviderRequest construction with tools, response_format, model_params"
  - "build_response_format() converts Pydantic output_model to OpenAI json_schema response_format"
  - "Both run() and _resolve_tool_calls() use the same helper for consistent field threading"
affects: [19-03-agent-node-llm-api-parity]

# Tech tracking
tech-stack:
  added: []
  patterns: ["Centralized ProviderRequest construction via _build_provider_request helper", "Lazy import for response_format module inside helper method"]

key-files:
  created:
    - src/zeroth/agent_runtime/response_format.py
    - tests/test_runner_api_parity.py
  modified:
    - src/zeroth/agent_runtime/runner.py

key-decisions:
  - "Lazy import of build_response_format inside _build_provider_request to avoid circular imports"
  - "response_format returns None for bare BaseModel (no fields) to preserve backward compat"

patterns-established:
  - "Centralized ProviderRequest builder: all LLM calls go through _build_provider_request() for consistent field population"

requirements-completed: [API-01, API-02, API-03]

# Metrics
duration: 345s
completed: 2026-04-08
---

# Phase 19 Plan 02: Runner Wiring Summary

**AgentRunner._build_provider_request() wires tool schemas, structured output format, and model parameters into every LLM API call**

## Performance

- **Duration:** 5 min 45s
- **Started:** 2026-04-08T11:00:43Z
- **Completed:** 2026-04-08T11:06:28Z
- **Tasks:** 1
- **Files modified:** 3

## Accomplishments
- Created `response_format.py` with `build_response_format()` that converts Pydantic output_model to OpenAI json_schema format (returns None for bare BaseModel)
- Added `_build_provider_request()` helper to AgentRunner that centralizes ProviderRequest construction with tools, response_format, and model_params
- Replaced both ProviderRequest construction sites in run() and _resolve_tool_calls() with the new helper
- Added 8 tests: 3 for build_response_format unit tests, 5 for runner wiring (tools, response_format, model_params, backward compat, tool-call re-invocation)

## Task Commits

Each task was committed atomically:

1. **Task 1: Create response_format builder and wire runner ProviderRequest construction** - `2049adf` (feat)

## Files Created/Modified
- `src/zeroth/agent_runtime/response_format.py` - build_response_format() converts output_model to json_schema response_format
- `src/zeroth/agent_runtime/runner.py` - _build_provider_request() helper, replaced 2 ProviderRequest construction sites
- `tests/test_runner_api_parity.py` - 8 tests for runner wiring of tools, response_format, model_params

## Decisions Made
- Lazy import of `build_response_format` inside `_build_provider_request` to avoid circular imports between runner and response_format modules
- `build_response_format` returns None for bare BaseModel (no custom fields) to preserve backward compatibility -- bare BaseModel means "any output", not "structured output"

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered
- Plan 01 changes were on a separate worktree branch; required merging `worktree-agent-a35aade8` to get the prerequisite interfaces (ProviderRequest fields, to_openai_tool, ModelParams)
- Bare BaseModel cannot be passed to `model_json_schema()` in Pydantic v2 (raises AttributeError); test helper adjusted to use SimpleOutput as default instead of BaseModel

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Runner wiring complete, ready for Plan 03 (end-to-end integration and OpenAI adapter validation)
- All existing agent_runtime tests pass (45 tests in runner suite)

---
*Phase: 19-agent-node-llm-api-parity*
*Completed: 2026-04-08*
