# Phase 37: Context Window Management - Research

**Researched:** 2026-04-12
**Domain:** LLM context window tracking, message compaction, token counting
**Confidence:** HIGH

## Summary

Phase 37 adds automatic context window management to the agent runtime. Agent threads track accumulated token usage via `litellm.token_counter` (already in dependencies at v1.83.0) and apply configurable compaction strategies before context overflow. Three built-in strategies are required: truncation, observation masking (default), and LLM-based summarization (opt-in). The feature integrates into the existing `AgentRunner.run()` loop between prompt assembly and LLM invocation, persists compaction results via `ThreadStateStore`, and is configured per-agent-node via `AgentNodeData`.

The codebase is well-prepared for this feature. The `messages` list in `runner.py` (line 128) is the accumulation point that needs counting. `litellm.token_counter` accepts dicts, LangChain message objects, and Pydantic models interchangeably -- all types present in the runner's message list. The project's established Protocol pattern for pluggable interfaces (12+ existing Protocols) directly applies to the `CompactionStrategy` interface. The `templates` package (Phase 36) provides a clean structural template: models.py, errors.py, registry-like module, __init__.py with explicit re-exports.

**Primary recommendation:** Create `zeroth.core.context_window` package following the `templates` module pattern. Token tracker wraps `litellm.token_counter`. Compaction happens in `AgentRunner.run()` after prompt assembly but before the retry loop. Strategy selection is Protocol-based with three built-in implementations. Compacted messages replace the original messages list in-place, and results persist via `ThreadStateStore.checkpoint()`.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Token counting uses `litellm.token_counter` (already in dependencies via the LLM provider layer). Called after each LLM invocation to update the thread's accumulated token count.
- **D-02:** Token count is stored per-thread in `ThreadState` (or equivalent thread memory model). Updated after each agent turn (user message + assistant response).
- **D-03:** Per-agent-node configurable settings: `max_context_tokens` (int), `summary_trigger_ratio` (float, default 0.8), `compaction_strategy` (str, default "observation_masking"), `preserve_recent_messages_count` (int, default 4).
- **D-04:** Three built-in strategies, all implementing a `CompactionStrategy` Protocol:
  - `truncation` -- Drop oldest messages (preserving system message and recent N messages)
  - `observation_masking` -- Replace tool/observation outputs in older messages with `[output omitted -- N tokens]` placeholders (default strategy per REQUIREMENTS.md)
  - `llm_summarization` -- Condense older messages into a summary message via LLM call (opt-in only per REQUIREMENTS.md)
- **D-05:** Strategy is pluggable per agent node -- the Protocol allows custom strategies to be registered.
- **D-06:** Compaction triggers when `accumulated_tokens / max_context_tokens >= summary_trigger_ratio` -- checked before each LLM invocation.
- **D-07:** Compaction results (compacted messages, summary if applicable) are stored in thread memory via the existing `ThreadStateStore` so they persist across runs.
- **D-08:** Original messages can optionally be archived for audit retrieval -- controlled by a `archive_originals` boolean on the compaction settings (default: false).
- **D-09:** Compaction runs in the agent runtime loop, after prompt assembly but before LLM invocation. The `AgentRunner` (or equivalent) checks token count and applies compaction if needed.
- **D-10:** New `zeroth.core.context_window` package with models.py, strategies.py, tracker.py, errors.py, __init__.py.
- **D-11:** `ContextWindowSettings` added to agent node config (not global ZerothSettings) -- per-node configuration.

### Claude's Discretion
- Whether to add a `CompactionResult` model for tracking what was compacted
- How to handle the LLM summarization strategy's own token cost (deduct from budget)
- Test approach for litellm.token_counter (mock vs real)

