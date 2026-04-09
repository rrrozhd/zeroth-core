# Phase 19: Agent Node LLM API Parity - Research

**Researched:** 2026-04-08
**Domain:** LLM API integration (tool calling, structured output, model params, MCP)
**Confidence:** HIGH

## Summary

This phase closes the gap between what modern LLM APIs offer and what Zeroth's ProviderRequest currently carries. Today ProviderRequest only has `model_name`, `messages`, and `metadata`. The four additions needed are: (1) native tool/function-calling schemas forwarded to providers, (2) response_format for structured output, (3) per-node model parameters, and (4) MCP server integration for external tool discovery.

The existing codebase is well-structured for these additions. ProviderRequest is a Pydantic model with `extra="forbid"`, so new fields must be explicitly added. The LiteLLMProviderAdapter wraps LangChain's ChatLiteLLM, which already supports `bind_tools()`, `with_structured_output()`, and forwards kwargs through to litellm.completion() via its `_generate()` method. The core work is extending the data models, threading new fields through the adapter layer, and adding an MCP client lifecycle manager.

**Primary recommendation:** Add fields to ProviderRequest (tools, tool_choice, response_format, model_params), update LiteLLMProviderAdapter to forward them via ChatLiteLLM kwargs, add MCP client manager as an async context manager with tool discovery, and wire everything through AgentConfig/AgentNodeData/AgentRunner.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- ProviderRequest must carry a `tools` field with function-calling schemas (name, description, parameters JSON schema)
- ToolAttachmentManifest must be convertible to provider-native tool definitions (OpenAI function format)
- Provider adapters (LiteLLM, GovernedLLM) must forward tool schemas to the underlying API
- `tool_choice` parameter support: "auto", "none", "required", or specific tool name
- ProviderRequest must carry a `response_format` field
- When AgentConfig has output_model, its JSON schema should be sent as native `response_format` to providers that support it
- Fall back to post-hoc Pydantic validation for providers that don't support structured output
- Support both `json_schema` and `json_object` response format modes
- AgentConfig and AgentNodeData must support per-node model parameters: temperature, top_p, max_tokens, stop_sequences, seed
- ProviderRequest must carry these to the provider adapter
- Provider adapters must forward them to the underlying API
- Defaults should be None (use provider defaults) so existing behavior is unchanged
- Agent nodes can declare MCP server connections as tool sources
- MCP servers are discovered at agent startup, tools registered as ToolAttachmentManifest entries
- MCP tool calls are routed through the MCP client at runtime
- AgentConfig gets an `mcp_servers` field (list of MCP server configs)

### Claude's Discretion
- Internal representation of MCP server configs (URI, transport, auth)
- Whether to use LangChain's tool format or raw OpenAI format as the canonical tool schema
- How to handle provider-specific tool schema differences (OpenAI vs Anthropic format)
- Caching strategy for MCP tool discovery

### Deferred Ideas (OUT OF SCOPE)
- Streaming response support (separate phase)
- Vision/multimodal input support
- Model fallback chains (LLM-05 requirement)
- MCP resource and prompt protocol support (only tools for now)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| API-01 | ProviderRequest carries native tool/function-calling schemas to provider adapters; ToolAttachmentManifest converts to provider-native format | ProviderRequest needs `tools` and `tool_choice` fields; ChatLiteLLM bind_tools() accepts OpenAI format; ToolAttachmentManifest needs `to_openai_tool()` method |
| API-02 | Agent nodes support native structured output via response_format (json_schema mode) with post-hoc validation fallback | ProviderRequest needs `response_format` field; ChatLiteLLM forwards response_format via kwargs; OutputValidator already does post-hoc Pydantic validation |
| API-03 | Agent nodes support per-node model parameters forwarded to provider API | ProviderRequest needs ModelParams dataclass; ChatLiteLLM accepts temperature/top_p/max_tokens/stop/seed as kwargs |
| API-04 | Agent nodes can declare MCP server connections; tools discovered at startup and callable during execution | MCP Python SDK (`mcp` package) provides ClientSession with list_tools/call_tool; needs MCPServerConfig model, MCPClientManager lifecycle class, integration with ToolAttachmentRegistry |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Build/test: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- Project layout: `src/zeroth/` main package, `tests/` pytest tests
- Must use progress-logger skill for tracking
- Progress tracked in root `PROGRESS.md`

