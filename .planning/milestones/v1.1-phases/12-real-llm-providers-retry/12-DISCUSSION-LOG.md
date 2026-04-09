# Phase 12: Real LLM Providers & Retry - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-07
**Phase:** 12-real-llm-providers-retry
**Areas discussed:** Provider SDK choice, Retry & fallback scope, Token usage capture, Config & credentials, Agent node model config, Retry configurability, Testing strategy, GovernAI adapter coexistence

---

## Provider SDK Choice

| Option | Description | Selected |
|--------|-------------|----------|
| Native SDKs directly | Use openai and anthropic Python SDKs directly. Thinner dependency, full control. | |
| LangChain wrappers | Use langchain-openai and langchain-anthropic as specified in requirements. | ✓ |
| Both — native + GovernAI wrapper | Native SDK adapters plus GovernedLLMProviderAdapter for GovernAI calls. | |

**User's choice:** LangChain wrappers
**Notes:** User later clarified they want LangChain's ChatLiteLLM wrapper specifically — combining LiteLLM's universal provider routing with LangChain's interface.

---

## Retry & Fallback Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Retry only | Exponential backoff + jitter on rate-limit/transient errors. No fallback. LLM-05 stays deferred. | ✓ |
| Retry + simple fallback | Add fallback_model field. Try fallback once after primary exhausts retries. | |
| Retry + full fallback chain | Ordered list of models tried in sequence. Pulls LLM-05 fully in. | |

**User's choice:** Retry only (Recommended)
**Notes:** Clean scope boundary. Model fallback chains remain deferred to LLM-05.

---

## Token Usage Capture

| Option | Description | Selected |
|--------|-------------|----------|
| Typed TokenUsage model | New pydantic model as dedicated field on NodeAuditRecord. Strongly typed, queryable. | ✓ |
| Inside execution_metadata dict | Store as execution_metadata['token_usage']. No schema change but loosely typed. | |
| On ProviderResponse + audit field | Add to ProviderResponse first, copy to audit field. | |

**User's choice:** Typed TokenUsage model

**Follow-up: Should TokenUsage also be on ProviderResponse?**

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — on ProviderResponse too | Adapters extract tokens, set ProviderResponse.token_usage. Runner copies to audit. | ✓ |
| No — audit-only | Token extraction in AgentRunner from ProviderResponse.raw. | |

**User's choice:** Yes — on ProviderResponse too
**Notes:** Clean data flow: provider → response → audit.

---

## Config & Credentials

| Option | Description | Selected |
|--------|-------------|----------|
| LLMSettings sub-model | New sub-model with default_provider, keys, timeout, retries. | |
| Per-provider sub-models | Separate OpenAISettings, AnthropicSettings. | |
| Flat on ZerothSettings | Top-level ZEROTH_OPENAI_API_KEY fields. | |

**User's choice:** (Custom) No global LLM defaults — each agent node specifies its own provider/model. Use a universal client like LiteLLM via LangChain's ChatLiteLLM wrapper.

**Follow-up: Universal provider approach**

| Option | Description | Selected |
|--------|-------------|----------|
| LiteLLM | Use litellm directly. 100+ providers, normalized token usage. | |
| LangChain ChatModel | Use LangChain's BaseChatModel with langchain-openai/anthropic. | |
| LiteLLM + LangChain ChatLiteLLM | LangChain's ChatLiteLLM wrapper combining both. | ✓ |
| Custom thin wrapper | Build own dispatcher. Full control, more maintenance. | |

**User's choice:** LiteLLM via LangChain's ChatLiteLLM wrapper

**Follow-up: API key management**

| Option | Description | Selected |
|--------|-------------|----------|
| Standard env vars | LiteLLM reads OPENAI_API_KEY etc. directly. No ZEROTH_ prefix. | ✓ |
| ZerothSettings with passthrough | ZEROTH_ prefixed keys passed to LiteLLM. | |
| Secrets provider integration | Use zeroth.secrets for runtime key resolution. | |

**User's choice:** Standard env vars (Recommended)

---

## Agent Node Model Config

| Option | Description | Selected |
|--------|-------------|----------|
| model field on AgentConfig | Required 'model: str' using LiteLLM format. Clear, explicit, per-node. | ✓ |
| model in prompt_config | Model in existing PromptConfig sub-model. Less discoverable. | |
| model on graph node definition | Model from workflow definition at authoring time. | |

**User's choice:** model field on AgentConfig (Recommended)

---

## Retry Configurability

| Option | Description | Selected |
|--------|-------------|----------|
| Global LLMSettings defaults | Default retry params in ZerothSettings. All nodes share. | |
| Per-node on AgentConfig | Each node sets own retry params. Different latency tolerances. | ✓ |
| On the adapter itself | Adapter constructor takes retry params. Single instance shared. | |

**User's choice:** Per-node on AgentConfig

**Follow-up: Which errors are retryable?**

| Option | Description | Selected |
|--------|-------------|----------|
| Rate limit + server errors | Retry 429, 500, 502, 503, 529, connection/timeout errors. | |
| Rate limit only | Only retry 429. Most conservative. | |
| You decide | Claude determines based on LiteLLM best practices. | ✓ |

**User's choice:** You decide

---

## Testing Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Mock LiteLLM responses | Mock litellm.acompletion() for canned responses. Fast, deterministic. | ✓ |
| Recorded responses (VCR) | Record real API responses, replay in tests. | |
| Optional live tests (marker-gated) | @pytest.mark.live only when API keys present. | ✓ |

**User's choice:** Mock LiteLLM responses + Optional live tests (both selected)
**Notes:** Layered strategy — mocks for CI/unit tests, live tests for manual validation.

---

## GovernAI Adapter Coexistence

| Option | Description | Selected |
|--------|-------------|----------|
| LiteLLM primary, GovernAI wraps it | LiteLLMProviderAdapter is standard. GovernAI wraps it when governance enabled. | ✓ |
| Keep both independent | Separate independent choices. No coupling. | |
| Replace GovernAI adapter | Deprecate GovernedLLMProviderAdapter. GovernAI as middleware. | |

**User's choice:** LiteLLM is primary, GovernAI wraps it (Recommended)

---

## Claude's Discretion

- Retryable error set (standard provider best practices)
- TokenUsage field naming and optional fields
- LiteLLM configuration details (timeouts, custom headers)
- Whether an LLMSettings sub-model for retry defaults is warranted

## Deferred Ideas

- **LLM-05:** Model fallback chains — deferred to future phase
- **LLM-06:** Streaming response support — out of scope
- **ECON-01:** Regulus instrumentation — belongs in Phase 13
- **Global LLM defaults** — can be added later without breaking per-node config