### Deferred Ideas (OUT OF SCOPE)
- Automatic context summarization as default -- explicitly out of scope per REQUIREMENTS.md (observation masking outperforms, summarization is opt-in only)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CTXW-01 | Approximate token count of accumulated agent messages per thread is tracked using the LLM provider's tokenizer (via litellm.token_counter), updated after each LLM invocation | litellm.token_counter verified working at v1.83.0; accepts all message types in runner's list; returns int count |
| CTXW-02 | When token count exceeds a configurable threshold, a compaction strategy is applied before the next LLM invocation (default: observation masking of older messages) | Runner.run() has clear insertion point at line 128 (after prompt assembly, before retry loop); threshold check is ratio-based per D-06 |
| CTXW-03 | Compaction strategy is pluggable per agent node with three built-in strategies: truncation (drop oldest), observation masking (replace tool outputs with placeholders), and LLM-based summarization (condense older messages) | Protocol pattern well-established in codebase (12+ examples); AgentNodeData already has per-node config fields |
| CTXW-04 | Compaction results are stored in thread memory so they persist across runs; original messages can optionally be archived for audit retrieval | ThreadStateStore.checkpoint() already stores arbitrary dict state; InMemoryThreadStateStore keeps history; RepositoryThreadStateStore persists to DB |
| CTXW-05 | Per-agent-node settings are configurable: max_context_tokens, summary_trigger_ratio, compaction_strategy, and preserve_recent_messages_count | AgentNodeData on graph/models.py is the canonical config location; follows same pattern as template_ref (Phase 36) |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| litellm | 1.83.0 (installed) | Token counting via `litellm.token_counter` | Already in dependency tree; handles all LiteLLM model formats; falls back to cl100k_base for unknown models [VERIFIED: pip show + runtime test] |
| pydantic | 2.x (installed) | Data models for settings, results, compaction state | Project standard; ConfigDict(extra="forbid") pattern [VERIFIED: codebase inspection] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| litellm.get_model_info | 1.83.0 | Look up model's max_input_tokens | Optional auto-detection of max_context_tokens; unreliable for newer models [VERIFIED: runtime test -- fails for newer Claude models] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| litellm.token_counter | tiktoken directly | tiktoken only handles OpenAI models; litellm abstracts tokenizer selection per model -- use litellm |
| Per-message token counting | Counting entire message list at once | Per-message is more accurate for compaction targeting but slower; counting entire list is simpler and sufficient for threshold detection |

**Installation:**
```bash
# No new packages needed -- all dependencies already installed
uv sync
```

**Version verification:** litellm 1.83.0 confirmed installed and `token_counter` function verified working with dict, LangChain, and Pydantic message formats. [VERIFIED: runtime tests in research session]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/context_window/
    __init__.py          # Public API re-exports
    models.py            # ContextWindowSettings, CompactionResult, CompactionState
    strategies.py        # CompactionStrategy Protocol + 3 built-in implementations
    tracker.py           # ContextWindowTracker (token counting + threshold detection)
    errors.py            # ContextWindowError hierarchy
```

### Pattern 1: CompactionStrategy Protocol
**What:** A runtime-checkable Protocol that all compaction strategies implement. Each strategy takes the current messages list and settings, returns a compacted messages list.
**When to use:** Always -- this is the core abstraction per D-04/D-05.
**Example:**
```python
# Source: Established project pattern from ProviderAdapter, ThreadStateStore, etc.
from typing import Any, Protocol, runtime_checkable

@runtime_checkable
class CompactionStrategy(Protocol):
    """Interface for context window compaction strategies."""

    async def compact(
        self,
        messages: list[Any],
        *,
        settings: "ContextWindowSettings",
        model_name: str,
    ) -> "CompactionResult":
        """Compact messages to fit within the context window budget."""
        ...