## Standard Stack

### Core (Already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| litellm | >=1.83,<2.0 | Universal LLM API proxy | Already used; supports tools, response_format, model params natively |
| langchain-litellm | >=0.3.4 | LangChain ChatModel wrapper | Already used; bind_tools(), with_structured_output(), kwargs forwarding |
| pydantic | >=2.10 | Data models, JSON schema | Already used; model_json_schema() generates response_format schemas |

### New Dependency
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| mcp | >=1.7,<2.0 | MCP Python SDK | API-04: Client sessions, tool discovery, tool execution |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| OpenAI tool format as canonical | LangChain tool format | OpenAI format is the de facto standard; LiteLLM accepts it natively; LangChain's convert_to_openai_tool() converts to it anyway. **Use OpenAI format.** |
| mcp SDK directly | fastmcp | fastmcp focuses on server building; official mcp SDK has first-class client support. **Use official mcp SDK.** |
| ChatLiteLLM kwargs forwarding | Direct litellm.completion() calls | Would bypass LangChain message conversion and break GovernAI integration. **Keep ChatLiteLLM.** |

**Installation:**
```bash
uv add "mcp>=1.7,<2.0"
```

## Architecture Patterns

### Where New Fields Live

```
AgentConfig
  +-- model_params: ModelParams | None       (NEW - temperature, top_p, etc.)
  +-- mcp_servers: list[MCPServerConfig]     (NEW - MCP server declarations)
  +-- tool_attachments: list[ToolAttachmentManifest]  (existing)
  +-- output_model: type[BaseModel]          (existing - drives response_format)

AgentNodeData
  +-- model_params: dict[str, Any]           (NEW - serialized model params)
  +-- mcp_servers: list[dict[str, Any]]      (NEW - serialized MCP configs)

ProviderRequest
  +-- tools: list[dict[str, Any]] | None     (NEW - OpenAI function schemas)
  +-- tool_choice: str | dict | None         (NEW - "auto"/"none"/"required"/name)
  +-- response_format: dict[str, Any] | None (NEW - json_schema/json_object)
  +-- model_params: ModelParams | None       (NEW - temperature, etc.)
```

### Pattern 1: ProviderRequest Extension
**What:** Add optional fields to ProviderRequest with None defaults so existing code is unaffected.
**When to use:** All four requirements.
**Example:**
```python
class ModelParams(BaseModel):
    """Per-node LLM parameters. None means use provider default."""
    model_config = ConfigDict(extra="forbid")
    temperature: float | None = None
    top_p: float | None = None
    max_tokens: int | None = None
    stop: list[str] | None = None
    seed: int | None = None

class ProviderRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")
    model_name: str
    messages: list[Any] = Field(default_factory=list)
    metadata: dict[str, Any] = Field(default_factory=dict)
    # NEW fields - all default None for backward compat
    tools: list[dict[str, Any]] | None = None
    tool_choice: str | dict[str, Any] | None = None
    response_format: dict[str, Any] | None = None
    model_params: ModelParams | None = None
```

### Pattern 2: LiteLLMProviderAdapter Forwarding via ChatLiteLLM kwargs
**What:** In `ainvoke()`, build a kwargs dict from ProviderRequest fields and pass to ChatLiteLLM.
**When to use:** API-01, API-02, API-03.
**Example:**
```python
async def ainvoke(self, request: ProviderRequest) -> ProviderResponse:
    client = self._get_client(request.model_name)
    lc_messages = self._to_langchain_messages(request.messages)
    # Build kwargs from ProviderRequest fields
    kwargs: dict[str, Any] = {}
    if request.tools is not None:
        kwargs["tools"] = request.tools
    if request.tool_choice is not None:
        kwargs["tool_choice"] = request.tool_choice
    if request.response_format is not None:
        kwargs["response_format"] = request.response_format
    if request.model_params is not None:
        params = request.model_params
        if params.temperature is not None:
            kwargs["temperature"] = params.temperature
        if params.max_tokens is not None:
            kwargs["max_tokens"] = params.max_tokens
        # ... etc for top_p, stop, seed
    ai_message: AIMessage = await client.ainvoke(lc_messages, **kwargs)
    # ... rest unchanged
```
**Why this works:** ChatLiteLLM._generate() merges kwargs into params (line 587: `params = {**params, **kwargs}`), which are then forwarded to litellm.completion().

