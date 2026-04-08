# Phase 19: Agent Node LLM API Parity - Context

**Gathered:** 2026-04-08
**Status:** Ready for planning
**Source:** Direct user requirements

<domain>
## Phase Boundary

Make agent graph nodes expose the full power of modern LLM APIs. Today, ProviderRequest only carries model_name + messages + metadata — no native tool schemas, no structured output, no model params, no MCP. This phase closes that gap so an agent node behaves like a direct LLM API call but embedded in a governed workflow graph.

</domain>

<decisions>
## Implementation Decisions

### Native Tool Schemas
- ProviderRequest must carry a `tools` field with function-calling schemas (name, description, parameters JSON schema)
- ToolAttachmentManifest must be convertible to provider-native tool definitions (OpenAI function format)
- Provider adapters (LiteLLM, GovernedLLM) must forward tool schemas to the underlying API
- `tool_choice` parameter support: "auto", "none", "required", or specific tool name

### Structured Output
- ProviderRequest must carry a `response_format` field
- When AgentConfig has output_model, its JSON schema should be sent as native `response_format` to providers that support it (OpenAI, Anthropic)
- Fall back to post-hoc Pydantic validation for providers that don't support structured output
- Support both `json_schema` and `json_object` response format modes

### Model Parameters
- AgentConfig and AgentNodeData must support per-node model parameters: temperature, top_p, max_tokens, stop_sequences, seed
- ProviderRequest must carry these to the provider adapter
- Provider adapters must forward them to the underlying API
- Defaults should be None (use provider defaults) so existing behavior is unchanged

### MCP Server Integration
- Agent nodes can declare MCP server connections as tool sources
- MCP servers are discovered at agent startup, tools registered as ToolAttachmentManifest entries
- MCP tool calls are routed through the MCP client at runtime
- AgentConfig gets an `mcp_servers` field (list of MCP server configs)

### Claude's Discretion
- Internal representation of MCP server configs (URI, transport, auth)
- Whether to use LangChain's tool format or raw OpenAI format as the canonical tool schema
- How to handle provider-specific tool schema differences (OpenAI vs Anthropic format)
- Caching strategy for MCP tool discovery

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Agent Runtime
- `src/zeroth/agent_runtime/provider.py` — ProviderRequest, ProviderResponse, all provider adapters
- `src/zeroth/agent_runtime/models.py` — AgentConfig, PromptConfig, RetryPolicy, ToolAttachmentManifest
- `src/zeroth/agent_runtime/runner.py` — AgentRunner execution loop, tool call resolution
- `src/zeroth/agent_runtime/prompt.py` — PromptAssembler, how prompts are built
- `src/zeroth/agent_runtime/tools.py` — ToolAttachmentRegistry, ToolAttachmentBridge

### Graph Models
- `src/zeroth/graph/models.py` — AgentNodeData, AgentNode, graph node types

### Orchestrator
- `src/zeroth/orchestrator/runtime.py` — RuntimeOrchestrator._dispatch_node(), where agents are invoked

### Bootstrap
- `src/zeroth/service/bootstrap.py` — How agent runners and providers are wired

</canonical_refs>

<specifics>
## Specific Ideas

- User wants agent nodes to feel like "talking to raw LLM API of some provider just via a graph node"
- Tools, prompt, MCPs, structured output are the four pillars
- Provider-native function calling is critical (not text-based tool hints in system prompt)
- Structured output via response_format is critical (not just post-hoc validation)
- Model params (temperature, max_tokens, etc.) must be per-node configurable

</specifics>

<deferred>
## Deferred Ideas

- Streaming response support (separate phase)
- Vision/multimodal input support
- Model fallback chains (LLM-05 requirement)
- MCP resource and prompt protocol support (only tools for now)

</deferred>

---

*Phase: 19-agent-node-llm-api-parity*
*Context gathered: 2026-04-08 via direct user requirements*