```

### Pattern 2: Tracker as Coordinator
**What:** `ContextWindowTracker` wraps `litellm.token_counter`, checks thresholds, and coordinates strategy execution. Injected into `AgentRunner` similar to how `budget_enforcer` is injected.
**When to use:** The tracker is the single entry point the runner calls. It encapsulates the "should I compact?" + "do the compaction" logic.
**Example:**
```python
# Source: Pattern modeled on budget_enforcer injection in runner.py
class ContextWindowTracker:
    """Tracks token usage and applies compaction when needed."""

    def __init__(
        self,
        settings: ContextWindowSettings,
        strategy: CompactionStrategy,
    ) -> None:
        self.settings = settings
        self.strategy = strategy
        self._accumulated_tokens: int = 0

    def count_tokens(self, messages: list[Any], model_name: str) -> int:
        """Count tokens in the message list using litellm."""
        return litellm.token_counter(model=model_name, messages=self._normalize(messages))

    def needs_compaction(self, token_count: int) -> bool:
        """Check if token count exceeds the trigger threshold."""
        if self.settings.max_context_tokens <= 0:
            return False
        ratio = token_count / self.settings.max_context_tokens
        return ratio >= self.settings.summary_trigger_ratio

    async def maybe_compact(
        self,
        messages: list[Any],
        model_name: str,
    ) -> tuple[list[Any], "CompactionResult | None"]:
        """Count tokens and compact if threshold exceeded."""
        token_count = self.count_tokens(messages, model_name)
        if not self.needs_compaction(token_count):
            return messages, None
        result = await self.strategy.compact(
            messages, settings=self.settings, model_name=model_name,
        )
        return result.messages, result
```

### Pattern 3: Runner Integration Point
**What:** Compaction check inserted in `AgentRunner.run()` after prompt assembly (line 128) but before the retry loop (line 149). Follows the same injection pattern as `budget_enforcer`.
**When to use:** Every agent run where context_window settings are configured.
**Integration sketch:**
```python
# In AgentRunner.run(), after line 128:
# messages: list[Any] = list(prompt.messages)
if self.context_tracker is not None:
    messages, compaction_result = await self.context_tracker.maybe_compact(
        messages, self.config.model_name,
    )
    # Also compact after tool call resolution (inside the retry loop)