### Pattern 3: ToolAttachmentManifest to OpenAI Tool Conversion
**What:** Add a method to convert ToolAttachmentManifest to OpenAI function-calling format.
**When to use:** API-01.
**Example:**
```python
class ToolAttachmentManifest(BaseModel):
    # ... existing fields ...
    description: str = ""                              # NEW
    parameters_schema: dict[str, Any] | None = None    # NEW - JSON Schema

    def to_openai_tool(self) -> dict[str, Any]:
        """Convert to OpenAI function-calling tool format."""
        func: dict[str, Any] = {"name": self.alias}
        if self.description:
            func["description"] = self.description
        if self.parameters_schema:
            func["parameters"] = self.parameters_schema
        return {"type": "function", "function": func}
```

### Pattern 4: MCP Client Lifecycle Management
**What:** An async context manager that connects to MCP servers, discovers tools, and provides call_tool dispatch.
**When to use:** API-04.
**Example:**
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class MCPServerConfig(BaseModel):
    """Configuration for connecting to an MCP server."""
    model_config = ConfigDict(extra="forbid")
    name: str
    command: str                         # e.g. "python", "node", "npx"
    args: list[str] = Field(default_factory=list)
    env: dict[str, str] | None = None
    # Future: url and transport for SSE/HTTP servers

class MCPClientManager:
    """Manages MCP server connections and tool discovery."""
    def __init__(self, configs: list[MCPServerConfig]):
        self._configs = configs
        self._sessions: dict[str, ClientSession] = {}
        self._tool_map: dict[str, str] = {}  # tool_name -> server_name
        self._exit_stack = AsyncExitStack()

    async def start(self) -> list[ToolAttachmentManifest]:
        """Connect to all MCP servers and discover tools."""
        manifests = []
        for config in self._configs:
            params = StdioServerParameters(
                command=config.command, args=config.args, env=config.env
            )
            transport = await self._exit_stack.enter_async_context(
                stdio_client(params)
            )
            session = await self._exit_stack.enter_async_context(
                ClientSession(transport[0], transport[1])
            )
            await session.initialize()
            self._sessions[config.name] = session
            response = await session.list_tools()
            for tool in response.tools:
                self._tool_map[tool.name] = config.name
                manifests.append(ToolAttachmentManifest(
                    alias=tool.name,
                    executable_unit_ref=f"mcp://{config.name}/{tool.name}",
                    description=tool.description or "",
                    parameters_schema=tool.inputSchema,
                ))
        return manifests

    async def call_tool(self, tool_name: str, args: dict) -> Any:
        server_name = self._tool_map[tool_name]
        session = self._sessions[server_name]
        result = await session.call_tool(tool_name, args)
        return result

    async def stop(self):
        await self._exit_stack.aclose()
```

### Pattern 5: response_format from output_model
**What:** When AgentConfig has an output_model, derive a json_schema response_format from its Pydantic schema.
**When to use:** API-02.
**Example:**
```python
def build_response_format(output_model: type[BaseModel]) -> dict[str, Any]:
    """Build OpenAI-style response_format from a Pydantic model."""
    schema = output_model.model_json_schema()
    return {
        "type": "json_schema",
        "json_schema": {
            "name": output_model.__name__,
            "schema": schema,
            "strict": True,
        },
    }
