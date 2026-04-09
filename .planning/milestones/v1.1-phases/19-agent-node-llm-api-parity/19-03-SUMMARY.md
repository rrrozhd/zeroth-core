---
phase: 19-agent-node-llm-api-parity
plan: 03
subsystem: agent-runtime
tags: [mcp, tool-discovery, client-lifecycle, async-exit-stack, stdio-transport]

# Dependency graph
requires:
  - phase: 19-agent-node-llm-api-parity
    plan: 01
    provides: ToolAttachmentManifest, ToolAttachmentRegistry, AgentConfig, AgentRunner
provides:
  - MCPServerConfig model for stdio-based MCP server configuration
  - MCPClientManager async lifecycle class (start/call_tool/stop)
  - AgentRunner MCP lifecycle integration (start before run, stop in finally)
  - MCP tool routing via executable_unit_ref prefix check
affects: []

# Tech tracking
tech-stack:
  added: [mcp>=1.7]
  patterns: [async-exit-stack-lifecycle, lazy-mcp-imports, tool-name-collision-namespacing]

key-files:
  created:
    - src/zeroth/agent_runtime/mcp.py
    - tests/test_mcp_integration.py
    - tests/test_runner_mcp_wiring.py
  modified:
    - pyproject.toml
    - src/zeroth/agent_runtime/models.py
    - src/zeroth/agent_runtime/runner.py
    - src/zeroth/agent_runtime/tools.py

key-decisions:
  - "MCP imports lazy inside start() to avoid import-time dependency on mcp SDK"
  - "Tool name collisions resolved via server_name__tool_name namespacing prefix"
  - "MCP tool calls routed by checking executable_unit_ref.startswith('mcp://') before dispatching to MCPClientManager"
  - "tool_executor=None allowed when _mcp_manager handles MCP tool calls"

patterns-established:
  - "AsyncExitStack lifecycle: start() enters contexts, stop() calls aclose()"
  - "MCP tool discovery returns ToolAttachmentManifest entries registered in bridge registry"
  - "try/finally pattern for MCP cleanup around run() retry loop"

requirements-completed: [API-04]

# Metrics
duration: 10min
completed: 2026-04-08
---

# Phase 19 Plan 03: MCP Server Integration Summary

**MCP client lifecycle management with tool discovery, name-collision handling, and AgentRunner wiring for external MCP server tool access**

## Performance

- **Duration:** 10 min (603s)
- **Started:** 2026-04-08T11:00:42Z
- **Completed:** 2026-04-08T11:10:45Z
- **Tasks:** 2
- **Files modified:** 7

## Accomplishments

- Created MCPServerConfig model for stdio-based MCP server configuration (name, command, args, env)
- Created MCPClientManager with full async lifecycle: start() discovers tools, call_tool() routes to correct server, stop() cleans up via AsyncExitStack
- Tool discovery returns ToolAttachmentManifest entries with description and parameters_schema
- Tool name collisions across servers handled via server_name__tool_name namespacing
- Added mcp_servers field to AgentConfig for declarative MCP server configuration
- Wired MCP lifecycle into AgentRunner: start before retry loop, stop in finally block (cleanup even on error)
- MCP tool calls routed through MCPClientManager.call_tool() based on mcp:// executable_unit_ref prefix
- Added mcp>=1.7,<2.0 SDK dependency
- 21 new tests (11 MCP integration + 10 runner wiring) all passing
- All existing agent_runtime tests continue to pass

## Task Commits

1. **Task 1: MCPServerConfig + MCPClientManager** - `c1ecfa8`
2. **Task 2: AgentConfig + AgentRunner MCP wiring** - `e1fb129`

## Files Created/Modified

- `src/zeroth/agent_runtime/mcp.py` - New module: MCPServerConfig, MCPClientManager with start/call_tool/stop lifecycle
- `src/zeroth/agent_runtime/models.py` - Added MCPServerConfig import and mcp_servers field to AgentConfig
- `src/zeroth/agent_runtime/runner.py` - Added MCPClientManager import, _mcp_manager attribute, _start_mcp_servers/_stop_mcp_servers methods, MCP tool routing in _resolve_tool_calls, try/finally wrapper
- `src/zeroth/agent_runtime/tools.py` - Added description and parameters_schema fields to ToolAttachmentManifest and ToolAttachmentBinding
- `pyproject.toml` - Added mcp>=1.7,<2.0 dependency
- `tests/test_mcp_integration.py` - 11 tests for MCPServerConfig, MCPClientManager lifecycle
- `tests/test_runner_mcp_wiring.py` - 10 tests for AgentConfig mcp_servers and AgentRunner MCP wiring

## Decisions Made

- MCP SDK imports are lazy (inside start() method) to avoid import-time dependency
- Tool name collisions resolved via server_name__tool_name prefix convention
- MCP tool routing uses executable_unit_ref prefix check (mcp://) to distinguish from regular tool_executor calls
- tool_executor=None check relaxed when _mcp_manager is available

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Added description and parameters_schema fields to ToolAttachmentManifest**
- **Found during:** Task 1
- **Issue:** Plan 19-03 depends on Plan 19-01 adding description and parameters_schema to ToolAttachmentManifest, but 19-01 changes haven't been merged to this worktree's branch yet
- **Fix:** Added the fields directly to ToolAttachmentManifest and ToolAttachmentBinding, matching what 19-01 specifies
- **Files modified:** src/zeroth/agent_runtime/tools.py
- **Commit:** c1ecfa8

**2. [Rule 1 - Bug] Updated tool_executor None check for MCP-only agents**
- **Found during:** Task 2 test execution
- **Issue:** _resolve_tool_calls raised "no tool executor configured" even when MCP manager was available to handle tool calls
- **Fix:** Changed check to `if self.tool_executor is None and self._mcp_manager is None`
- **Files modified:** src/zeroth/agent_runtime/runner.py
- **Commit:** e1fb129

---

**Total deviations:** 2 auto-fixed (1 blocking, 1 bug)
**Impact on plan:** No functional difference. All must_haves truths satisfied.

## Issues Encountered

None beyond the deviations documented above.

## User Setup Required

None - mcp SDK installed automatically via uv sync.

## Known Stubs

None - all MCP lifecycle methods are fully implemented with real SDK integration.

---
*Phase: 19-agent-node-llm-api-parity*
*Completed: 2026-04-08*