```

### Pattern 4: Observation Masking Strategy
**What:** The default strategy. Iterates older messages (outside the preserve window), finds tool/observation messages with large `content` fields, replaces content with `[output omitted -- N tokens]`.
**When to use:** Default strategy per REQUIREMENTS.md. Most token-efficient because it targets the highest-token messages (tool outputs) without losing conversation structure.
**Key insight:** Tool/observation messages in the runner contain the full JSON output from tool executions. A single tool response can be thousands of tokens (e.g., database query results, API responses). Masking these preserves the fact that a tool was called and what arguments were used, while removing the bulky output.

### Anti-Patterns to Avoid
- **Counting tokens on every message individually then summing:** LiteLLM's `token_counter` with `messages=` parameter already handles the full list efficiently including message overhead tokens. Counting individually would miss per-message framing tokens.
- **Modifying the original messages list in-place during compaction:** The runner's `messages` list may be referenced elsewhere. Always return a new list from compaction.
- **Compacting inside the retry loop only:** Compaction should happen before the first attempt too, not just between retries. The message list from thread restoration may already exceed the threshold.
- **Ignoring the system message during truncation:** System messages must never be removed by truncation. The strategy must preserve system messages regardless of age.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Token counting | Custom tokenizer wrapper | `litellm.token_counter(model=..., messages=...)` | Handles 100+ model tokenizers, falls back gracefully, already tested with all message types in codebase [VERIFIED: runtime tests] |
| Message normalization for counting | Custom dict/LC/Pydantic converter | `litellm.token_counter` accepts all three directly | Verified it handles PromptMessage, dict, AIMessage, ToolMessage without conversion [VERIFIED: runtime tests] |
| Model context window lookup | Hardcoded model limits table | `litellm.get_model_info(model)["max_input_tokens"]` for auto-detection (with fallback to explicit config) | LiteLLM maintains model database; but requires explicit `max_context_tokens` as primary config since newer models may not be mapped [VERIFIED: get_model_info fails for some Anthropic models] |

**Key insight:** The only custom code needed is the compaction strategies themselves and the integration glue. Token counting and model info are delegated to litellm.

## Common Pitfalls

### Pitfall 1: Mixed Message Types in Token Counting
**What goes wrong:** The `messages` list in `runner.py` accumulates `PromptMessage` objects (from prompt assembly), `AIMessage` objects (from `_assistant_message_for`), and `ToolMessage` objects (from `build_tool_message`). Passing these to `litellm.token_counter` could fail if the function doesn't handle all types.
**Why it happens:** Multiple message construction paths use different types.
**How to avoid:** Verified that `litellm.token_counter` handles all three types natively. No normalization needed. [VERIFIED: runtime tests with all three types]
**Warning signs:** `TypeError` from litellm during token counting.

### Pitfall 2: Compacting Messages That Are Still Referenced
**What goes wrong:** The `messages` list is passed by reference throughout the runner. If compaction mutates the list in-place, other code paths that reference the original messages (like audit serialization) see corrupted data.
**Why it happens:** Python list aliasing.
**How to avoid:** Compaction strategies must return a new list. The runner replaces its local `messages` variable with the compacted list.
**Warning signs:** Audit records contain `[output omitted]` placeholders when they shouldn't.

### Pitfall 3: Token Count Drift Between Counting and LLM Invocation
**What goes wrong:** Token count is calculated before LLM invocation, but the actual token consumption reported by the provider differs slightly (due to tokenizer version differences or provider-specific overhead).
**Why it happens:** `litellm.token_counter` uses a local tokenizer that may not exactly match the provider's server-side tokenizer.
**How to avoid:** Use the count for threshold detection only (approximate), not as an exact budget. The 0.8 default trigger ratio provides 20% headroom. After LLM invocation, update accumulated count using the provider's reported `token_usage` from `ProviderResponse` when available. [VERIFIED: ProviderResponse.token_usage exists and is populated by LiteLLMProviderAdapter]
**Warning signs:** Context overflow errors from the provider despite compaction.

### Pitfall 4: LLM Summarization Strategy's Own Token Cost
**What goes wrong:** The summarization strategy makes an LLM call to generate a summary. This call itself consumes tokens and costs money. If not accounted for, it can blow the budget or cause infinite compaction loops.
**Why it happens:** The summarization LLM call is outside the normal agent runner flow.
**How to avoid:** (a) Deduct the summarization call's token cost from the available budget before the main agent call. (b) Use the existing `ProviderAdapter` for the summarization call so cost instrumentation captures it. (c) The summarized messages should be shorter than what they replaced -- verify post-compaction token count is actually lower.
**Warning signs:** `BudgetExceededError` triggered by the compaction step itself.

### Pitfall 5: System Message Removal During Truncation
**What goes wrong:** Naive truncation ("drop the first N messages") removes the system message, causing the agent to lose its instruction.
**Why it happens:** The system message is always the first message in the list.
**How to avoid:** All strategies must identify and preserve the system message (role="system") as well as the most recent N messages (per `preserve_recent_messages_count`). Only messages in the middle are eligible for compaction.
**Warning signs:** Agent behavior changes dramatically after compaction -- responds generically instead of following instructions.

### Pitfall 6: Thread State Doesn't Include Compacted Messages
**What goes wrong:** After compaction, the runner saves thread state but doesn't include the compacted message list. On the next run, thread state restoration uses the original (uncompacted) messages, and the context overflows again immediately.
**Why it happens:** Current `_checkpoint_thread_state` saves input/output/audit but not the message list.
**How to avoid:** Store compacted messages in thread state so they're restored in subsequent runs. The compacted messages become the canonical message history for that thread.
**Warning signs:** Every run for a long-lived thread triggers compaction on the very first message.

## Code Examples

Verified patterns from the existing codebase:

### Token Counting with litellm
```python
# Source: Verified via runtime test during research
import litellm

# Works with all message types in the runner's list:
# - dicts: {"role": "system", "content": "..."}
# - PromptMessage: pydantic model with role + content
# - LangChain messages: AIMessage, ToolMessage, etc.
token_count = litellm.token_counter(
    model="gpt-4o",  # Uses model-specific tokenizer
    messages=messages_list,
)
# Returns: int (approximate token count)
# For unknown models: falls back to cl100k_base tokenizer
```

### Model Context Window Size Lookup
```python
# Source: Verified via runtime test during research
import litellm

