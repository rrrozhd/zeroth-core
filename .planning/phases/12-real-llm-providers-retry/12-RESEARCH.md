# Phase 12: Real LLM Providers & Retry - Research

**Researched:** 2026-04-07
**Domain:** LLM provider integration (LiteLLM + LangChain), retry strategies, token usage auditing
**Confidence:** MEDIUM-HIGH

## Summary

Phase 12 replaces test provider stubs with a single `LiteLLMProviderAdapter` that uses LangChain's `ChatLiteLLM` wrapper to route calls to OpenAI, Anthropic, and 100+ other providers. LiteLLM's model format strings (`openai/gpt-4o`, `anthropic/claude-sonnet-4-5-20250514`) handle implicit provider routing, so no per-provider adapter code is needed. The existing `ProviderAdapter` protocol (`ainvoke(ProviderRequest) -> ProviderResponse`) is the integration seam -- the new adapter implements it directly.

The existing `RetryPolicy` on `AgentConfig` already supports `max_retries`, `backoff_seconds`, and toggles for retrying on provider errors/timeouts. Phase 12 must enhance the backoff from fixed-delay to exponential with jitter, and add error classification to distinguish retryable transient failures (429, 503, 500, timeouts) from permanent failures (401, 403, 400). Token usage is available from both LiteLLM's `ModelResponse.usage` object and LangChain's `AIMessage.usage_metadata` -- the adapter extracts these into a new `TokenUsage` pydantic model that flows through `ProviderResponse` into `NodeAuditRecord`.

**Primary recommendation:** Use `litellm.acompletion()` directly in the `LiteLLMProviderAdapter` rather than LangChain's `ChatLiteLLM` class. This gives direct access to the standardized `ModelResponse.usage` object (prompt_tokens, completion_tokens, total_tokens), simpler error handling via litellm's mapped exception types, and avoids LangChain's AIMessage layer which complicates token extraction. Reserve `ChatLiteLLM` for `GovernedLLMProviderAdapter` where LangChain interface compatibility with GovernAI is actually needed.

**IMPORTANT CAVEAT on D-01:** The user decision D-01 specifies "LangChain's ChatLiteLLM wrapper as the universal provider interface." The research recommends a refinement: use `litellm.acompletion()` for the base `LiteLLMProviderAdapter` (better token access, simpler retry control), and ChatLiteLLM only when wrapping for governance. If the planner must honor D-01 literally, ChatLiteLLM can work -- token usage is accessible via `AIMessage.response_metadata["token_usage"]` or `AIMessage.usage_metadata`, but extraction is less direct. Both approaches implement `ProviderAdapter` protocol identically.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Use LangChain's `ChatLiteLLM` wrapper as the universal provider interface -- LiteLLM handles provider routing (OpenAI, Anthropic, 100+ others), LangChain interface maintains GovernAI compatibility
- **D-02:** Single `LiteLLMProviderAdapter` implementing the existing `ProviderAdapter` protocol -- replaces the need for separate per-provider adapters
- **D-03:** `LiteLLMProviderAdapter` is the primary adapter. `GovernedLLMProviderAdapter` wraps it when governance is enabled on a node -- users choose governed vs ungoverned per node
- **D-04:** No global LLM defaults in `ZerothSettings` -- each agent node specifies its own provider and model
- **D-05:** Required `model: str` field added to `AgentConfig` using LiteLLM format strings (e.g., `openai/gpt-4o`, `anthropic/claude-sonnet-4-5-20250514`)
- **D-06:** Per-node retry configuration on `AgentConfig` -- max_retries, base_delay, max_delay. Different nodes may have different latency tolerances
- **D-07:** Retry-only in Phase 12 -- exponential backoff with jitter on transient failures. No model fallback chains (deferred to LLM-05)
- **D-08:** API keys managed via standard environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.) -- LiteLLM reads these directly, no `ZEROTH_` prefix wrapping
- **D-09:** New typed `TokenUsage` pydantic model with `input_tokens`, `output_tokens`, `total_tokens`, `model_name` fields
- **D-10:** `TokenUsage` added to `ProviderResponse` -- each adapter extracts tokens from the raw SDK response and populates it
- **D-11:** `TokenUsage` also added as a dedicated typed field on `NodeAuditRecord` -- `AgentRunner` copies from `ProviderResponse` to audit record
- **D-12:** Unit tests mock `litellm.acompletion()` to return canned responses -- validates adapter logic without network calls
- **D-13:** Optional live integration tests gated behind `@pytest.mark.live` -- only run when API keys are present, CI skips them

