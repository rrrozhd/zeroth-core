---
phase: 19-agent-node-llm-api-parity
verified: 2026-04-08T14:30:00Z
status: passed
score: 7/7 must-haves verified
gaps: []
---

# Phase 19: Agent Node LLM API Parity Verification Report

**Phase Goal:** Make agent graph nodes expose the full power of modern LLM APIs -- native tool schemas, structured output, model parameters, and MCP server integration.
**Verified:** 2026-04-08
**Status:** passed
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | ProviderRequest accepts tools, tool_choice, response_format, and model_params fields without breaking existing construction | VERIFIED | `provider.py` lines 39-42: all four fields present with `None` defaults. 14 tests in `test_provider_request_extensions.py` pass including backward compat. |
| 2 | ToolAttachmentManifest exposes description and parameters_schema and converts to OpenAI function-calling format | VERIFIED | `tools.py` lines 42-58: `description`, `parameters_schema`, and `to_openai_tool()` method present. Tests verify correct OpenAI format output. |
| 3 | LiteLLMProviderAdapter forwards tools, tool_choice, response_format, and model_params to ChatLiteLLM.ainvoke() kwargs | VERIFIED | `provider.py` lines 168-187: kwargs building and `**kwargs` forwarding. 7 tests in `test_litellm_adapter_forwarding.py` pass. |
| 4 | AgentRunner._build_provider_request() wires tool_attachments, response_format, and model_params from config into every ProviderRequest | VERIFIED | `runner.py` lines 242-265: `_build_provider_request()` calls `to_openai_tool()` on attachments, `build_response_format()` on output_model, passes `model_params`. Used at both call sites (line 153 and line 355). 8 tests in `test_runner_api_parity.py` pass. |
| 5 | build_response_format() converts Pydantic output_model to json_schema response_format (None for bare BaseModel) | VERIFIED | `response_format.py` lines 10-27: full implementation with bare BaseModel guard. Tests verify None for BaseModel and json_schema dict for custom models. |
| 6 | MCPClientManager connects to MCP servers, discovers tools as ToolAttachmentManifest entries, and routes call_tool to correct session | VERIFIED | `mcp.py` lines 30-126: MCPServerConfig, MCPClientManager with `start()`, `call_tool()`, `stop()`. Tool name collision handling via namespacing. 11 tests in `test_mcp_integration.py` pass. |
| 7 | AgentRunner starts MCP lifecycle at run time, registers discovered tools, routes MCP tool calls, and cleans up in finally block | VERIFIED | `runner.py` lines 145/239-240: `_start_mcp_servers()` before retry loop, `_stop_mcp_servers()` in `finally`. Lines 311-321: MCP tool call routing via `mcp://` prefix check. 10 tests in `test_runner_mcp_wiring.py` pass. |