try:
    info = litellm.get_model_info("gpt-4o")
    max_input = info["max_input_tokens"]  # 128000
except Exception:
    # Fallback: require explicit max_context_tokens config
    max_input = None  # User must configure manually
```

### Protocol Definition Pattern (existing codebase)
```python
# Source: src/zeroth/core/agent_runtime/models.py lines 164-180
class ThreadStateStore(Protocol):
    async def load(self, thread_id: str) -> dict[str, Any] | None: ...
    async def checkpoint(self, thread_id: str, state: dict[str, Any]) -> None: ...
```

### Runner Injection Pattern (existing codebase)
```python
# Source: src/zeroth/core/agent_runtime/runner.py lines 56-84
class AgentRunner:
    def __init__(
        self,
        config: AgentConfig,
        provider: ProviderAdapter,
        *,
        # ... existing params ...
        budget_enforcer: Any | None = None,
        # New: context_tracker follows same pattern
        # context_tracker: ContextWindowTracker | None = None,
    ) -> None:
        # ...
        self.budget_enforcer = budget_enforcer
```

### Per-Node Config on AgentNodeData (existing codebase)
```python
# Source: src/zeroth/core/graph/models.py lines 110-128
class AgentNodeData(BaseModel):
    instruction: str
    model_provider: str
    # ... existing fields ...
    template_ref: TemplateReference | None = None  # Phase 36 added this
    # Phase 37 adds:
    # context_window: ContextWindowSettings | None = None
```

### Orchestrator Injection Pattern (existing codebase)
```python
# Source: src/zeroth/core/orchestrator/runtime.py lines 376-394
# The orchestrator injects services into runner before execution:
original_memory_resolver = getattr(runner, "memory_resolver", _MISSING)
if (
    self.memory_resolver is not None
    and original_memory_resolver is not _MISSING
    and original_memory_resolver is None
):
    runner.memory_resolver = self.memory_resolver
# Phase 37: Same pattern for context_tracker injection
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Fixed context window, no management | Automatic compaction before overflow | Current (this phase) | Prevents context overflow errors for long-running agent threads |
| Naive truncation (drop oldest) | Observation masking as default | Current (this phase) | Preserves conversation structure while removing bulky tool outputs |
| Global summarization | Per-node configurable strategy | Current (this phase) | Agents with different needs (e.g., code agent vs chat agent) use different strategies |

**Deprecated/outdated:**
- Manual context management by users: This phase automates it. Users previously had to manage thread length themselves.

## Assumptions Log

> List all claims tagged `[ASSUMED]` in this research.

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `litellm.token_counter` with `messages=` includes per-message overhead tokens (framing tokens) in the count | Architecture Patterns | LOW -- if it undercounts slightly, the 20% headroom (0.8 trigger ratio) provides buffer; verified count of 23 tokens for 2 short messages suggests framing is included [VERIFIED via test but framing accuracy not formally documented] |
| A2 | The runner's `messages` list at the compaction point contains all accumulated messages including tool call messages from previous turns | Architecture Patterns | MEDIUM -- if thread restoration doesn't populate the messages list with previous turn messages, compaction has nothing to compact; requires examining how thread_state is used in prompt assembly |
| A3 | Compacted messages stored in thread state will be correctly restored and used as the starting messages list on subsequent runs | Common Pitfalls | HIGH -- this is critical for cross-run persistence; current `_checkpoint_thread_state` stores input/output/audit but not messages directly; the compacted messages need a new storage path |

**If this table is empty:** N/A -- 3 assumptions identified above.

## Open Questions

