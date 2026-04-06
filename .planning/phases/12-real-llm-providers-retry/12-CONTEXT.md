# Phase 12: Real LLM Providers & Retry - Context

**Gathered:** 2026-04-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 12 replaces test provider stubs with a universal LLM adapter backed by LiteLLM (via LangChain's ChatLiteLLM wrapper), adds retry with exponential backoff and jitter on transient failures, and captures token usage from provider responses into typed audit records. Each agent node specifies its own provider/model — there are no global LLM defaults.

</domain>

<decisions>
## Implementation Decisions

### Provider SDK & Architecture
- **D-01:** Use LangChain's `ChatLiteLLM` wrapper as the universal provider interface — LiteLLM handles provider routing (OpenAI, Anthropic, 100+ others), LangChain interface maintains GovernAI compatibility
- **D-02:** Single `LiteLLMProviderAdapter` implementing the existing `ProviderAdapter` protocol — replaces the need for separate per-provider adapters
- **D-03:** `LiteLLMProviderAdapter` is the primary adapter. `GovernedLLMProviderAdapter` wraps it when governance is enabled on a node — users choose governed vs ungoverned per node
- **D-04:** No global LLM defaults in `ZerothSettings` — each agent node specifies its own provider and model

### Agent Node Configuration
- **D-05:** Required `model: str` field added to `AgentConfig` using LiteLLM format strings (e.g., `openai/gpt-4o`, `anthropic/claude-sonnet-4-5-20250514`)
- **D-06:** Per-node retry configuration on `AgentConfig` — max_retries, base_delay, max_delay. Different nodes may have different latency tolerances

### Retry Strategy
- **D-07:** Retry-only in Phase 12 — exponential backoff with jitter on transient failures. No model fallback chains (deferred to LLM-05)
- **D-08:** API keys managed via standard environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.) — LiteLLM reads these directly, no `ZEROTH_` prefix wrapping

### Token Usage Capture
- **D-09:** New typed `TokenUsage` pydantic model with `input_tokens`, `output_tokens`, `total_tokens`, `model_name` fields
- **D-10:** `TokenUsage` added to `ProviderResponse` — each adapter extracts tokens from the raw SDK response and populates it
- **D-11:** `TokenUsage` also added as a dedicated typed field on `NodeAuditRecord` — `AgentRunner` copies from `ProviderResponse` to audit record. Clean flow: provider → response → audit

### Testing
- **D-12:** Unit tests mock `litellm.acompletion()` to return canned responses — validates adapter logic (token extraction, error mapping, retry) without network calls
- **D-13:** Optional live integration tests gated behind `@pytest.mark.live` — only run when API keys are present, CI skips them

### Claude's Discretion
- Exact retryable error set (rate limits, server errors, timeouts — standard provider best practices)
- `TokenUsage` field naming and optional fields beyond input/output/total
- Connection pooling and session management within LiteLLM
- How `LiteLLMProviderAdapter` handles LiteLLM-specific configuration (timeouts, custom headers)
- Whether to add an `LLMSettings` sub-model for retry defaults or keep it purely per-node

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Provider Interface (existing)
- `src/zeroth/agent_runtime/provider.py` — `ProviderAdapter` protocol, `ProviderRequest`, `ProviderResponse`, `GovernedLLMProviderAdapter`, `run_provider_with_timeout()`
- `src/zeroth/agent_runtime/runner.py` — `AgentRunner` that calls provider and builds audit records
- `src/zeroth/agent_runtime/models.py` — `AgentConfig`, `AgentRunResult`, `PromptMessage`

### Audit System
- `src/zeroth/audit/models.py` — `NodeAuditRecord` (where `token_usage` field will be added), `ToolCallRecord`, `MemoryAccessRecord`
- `src/zeroth/audit/repository.py` — `AuditRepository` for persisting audit records

### Configuration
- `src/zeroth/config/settings.py` — `ZerothSettings`, `DatabaseSettings`, `RedisSettings` sub-model pattern

### Service Wiring
- `src/zeroth/service/bootstrap.py` — `ServiceBootstrap` composition root that constructs all components
- `src/zeroth/orchestrator/runtime.py` — `RuntimeOrchestrator` that drives node execution and calls AgentRunner

### Requirements
- `.planning/REQUIREMENTS.md` — LLM-01, LLM-02, LLM-03, LLM-04

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ProviderAdapter` protocol with `ainvoke(ProviderRequest) -> ProviderResponse` — new adapter implements this directly
- `GovernedLLMProviderAdapter` — will wrap the new LiteLLM adapter instead of wrapping GovernedLLM directly
- `DeterministicProviderAdapter` — test fake returning canned responses, continues to work for non-LLM tests
- `run_provider_with_timeout()` — existing timeout wrapper, can be composed with retry logic
- `AgentRunner` — already handles the provider call loop, tool calls, and audit record creation

### Established Patterns
- All provider adapters follow `ProviderAdapter` protocol with `ainvoke()` method
- `ProviderResponse` uses pydantic `BaseModel` with `ConfigDict(extra="forbid")`
- `AgentConfig` is the per-node configuration object passed to `AgentRunner`
- Audit records use pydantic models with `Field(default_factory=...)` for optional collections

### Integration Points
- `AgentRunner.__init__()` takes `provider: ProviderAdapter` — new adapter slots in here
- `AgentRunner.run()` calls `run_provider_with_timeout()` — retry wrapper goes around this call
- `NodeAuditRecord` construction in `AgentRunner` — token usage gets attached here
- `ServiceBootstrap` — provider adapter instantiation happens here based on node config

</code_context>

<specifics>
## Specific Ideas

- LiteLLM model format strings (e.g., `openai/gpt-4o`) provide implicit provider routing — no separate provider selection needed
- LangChain's ChatLiteLLM bridges LiteLLM's universal routing with GovernAI's LangChain-based governance layer
- Per-node model config means the same workflow can use different models for different nodes (e.g., cheap model for classification, expensive model for generation)
- Standard env var conventions for API keys align with LiteLLM documentation and developer expectations

</specifics>

<deferred>
## Deferred Ideas

- **LLM-05: Model fallback chains** — ordered list of models tried in sequence on failure. Deferred to future phase per requirements doc.
- **LLM-06: Streaming response support** — async streaming for agent nodes. Out of scope for Phase 12.
- **ECON-01: Regulus instrumentation wrapper** — wrapping the LiteLLM adapter with cost events belongs in Phase 13.
- **Global LLM defaults / LLMSettings** — if a common default model is needed, can be added later without breaking per-node config.

</deferred>

---

*Phase: 12-real-llm-providers-retry*
*Context gathered: 2026-04-07*