**Score:** 7/7 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/agent_runtime/provider.py` | ModelParams, extended ProviderRequest, LiteLLM kwargs forwarding | VERIFIED | ModelParams imported from models.py; ProviderRequest has 4 new fields; LiteLLM adapter builds and forwards kwargs |
| `src/zeroth/agent_runtime/tools.py` | ToolAttachmentManifest with description, parameters_schema, to_openai_tool() | VERIFIED | All three present and functional |
| `src/zeroth/agent_runtime/models.py` | AgentConfig with model_params and mcp_servers fields | VERIFIED | `model_params: ModelParams \| None = None` (line 108), `mcp_servers: list[MCPServerConfig]` (line 106) |
| `src/zeroth/agent_runtime/runner.py` | _build_provider_request(), MCP lifecycle management | VERIFIED | Both methods exist and are wired into run() and _resolve_tool_calls() |
| `src/zeroth/agent_runtime/response_format.py` | build_response_format() helper | VERIFIED | Full implementation with bare BaseModel guard |
| `src/zeroth/agent_runtime/mcp.py` | MCPServerConfig, MCPClientManager | VERIFIED | Full lifecycle: start/call_tool/stop with AsyncExitStack |
| `src/zeroth/graph/models.py` | AgentNodeData with model_params and mcp_servers | VERIFIED | Lines 124-125: `model_params: dict[str, Any]`, `mcp_servers: list[dict[str, Any]]` |
| `tests/test_provider_request_extensions.py` | 14 tests | VERIFIED | 14 tests, all pass |
| `tests/test_litellm_adapter_forwarding.py` | 7 tests | VERIFIED | 7 tests, all pass |
| `tests/test_runner_api_parity.py` | 8 tests | VERIFIED | 8 tests, all pass |
| `tests/test_mcp_integration.py` | 11 tests | VERIFIED | 11 tests, all pass |
| `tests/test_runner_mcp_wiring.py` | 10 tests | VERIFIED | 10 tests, all pass |
| `pyproject.toml` | mcp dependency | VERIFIED | `"mcp>=1.7,<2.0"` present |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `provider.py` LiteLLMProviderAdapter | ChatLiteLLM.ainvoke() | `**kwargs` forwarding | WIRED | Line 187: `await client.ainvoke(lc_messages, **kwargs)` with tools/tool_choice/response_format/model_params in kwargs |
| `tools.py` ToolAttachmentManifest | `provider.py` ProviderRequest.tools | to_openai_tool() produces dicts | WIRED | `to_openai_tool()` at line 51-58; consumed in runner `_build_provider_request()` |
| `runner.py` AgentRunner | `provider.py` ProviderRequest | _build_provider_request() | WIRED | Lines 153 and 355 both use `self._build_provider_request()` |
| `response_format.py` | pydantic BaseModel.model_json_schema() | build_response_format() | WIRED | Line 19: `schema = output_model.model_json_schema()` |
| `runner.py` | `tools.py` to_openai_tool() | _build_provider_request() calls to_openai_tool() | WIRED | Line 253: `tools = [att.to_openai_tool() for att in self.config.tool_attachments]` |
| `mcp.py` MCPClientManager | mcp.ClientSession | start() creates sessions via stdio_client | WIRED | Lines 50-66: imports and uses ClientSession, StdioServerParameters, stdio_client |
| `runner.py` | `mcp.py` MCPClientManager | start/stop/call_tool lifecycle | WIRED | Lines 28/84/145/239-240/311-321: import, init, start, stop, call_tool routing |
| `mcp.py` | `tools.py` ToolAttachmentManifest | start() returns manifests from discovered tools | WIRED | Lines 83-91: creates ToolAttachmentManifest for each discovered MCP tool |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-----------|-------------|--------|----------|
| API-01 | 19-01, 19-02 | ProviderRequest carries native tool/function-calling schemas; ToolAttachmentManifest converts to provider-native format | SATISFIED | ProviderRequest.tools field + ToolAttachmentManifest.to_openai_tool() + LiteLLM kwargs forwarding + runner _build_provider_request() wiring |
| API-02 | 19-01, 19-02 | Native structured output via response_format (json_schema mode) with post-hoc validation fallback | SATISFIED | ProviderRequest.response_format + build_response_format() + runner wiring; OutputValidator provides existing post-hoc fallback |
| API-03 | 19-01, 19-02 | Per-node model parameters (temperature, max_tokens, top_p, stop, seed, tool_choice) forwarded to provider API | SATISFIED | ModelParams class with 5 params + ProviderRequest.tool_choice + LiteLLM kwargs forwarding + AgentConfig.model_params + runner wiring |
| API-04 | 19-03 | Agent nodes declare MCP server connections; tools discovered at startup and callable during execution | SATISFIED | MCPServerConfig + MCPClientManager + AgentConfig.mcp_servers + AgentRunner MCP lifecycle (start/stop/call_tool routing) |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `src/zeroth/agent_runtime/tools.py` | 42-43, 47-48 | Duplicate field declarations (`description` and `parameters_schema` declared twice in ToolAttachmentManifest) | Info | Pydantic uses last declaration; no runtime impact but code quality issue |
| `src/zeroth/agent_runtime/tools.py` | 93-94, 98-99 | Duplicate field declarations (`description` and `parameters_schema` declared twice in ToolAttachmentBinding) | Info | Same issue in ToolAttachmentBinding |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| All 50 phase tests pass | `uv run pytest tests/test_provider_request_extensions.py tests/test_litellm_adapter_forwarding.py tests/test_runner_api_parity.py tests/test_mcp_integration.py tests/test_runner_mcp_wiring.py -v` | 50 passed in 0.17s | PASS |
| Ruff lint clean | `uv run ruff check` on all 7 modified source files | All checks passed | PASS |

### Human Verification Required

### 1. LiteLLM kwargs forwarding with real provider

**Test:** Run an agent with `model_params=ModelParams(temperature=0.0)` against a real OpenAI/Anthropic endpoint and verify the response reflects deterministic sampling.
**Expected:** Low-temperature requests produce consistent outputs across repeated calls.
**Why human:** Requires live API credentials and subjective assessment of output consistency.

### 2. MCP server lifecycle with real MCP server

**Test:** Configure an agent with a real MCP server (e.g., a simple Python MCP tool server), run the agent, and verify tools are discovered and callable.
**Expected:** MCP server starts, tools appear in ProviderRequest.tools, tool calls route through MCP and return results.
**Why human:** Requires a running MCP server process and end-to-end integration testing.

### Gaps Summary

No gaps found. All four requirements (API-01 through API-04) are fully implemented and verified:

- **API-01:** ProviderRequest carries tool schemas, ToolAttachmentManifest.to_openai_tool() converts to OpenAI format, LiteLLM adapter forwards via kwargs, runner wires them through _build_provider_request().
- **API-02:** response_format field on ProviderRequest, build_response_format() converts Pydantic models to json_schema format, runner auto-generates from output_model. Post-hoc validation via existing OutputValidator remains as fallback.
- **API-03:** ModelParams class with temperature/top_p/max_tokens/stop/seed, tool_choice on ProviderRequest, all forwarded by LiteLLM adapter, threaded by runner from AgentConfig.
- **API-04:** MCPServerConfig + MCPClientManager with full lifecycle, AgentConfig.mcp_servers, AgentRunner starts/stops MCP servers around run(), discovered tools registered as ToolAttachmentManifest entries, MCP tool calls routed via mcp:// prefix.

Minor code quality note: duplicate field declarations in tools.py (description and parameters_schema declared twice in both ToolAttachmentManifest and ToolAttachmentBinding). No runtime impact.

---

_Verified: 2026-04-08_
_Verifier: Claude (gsd-verifier)_