```

### Anti-Patterns to Avoid
- **Constructing new ChatLiteLLM per parameter set:** ChatLiteLLM instances are cached by model name. Pass model params via ainvoke kwargs, not constructor args.
- **Putting MCP tool call routing in AgentRunner directly:** Keep MCP as a tool source that registers into ToolAttachmentRegistry; route through existing tool_executor mechanism.
- **Sending response_format to all providers blindly:** Some providers don't support it. The fallback path (post-hoc Pydantic validation) must remain the default; response_format is an optimization layer.
- **Breaking ProviderRequest backward compatibility:** All new fields MUST default to None. Existing code that creates ProviderRequest(model_name=..., messages=..., metadata=...) must continue to work.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Tool schema format conversion | Custom schema transformer | OpenAI function format (de facto standard) | LiteLLM, LangChain, and all providers normalize to/from OpenAI format |
| MCP client/server protocol | Custom RPC/transport layer | `mcp` Python SDK (ClientSession, stdio_client) | Protocol is complex with negotiation, capability exchange, lifecycle management |
| JSON Schema generation from Pydantic | Manual schema construction | `BaseModel.model_json_schema()` | Pydantic handles $defs, nested models, optional fields correctly |
| Provider param compatibility checks | Custom provider capability matrix | LiteLLM's internal param mapping | LiteLLM already strips unsupported params per provider |

**Key insight:** LiteLLM already handles the hard problem of provider-specific parameter mapping. The Zeroth layer should be a clean pass-through that carries these parameters from the graph node config to the LiteLLM call, not a second normalization layer.

## Common Pitfalls

### Pitfall 1: ChatLiteLLM Client Caching vs Per-Request Params
**What goes wrong:** The current `_get_client()` caches ChatLiteLLM instances by model name. If you try to set temperature/response_format on the ChatLiteLLM instance itself, it would affect all requests to that model.
**Why it happens:** ChatLiteLLM stores temperature, max_tokens etc. as instance attributes.
**How to avoid:** Pass model params as kwargs to `ainvoke()`, NOT to the ChatLiteLLM constructor. The `_generate()` method merges kwargs into params before calling litellm.completion().
**Warning signs:** Two different agent nodes with the same model but different temperatures getting the same results.

### Pitfall 2: response_format Incompatibility
**What goes wrong:** Sending `response_format: {"type": "json_schema", ...}` to a provider that doesn't support it causes an API error.
**Why it happens:** Not all providers support structured output natively.
**How to avoid:** Make response_format opt-in per request. The OutputValidator already does post-hoc Pydantic validation as fallback. Add response_format only when the ProviderRequest explicitly carries it.
**Warning signs:** API errors mentioning "unsupported response_format" or "json_schema not supported".

### Pitfall 3: MCP Server Process Leaks
**What goes wrong:** MCP servers run as child processes (stdio transport). If not properly cleaned up, they become zombie processes.
**Why it happens:** Missing cleanup on agent shutdown, exceptions during tool calls, or unhandled cancellation.
**How to avoid:** Use AsyncExitStack for lifecycle management. Always call stop() in a finally block. Register cleanup in the agent bootstrap.
**Warning signs:** Growing number of orphan processes, port exhaustion, file descriptor leaks.

### Pitfall 4: Tool Name Collisions Between MCP and Static Tools
**What goes wrong:** An MCP server exposes a tool with the same name as a statically declared ToolAttachmentManifest.
**Why it happens:** MCP tool names are chosen by the server author, not the Zeroth graph author.
**How to avoid:** Namespace MCP tools with server name prefix (e.g., `mcp_servername_toolname`) or detect collisions at registration time and raise an error.
**Warning signs:** Wrong tool being called, unexpected tool results.

### Pitfall 5: ProviderRequest extra="forbid" Breaks Silently
**What goes wrong:** Adding new fields to ProviderRequest without updating the model causes Pydantic validation errors at every call site that constructs ProviderRequests.
**Why it happens:** `extra="forbid"` rejects unknown fields.
**How to avoid:** All new fields must have defaults (None). Verify all ProviderRequest construction sites still pass validation.
**Warning signs:** ValidationError on ProviderRequest construction in tests.

## Code Examples

### Converting ToolAttachmentManifest to OpenAI Tool Format
```python
# OpenAI function-calling tool format (the de facto standard)
tool_schema = {
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
}
```

### LiteLLM response_format with json_schema
```python
# Source: https://docs.litellm.ai/docs/completion/json_mode
response_format = {
    "type": "json_schema",
    "json_schema": {
        "name": "MyOutputModel",
        "schema": MyOutputModel.model_json_schema(),
        "strict": True,
    }
}
# Passed via: litellm.completion(..., response_format=response_format)
# Via ChatLiteLLM: client.ainvoke(messages, response_format=response_format)
```

### MCP Client Session Lifecycle
```python
# Source: https://modelcontextprotocol.io/docs/develop/build-client
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from contextlib import AsyncExitStack