1. **How are previous conversation messages restored from thread state?**
   - What we know: `_load_thread_state` returns a dict, and `PromptAssembler.assemble()` receives `thread_state` as a kwarg. The assembler injects thread state as a JSON block in the user message (line 98-99 of prompt.py).
   - What's unclear: Thread state is currently a flat dict snapshot (last input/output/audit), not a message history. For compaction to persist across runs, the compacted messages need to be stored in and restored from thread state.
   - Recommendation: Plan 37-01 should extend `ThreadStateStore` or the checkpoint schema to include a `compacted_messages` field. Plan 37-02 integration should ensure the runner loads these messages as the starting point for subsequent runs.

2. **Should compaction also trigger during tool call resolution loops?**
   - What we know: `_resolve_tool_calls` can add many messages (assistant + tool result per call). If an agent makes 4 tool calls each returning large outputs, that's 8 new messages potentially adding thousands of tokens.
   - What's unclear: Whether to compact between tool calls or only before the initial LLM invocation.
   - Recommendation: Compact before each `run_provider_with_timeout` call -- both the initial call and the re-calls inside `_resolve_tool_calls`. This prevents overflow mid-conversation.

3. **What is the interaction between compaction and audit serialization?**
   - What we know: `AgentAuditSerializer.serialize_record()` captures the prompt assembly and response. If messages are compacted, the audit record should reflect the compacted state.
   - What's unclear: Whether the audit record should contain original messages, compacted messages, or both.
   - Recommendation: Audit record contains the compacted messages (what was actually sent to the LLM). If `archive_originals=True` (D-08), store originals separately in thread state.

## Project Constraints (from CLAUDE.md)

- **Build & Test:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Package layout:** `src/zeroth/core/` for source, `tests/` for tests
- **Progress tracking:** Must use progress-logger skill during implementation
- **Coding convention:** Pydantic `ConfigDict(extra="forbid")`, Protocol for interfaces, Google-style docstrings
- **Test patterns:** pytest with async support, DeterministicProviderAdapter for mocking LLM calls, `unittest.mock.patch` for litellm functions

## Sources

### Primary (HIGH confidence)
- `litellm.token_counter` -- signature verified via `help()`, behavior verified via runtime tests with dict, LangChain, and Pydantic messages [VERIFIED: runtime session]
- `litellm.get_model_info` -- tested with multiple model strings, confirmed partial coverage of model database [VERIFIED: runtime session]
- `src/zeroth/core/agent_runtime/runner.py` -- full read, identified exact integration points [VERIFIED: codebase read]
- `src/zeroth/core/agent_runtime/models.py` -- ThreadStateStore Protocol, AgentConfig, InMemoryThreadStateStore [VERIFIED: codebase read]
- `src/zeroth/core/agent_runtime/provider.py` -- ProviderResponse.token_usage, ProviderAdapter Protocol [VERIFIED: codebase read]
- `src/zeroth/core/agent_runtime/prompt.py` -- PromptAssembler message construction [VERIFIED: codebase read]
- `src/zeroth/core/graph/models.py` -- AgentNodeData per-node config pattern [VERIFIED: codebase read]
- `src/zeroth/core/agent_runtime/thread_store.py` -- RepositoryThreadStateStore persistence [VERIFIED: codebase read]
- `src/zeroth/core/orchestrator/runtime.py` -- Runner injection pattern, _drive() loop [VERIFIED: codebase read]
- `src/zeroth/core/templates/` -- Recent module structure pattern (Phase 36) [VERIFIED: codebase read]
- `governai.integrations.tool_calls.build_tool_message` -- returns LangChain ToolMessage [VERIFIED: runtime test]
- litellm version 1.83.0 [VERIFIED: importlib.metadata.version]

### Secondary (MEDIUM confidence)
- litellm per-message token framing overhead included in `messages=` counting -- inferred from count of 23 tokens for 2 short messages (raw text would be ~10 tokens) [VERIFIED: runtime test, but framing details undocumented]

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- zero new packages, litellm token_counter verified working with all message types
- Architecture: HIGH -- clear integration points in existing runner.py, well-established Protocol and injection patterns
- Pitfalls: HIGH -- all pitfalls identified from direct codebase analysis and verified runtime behavior

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable -- no fast-moving dependencies, litellm pinned to >=1.83,<2.0)