### Claude's Discretion
- Exact retryable error set (rate limits, server errors, timeouts)
- `TokenUsage` field naming and optional fields beyond input/output/total
- Connection pooling and session management within LiteLLM
- How `LiteLLMProviderAdapter` handles LiteLLM-specific configuration (timeouts, custom headers)
- Whether to add an `LLMSettings` sub-model for retry defaults or keep it purely per-node

### Deferred Ideas (OUT OF SCOPE)
- **LLM-05: Model fallback chains** -- ordered list of models tried in sequence on failure
- **LLM-06: Streaming response support** -- async streaming for agent nodes
- **ECON-01: Regulus instrumentation wrapper** -- wrapping with cost events belongs in Phase 13
- **Global LLM defaults / LLMSettings** -- can be added later without breaking per-node config
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| LLM-01 | OpenAI provider adapter implements ProviderAdapter protocol via langchain-openai | Covered by LiteLLMProviderAdapter using LiteLLM model strings (`openai/gpt-4o`). LiteLLM routes to OpenAI internally. No separate langchain-openai needed. |
| LLM-02 | Anthropic provider adapter implements ProviderAdapter protocol via langchain-anthropic | Covered by same LiteLLMProviderAdapter using `anthropic/claude-*` model strings. LiteLLM routes to Anthropic internally. |
| LLM-03 | Provider calls retry with exponential backoff and jitter on rate limits and transient failures | Enhanced `RetryPolicy` with exponential backoff + jitter. Classify litellm exceptions by status code (429, 500, 503, 408 = retryable). |
| LLM-04 | Token usage (input/output) captured from provider responses and attached to node audit records | `TokenUsage` pydantic model extracted from `ModelResponse.usage` (or `AIMessage.usage_metadata`), added to `ProviderResponse`, copied to `NodeAuditRecord` by `AgentRunner`. |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| litellm | >=1.63,<2.0 | Universal LLM provider routing | Routes 100+ providers through single API; standardizes response format including token usage. Pin above 1.63 for stable async. AVOID 1.82.7-1.82.8 (supply chain compromise, resolved in 1.83.0+). |
| langchain-litellm | >=0.3.4 | ChatLiteLLM wrapper for LangChain interface | Bridges LiteLLM routing with LangChain BaseChatModel interface needed for GovernAI compatibility. Replaces deprecated langchain-community ChatLiteLLM. |
| tenacity | >=8.2 | Retry decorator with exponential backoff + jitter | Battle-tested retry library. Avoids hand-rolling backoff math. Already a transitive dep of litellm. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| openai | >=1.0 | LiteLLM transitive dependency | Installed automatically by litellm. Exception types (RateLimitError, etc.) inherit from openai exceptions. |
| anthropic | >=0.30 | LiteLLM transitive dependency | Installed automatically when using anthropic models. |
| tokenizers | (transitive) | Fallback token counting | Used by litellm internally for providers that don't return usage. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| litellm | Direct openai + anthropic SDKs | Requires separate adapters per provider; litellm unifies interface |
| tenacity | Hand-rolled asyncio.sleep loop | Error-prone; tenacity handles edge cases (jitter, stop conditions, logging) |
| langchain-litellm | Raw litellm.acompletion() | Loses LangChain BaseChatModel interface needed for GovernAI; but simpler for base adapter |

**Installation:**
```bash
uv add "litellm>=1.63,<2.0" "langchain-litellm>=0.3.4" "tenacity>=8.2"
```

**Security note:** LiteLLM versions 1.82.7 and 1.82.8 were compromised in a supply chain attack (March 2026). Version 1.83.0+ includes new CI/CD security. Pin `>=1.63,<2.0` and verify installed version is not in the compromised range.

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/
├── agent_runtime/
│   ├── provider.py          # ProviderAdapter protocol + all adapters (add LiteLLMProviderAdapter)
│   ├── models.py            # AgentConfig (add model field), RetryPolicy (enhance for exp backoff)
│   ├── runner.py            # AgentRunner (add token_usage extraction to audit flow)
│   ├── retry.py             # NEW: retry_with_backoff() using tenacity, error classification
│   └── errors.py            # Add ProviderRateLimitError, ProviderTransientError subtypes
├── audit/
│   └── models.py            # NodeAuditRecord (add token_usage field), TokenUsage model
└── config/
    └── settings.py          # No changes (D-04: no global LLM defaults)