stack = AsyncExitStack()
params = StdioServerParameters(command="python", args=["server.py"])
transport = await stack.enter_async_context(stdio_client(params))
session = await stack.enter_async_context(ClientSession(transport[0], transport[1]))
await session.initialize()

# Discover tools
response = await session.list_tools()
for tool in response.tools:
    print(tool.name, tool.description, tool.inputSchema)

# Call a tool
result = await session.call_tool("tool_name", {"arg1": "value1"})

# Cleanup
await stack.aclose()
```

### ChatLiteLLM kwargs Forwarding Path
```python
# In ChatLiteLLM._generate() (langchain_litellm source, verified):
# Line 586: message_dicts, params = self._create_message_dicts(messages, stop)
# Line 587: params = {**params, **kwargs}  <-- kwargs merged here
# Line 588: response = self.completion_with_retry(messages=message_dicts, **params)
#
# This means ainvoke(messages, tools=..., response_format=..., temperature=...)
# all flow through to litellm.completion() as keyword arguments.
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Text-based tool hints in system prompt | Native function-calling via tools parameter | OpenAI 2023, now universal | Providers parse and enforce tool schemas; more reliable |
| Post-hoc JSON parsing | response_format json_schema | OpenAI Aug 2024, Anthropic Nov 2025 | Providers enforce JSON schema at generation time |
| Hardcoded model params | Per-request param overrides | Always available in LiteLLM | Enables per-node temperature/max_tokens tuning |
| Custom tool integrations | MCP protocol | Anthropic Nov 2024 | Standardized tool server protocol, growing ecosystem |

**Deprecated/outdated:**
- `function_call` / `functions` parameters: Replaced by `tools` / `tool_choice` (OpenAI deprecated Nov 2023)
- LiteLLM `response_format: {"type": "json_object"}` alone: Still works but `json_schema` provides stronger guarantees

## Open Questions

1. **MCP server process management in production**
   - What we know: stdio_client spawns child processes; AsyncExitStack manages cleanup
   - What's unclear: Should MCP servers be long-lived (started at bootstrap, shared across runs) or per-run (started per agent execution)?
   - Recommendation: Start MCP servers at agent bootstrap time (when AgentRunner is created), keep alive for the lifetime of the deployment. Per-run startup would be too slow.

2. **GovernedLLMProviderAdapter tool/param forwarding**
   - What we know: GovernedLLMProviderAdapter wraps GovernAI's GovernedLLM, which has its own invoke interface
   - What's unclear: Does GovernedLLM's invoke support tools/response_format kwargs?
   - Recommendation: Start with LiteLLMProviderAdapter support only. GovernedLLM adapter can be extended in a follow-up if needed. Add a warning log if tools/response_format are set but adapter doesn't support them.

## Sources

### Primary (HIGH confidence)
- LangChain-LiteLLM source code (installed at `.venv/lib/python3.12/site-packages/langchain_litellm/chat_models/litellm.py`) - verified bind_tools(), with_structured_output(), kwargs forwarding mechanism
- Zeroth codebase source files - ProviderRequest, AgentConfig, AgentRunner, ToolAttachmentManifest, OutputValidator current implementation
- [LiteLLM Function Calling docs](https://docs.litellm.ai/docs/completion/function_call) - tools parameter format, tool_choice options
- [LiteLLM Structured Output docs](https://docs.litellm.ai/docs/completion/json_mode) - response_format json_schema format, supported providers
- [LiteLLM Input Params docs](https://docs.litellm.ai/docs/completion/input) - full parameter list (temperature, top_p, max_tokens, stop, seed)

### Secondary (MEDIUM confidence)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) - ClientSession API, stdio_client, list_tools/call_tool
- [MCP Build Client tutorial](https://modelcontextprotocol.io/docs/develop/build-client) - lifecycle pattern, connection management
- [MCP PyPI](https://pypi.org/project/mcp/) - latest version 1.27.0

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all existing libraries verified in codebase; mcp SDK is official and well-documented
- Architecture: HIGH - ChatLiteLLM kwargs forwarding verified in source code; OpenAI tool format is de facto standard
- Pitfalls: HIGH - identified from actual source code analysis (client caching, extra="forbid", process management)

**Research date:** 2026-04-08
**Valid until:** 2026-05-08 (30 days - stable domain, well-established APIs)
