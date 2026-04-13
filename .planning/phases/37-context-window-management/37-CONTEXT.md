# Phase 37: Context Window Management - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Add token tracking and configurable compaction strategies to agent threads so that context overflow is prevented automatically before each LLM invocation. Three built-in strategies: truncation, observation masking, and LLM-based summarization. Compaction results persist in thread memory across runs.

</domain>

<decisions>
## Implementation Decisions

### Token Tracking
- **D-01:** Token counting uses `litellm.token_counter` (already in dependencies via the LLM provider layer). Called after each LLM invocation to update the thread's accumulated token count.
- **D-02:** Token count is stored per-thread in `ThreadState` (or equivalent thread memory model). Updated after each agent turn (user message + assistant response).
- **D-03:** Per-agent-node configurable settings: `max_context_tokens` (int), `summary_trigger_ratio` (float, default 0.8), `compaction_strategy` (str, default "observation_masking"), `preserve_recent_messages_count` (int, default 4).

### Compaction Strategies
- **D-04:** Three built-in strategies, all implementing a `CompactionStrategy` Protocol:
  - `truncation` — Drop oldest messages (preserving system message and recent N messages)
  - `observation_masking` — Replace tool/observation outputs in older messages with `[output omitted — N tokens]` placeholders (default strategy per REQUIREMENTS.md)
  - `llm_summarization` — Condense older messages into a summary message via LLM call (opt-in only per REQUIREMENTS.md)
- **D-05:** Strategy is pluggable per agent node — the Protocol allows custom strategies to be registered.
- **D-06:** Compaction triggers when `accumulated_tokens / max_context_tokens >= summary_trigger_ratio` — checked before each LLM invocation.

### Persistence
- **D-07:** Compaction results (compacted messages, summary if applicable) are stored in thread memory via the existing `ThreadStateStore` so they persist across runs.
- **D-08:** Original messages can optionally be archived for audit retrieval — controlled by a `archive_originals` boolean on the compaction settings (default: false).

### Integration
- **D-09:** Compaction runs in the agent runtime loop, after prompt assembly but before LLM invocation. The `AgentRunner` (or equivalent) checks token count and applies compaction if needed.
- **D-10:** New `zeroth.core.context_window` package with models.py, strategies.py, tracker.py, errors.py, __init__.py.
- **D-11:** `ContextWindowSettings` added to agent node config (not global ZerothSettings) — per-node configuration.

### Claude's Discretion
- Whether to add a `CompactionResult` model for tracking what was compacted
- How to handle the LLM summarization strategy's own token cost (deduct from budget)
- Test approach for litellm.token_counter (mock vs real)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Agent Runtime
- `src/zeroth/core/agent_runtime/runner.py` — AgentRunner loop — where compaction integrates
- `src/zeroth/core/agent_runtime/prompt.py` — PromptAssembler — message list construction
- `src/zeroth/core/agent_runtime/models.py` — ThreadStateStore Protocol, AgentConfig
- `src/zeroth/core/agent_runtime/provider.py` — ProviderAdapter, ProviderRequest — LLM invocation

### Thread State
- `src/zeroth/core/agent_runtime/thread_store.py` — ThreadState persistence (where compacted messages are stored)

### LLM Token Counting
- litellm.token_counter — Token counting API (already in dependency tree)

### Graph Models
- `src/zeroth/core/graph/models.py` — AgentNodeData — where per-node config goes

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `litellm.token_counter` — Already available for token counting
- `ThreadStateStore(Protocol)` — Thread state persistence, compaction results store here
- `ProviderAdapter` — LLM invocation interface (used by summarization strategy)
- `PromptAssembler` — Message construction (messages to count/compact)

### Established Patterns
- Protocol for pluggable interfaces
- Per-node configuration on AgentNodeData
- Pydantic ConfigDict(extra="forbid")
- Package structure: models.py, errors.py, __init__.py

### Integration Points
- `agent_runtime/runner.py` — Compaction check before LLM invocation
- `agent_runtime/models.py` — ContextWindowSettings on agent config
- `graph/models.py` — Per-node context window settings
- `agent_runtime/thread_store.py` — Persist compacted messages

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches.

</specifics>

<deferred>
## Deferred Ideas

- Automatic context summarization as default — explicitly out of scope per REQUIREMENTS.md (observation masking outperforms, summarization is opt-in only)

</deferred>

---

*Phase: 37-context-window-management*
*Context gathered: 2026-04-13*