```

### Pattern 1: LiteLLMProviderAdapter
**What:** Single adapter implementing `ProviderAdapter` protocol that calls `litellm.acompletion()` and normalizes the response into `ProviderResponse`.
**When to use:** For all agent nodes that need real LLM calls without governance wrapping.
**Example:**
```python
# Source: litellm docs + existing ProviderAdapter pattern
import litellm
from zeroth.agent_runtime.provider import ProviderAdapter, ProviderRequest, ProviderResponse
from zeroth.audit.models import TokenUsage

class LiteLLMProviderAdapter:
    """Adapter that calls litellm.acompletion() for any supported provider."""

    def __init__(self, *, default_timeout: float = 600.0) -> None:
        self._default_timeout = default_timeout

    async def ainvoke(self, request: ProviderRequest) -> ProviderResponse:
        response = await litellm.acompletion(
            model=request.model_name,  # e.g., "openai/gpt-4o"
            messages=self._normalize_messages(request.messages),
            timeout=self._default_timeout,
        )
        token_usage = self._extract_token_usage(response, request.model_name)
        content = response.choices[0].message.content
        tool_calls = self._extract_tool_calls(response)
        return ProviderResponse(
            content=content,
            raw=response,
            tool_calls=tool_calls,
            token_usage=token_usage,
            metadata={"provider": "litellm", "model": response.model},
        )

    def _extract_token_usage(self, response, model_name: str) -> TokenUsage | None:
        usage = getattr(response, "usage", None)
        if usage is None:
            return None
        return TokenUsage(
            input_tokens=usage.prompt_tokens or 0,
            output_tokens=usage.completion_tokens or 0,
            total_tokens=usage.total_tokens or 0,
            model_name=model_name,
        )
```

### Pattern 2: Exponential Backoff with Jitter via Tenacity
**What:** A reusable retry wrapper that classifies litellm exceptions and retries transient ones with exponential backoff + full jitter.
**When to use:** Wrapping the provider call in `AgentRunner.run()`.
**Example:**
```python
# Source: tenacity docs + litellm exception mapping
import random
import litellm
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception

RETRYABLE_STATUS_CODES = {408, 429, 500, 502, 503, 504}

def is_retryable_provider_error(exc: BaseException) -> bool:
    """Classify litellm exceptions as retryable or permanent."""
    if isinstance(exc, litellm.RateLimitError):
        return True  # 429
    if isinstance(exc, (litellm.ServiceUnavailableError, litellm.InternalServerError)):
        return True  # 503, 500
    if isinstance(exc, litellm.Timeout):
        return True  # 408
    if isinstance(exc, litellm.APIConnectionError):
        return True  # Network issues
    # All other litellm exceptions (AuthenticationError, BadRequestError, etc.) are permanent
    return False

async def call_with_retry(
    adapter, request, *, max_retries: int = 3, base_delay: float = 1.0, max_delay: float = 60.0
):
    """Call adapter.ainvoke with exponential backoff + jitter on transient failures."""
    @retry(
        retry=retry_if_exception(is_retryable_provider_error),
        stop=stop_after_attempt(max_retries + 1),
        wait=wait_exponential_jitter(initial=base_delay, max=max_delay, jitter=base_delay),
        reraise=True,
    )
    async def _invoke():
        return await adapter.ainvoke(request)
    return await _invoke()
```

### Pattern 3: TokenUsage Flow (Provider -> Response -> Audit)
**What:** Typed token usage data flowing from provider SDK response through the adapter into audit records.
**When to use:** Every successful provider call.
**Example:**
```python
# Source: existing audit model pattern + D-09/D-10/D-11
from pydantic import BaseModel, ConfigDict, Field

class TokenUsage(BaseModel):
    """Token consumption for a single LLM call."""
    model_config = ConfigDict(extra="forbid")

    input_tokens: int = 0
    output_tokens: int = 0
    total_tokens: int = 0
    model_name: str = ""

# On ProviderResponse (D-10):
class ProviderResponse(BaseModel):
    # ... existing fields ...
    token_usage: TokenUsage | None = None

# On NodeAuditRecord (D-11):
class NodeAuditRecord(BaseModel):
    # ... existing fields ...
    token_usage: TokenUsage | None = None

# In AgentRunner.run() after successful provider call:
# record["token_usage"] = response.token_usage.model_dump() if response.token_usage else None
```

### Anti-Patterns to Avoid
- **Per-provider adapter classes:** LiteLLM's model string routing eliminates this need. Do NOT create `OpenAIProviderAdapter` and `AnthropicProviderAdapter` separately.
- **Retry inside the adapter:** Keep retry logic outside the adapter (in `AgentRunner` or a wrapper function), not inside `LiteLLMProviderAdapter.ainvoke()`. The adapter should be a single-shot call. This allows different nodes to have different retry policies.
- **Catching all exceptions for retry:** Only retry transient errors. Retrying `AuthenticationError` or `BadRequestError` wastes time and masks configuration problems.
- **Fixed backoff without jitter:** Multiple clients retrying at identical intervals create thundering herd effects. Always add jitter.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Provider routing (OpenAI/Anthropic/etc.) | Per-provider adapter classes with SDK imports | `litellm.acompletion()` with model string routing | LiteLLM handles auth, endpoints, response normalization for 100+ providers |
| Exponential backoff + jitter | Custom sleep loop with `asyncio.sleep()` | `tenacity` with `wait_exponential_jitter` | Edge cases: max delay caps, jitter distribution, stop conditions, logging |
| Exception classification | Manual HTTP status code parsing | `litellm` mapped exception types | LiteLLM already maps all provider errors to consistent types (RateLimitError, ServiceUnavailableError, etc.) |
| Token counting fallback | Tiktoken tokenizer calls | LiteLLM's built-in usage tracking | LiteLLM handles provider-specific token counting and falls back to local tokenizers |

**Key insight:** LiteLLM's entire value proposition is normalizing the messy reality of LLM provider APIs. Every provider has different error formats, auth flows, response shapes, and token counting. LiteLLM handles all of this and presents a unified interface.

## Common Pitfalls

### Pitfall 1: LiteLLM Supply Chain Attack (March 2026)
**What goes wrong:** Versions 1.82.7-1.82.8 contained a credential harvester and backdoor.
**Why it happens:** CI/CD compromise by TeamPCP threat actor.
**How to avoid:** Pin `litellm>=1.83` (fixed CI/CD pipeline). Verify installed version after `uv add`. Never use `--no-verify` flags.
**Warning signs:** Unexpected network connections, credentials appearing in logs.

### Pitfall 2: Retrying Non-Transient Errors
**What goes wrong:** `AuthenticationError` (401) or `BadRequestError` (400) gets retried, wasting time and rate limit budget.
**Why it happens:** Catch-all exception handler doesn't classify errors.
**How to avoid:** Use `is_retryable_provider_error()` classification function. Only retry 408, 429, 500, 502, 503, 504 status codes.
**Warning signs:** Retry loops exhausting on every call (not just under load).

### Pitfall 3: Missing Token Usage on Some Providers
**What goes wrong:** `response.usage` is None for some providers or when the API call fails mid-stream.
**Why it happens:** Not all providers return usage data consistently. Anthropic and OpenAI do, but some smaller providers may not.
**How to avoid:** Make `token_usage` optional (`TokenUsage | None`) on both `ProviderResponse` and `NodeAuditRecord`. Never assume it's present.
**Warning signs:** `AttributeError` or `NoneType` errors when extracting token counts.

### Pitfall 4: AgentConfig.model_name Collision with D-05
**What goes wrong:** `AgentConfig` already has a `model_name: str` field. D-05 says add a `model: str` field with LiteLLM format strings.
**Why it happens:** The existing `model_name` is used in `ProviderRequest` construction.
**How to avoid:** Either repurpose existing `model_name` to accept LiteLLM format strings, or add `model` as a new field and deprecate/alias `model_name`. The simplest path: repurpose `model_name` since it already flows into `ProviderRequest.model_name` which goes to `litellm.acompletion(model=...)`.
**Warning signs:** Two fields with similar names causing confusion.

### Pitfall 5: Existing RetryPolicy Conflict
**What goes wrong:** `AgentRunner.run()` already has a retry loop with `RetryPolicy` (lines 124-198 of runner.py). Adding tenacity-based retry creates double retry layers.
**Why it happens:** The existing retry loop handles validation errors, provider errors, AND timeouts with a fixed backoff. Phase 12 needs exponential backoff with jitter specifically for transient provider errors.
**How to avoid:** Two options: (a) Enhance the existing retry loop to support exponential backoff + jitter and error classification, OR (b) Move transient-error retry into a wrapper around the provider call (between runner and adapter), keeping the existing loop for validation retries only. Option (a) is simpler and avoids nesting.
**Warning signs:** Exponential blowup of retries (3 retries x 3 retries = 9 calls instead of 3).

### Pitfall 6: GovernedLLMProviderAdapter Wiring
**What goes wrong:** D-03 says `GovernedLLMProviderAdapter` should wrap `LiteLLMProviderAdapter`. But the current implementation wraps a `GovernedLLM` object directly.
**Why it happens:** The existing adapter was built for GovernAI's LangChain-based interface, not litellm.
**How to avoid:** The planner needs to decide: either (a) update `GovernedLLMProviderAdapter` to compose `LiteLLMProviderAdapter` + governance, or (b) have `GovernedLLMProviderAdapter` use `ChatLiteLLM` (LangChain interface) which GovernAI wraps natively. Option (b) is more natural for GovernAI.
**Warning signs:** GovernAI governance checks not running because the LLM call bypasses the GovernedLLM wrapper.

## Code Examples

### LiteLLM acompletion Response Structure
```python
# Source: https://docs.litellm.ai/docs/completion/output
# litellm.acompletion() returns a ModelResponse with OpenAI-compatible structure:
response = await litellm.acompletion(
    model="openai/gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
# response.choices[0].message.content -> str
# response.choices[0].message.tool_calls -> list | None
# response.choices[0].finish_reason -> str
# response.model -> str (actual model used)
# response.usage.prompt_tokens -> int
# response.usage.completion_tokens -> int
# response.usage.total_tokens -> int
```

### LiteLLM Exception Hierarchy
```python
# Source: https://docs.litellm.ai/docs/exception_mapping
# All exceptions inherit from openai exception types.
# Each has: status_code, message, llm_provider attributes
import litellm

# Retryable (transient):
litellm.RateLimitError        # 429
litellm.ServiceUnavailableError  # 503
litellm.InternalServerError   # 500+
litellm.Timeout               # 408
litellm.APIConnectionError    # 500 (network)

# NOT retryable (permanent):
litellm.AuthenticationError   # 401
litellm.BadRequestError       # 400
litellm.PermissionDeniedError # 403
litellm.NotFoundError         # 404
litellm.ContextWindowExceededError  # 400 (prompt too long)
litellm.ContentPolicyViolationError # 400 (content filter)
```

### Tenacity Exponential Backoff with Jitter
```python
# Source: tenacity docs
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception

@retry(
    retry=retry_if_exception(is_retryable_provider_error),
    stop=stop_after_attempt(4),  # 1 initial + 3 retries
    wait=wait_exponential_jitter(
        initial=1.0,    # base_delay: first retry after ~1s
        max=60.0,       # max_delay: cap at 60s
        jitter=2.0,     # random jitter up to 2s added
    ),
    reraise=True,  # re-raise the last exception after exhaustion
)
async def call_provider(adapter, request):
    return await adapter.ainvoke(request)
```

### Existing AgentConfig model_name Usage
```python
# Source: src/zeroth/agent_runtime/models.py line 75
# Current: model_name: str  -- already on AgentConfig
# Current usage in runner.py line 129:
request = ProviderRequest(
    model_name=self.config.model_name,  # flows directly to litellm
    messages=messages,
    metadata=prompt.metadata,
)
# LiteLLM accepts format: "openai/gpt-4o", "anthropic/claude-sonnet-4-5-20250514"
# So existing model_name can be repurposed to accept LiteLLM format strings.
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| langchain-community ChatLiteLLM | langchain-litellm package | 2025 (deprecated in langchain-community 0.3.24) | Must use `langchain-litellm` import, not `langchain_community` |
| litellm.completion() sync | litellm.acompletion() async | Stable since 2024 | Async is preferred for server workloads; uses native async clients per provider |
| Manual per-provider SDKs | LiteLLM unified routing | 2024-2025 | Single `model="provider/model"` string replaces separate SDK setup |
| litellm CI (pre-compromise) | litellm CI v2 pipeline | March 2026 (v1.83.0) | New security gates after supply chain attack; use >=1.83 |

**Deprecated/outdated:**
- `langchain_community.chat_models.litellm.ChatLiteLLM`: Deprecated. Use `langchain_litellm.ChatLiteLLM` from the `langchain-litellm` package.
- `litellm` 1.82.7-1.82.8: COMPROMISED. Never install these versions.

## Open Questions

1. **D-01 vs Direct litellm: Interpretation scope**
   - What we know: D-01 says "use LangChain's ChatLiteLLM wrapper as the universal provider interface." Direct `litellm.acompletion()` gives simpler token access.
   - What's unclear: Whether the user intended ChatLiteLLM for ALL calls or just for the governance integration path.
   - Recommendation: The planner should use `litellm.acompletion()` for `LiteLLMProviderAdapter` (cleaner) and `ChatLiteLLM` for `GovernedLLMProviderAdapter` (needs LangChain interface). If strict D-01 adherence is required, ChatLiteLLM works for both -- just requires more work to extract token usage from `AIMessage.usage_metadata`.

2. **Double retry layer risk**
   - What we know: `AgentRunner.run()` already has a retry loop (lines 124-198). Adding tenacity retry around the provider call would create nested retries.
   - What's unclear: Whether to enhance the existing loop or replace it for provider errors.
   - Recommendation: Enhance the existing `RetryPolicy` model to support exponential backoff with jitter (`backoff_strategy: "fixed" | "exponential"`, `jitter: bool`). Modify the existing sleep logic in the runner to compute delay accordingly. This avoids nested retry layers entirely.

3. **model_name field naming**
   - What we know: D-05 says "add `model: str` field" but `model_name: str` already exists on `AgentConfig` and flows to `ProviderRequest.model_name`.
   - What's unclear: Whether to rename, alias, or keep both.
   - Recommendation: Keep `model_name` as-is. It already does what D-05 needs -- just start passing LiteLLM format strings into it. No rename needed, no breaking changes.

## Sources

### Primary (HIGH confidence)
- [LiteLLM Output Docs](https://docs.litellm.ai/docs/completion/output) - ModelResponse structure, usage object fields
- [LiteLLM Exception Mapping](https://docs.litellm.ai/docs/exception_mapping) - Full exception hierarchy, status codes, retry eligibility
- [LiteLLM Input Params](https://docs.litellm.ai/docs/completion/input) - acompletion signature, num_retries, timeout params
- [LiteLLM Reliability](https://docs.litellm.ai/docs/completion/reliable_completions) - Built-in retry mechanics
- [LiteLLM Getting Started](https://docs.litellm.ai/docs/) - Model format strings, provider routing
- [LiteLLM Security Update](https://docs.litellm.ai/blog/security-update-march-2026) - Supply chain attack details and remediation

### Secondary (MEDIUM confidence)
- [langchain-litellm PyPI](https://pypi.org/project/langchain-litellm/) - Package version (0.3.5), migration from deprecated langchain-community
- [LangChain ChatLiteLLM Reference](https://reference.langchain.com/python/langchain-litellm/chat_models/litellm/ChatLiteLLM) - ChatLiteLLM class parameters, max_retries
- [LangChain UsageMetadata](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.ai.UsageMetadata.html) - Token usage access from AIMessage

### Tertiary (LOW confidence)
- [LiteLLM GitHub Feature Request #16068](https://github.com/BerriAI/litellm/issues/16068) - Exponential backoff for completion() (suggests it's only natively in Router, not completion)
- [LiteLLM DeepWiki Retries](https://deepwiki.com/BerriAI/litellm/7.1-fallbacks-and-retries) - Community documentation on retry behavior

## Metadata

**Confidence breakdown:**
- Standard stack: MEDIUM-HIGH - litellm and langchain-litellm are well-documented; version pins verified against security advisory. Could not verify exact latest version via pip.
- Architecture: HIGH - Existing codebase patterns are clear. ProviderAdapter protocol, AgentRunner flow, and audit models are well-understood from source code review.
- Pitfalls: HIGH - Supply chain attack is well-documented. Double retry risk identified from source code. Exception classification verified from official litellm docs.

**Research date:** 2026-04-07
**Valid until:** 2026-04-21 (14 days -- litellm moves fast, check for new releases)
