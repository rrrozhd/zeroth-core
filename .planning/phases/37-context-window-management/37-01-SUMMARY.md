---
phase: 37-context-window-management
plan: 01
subsystem: context-window
tags: [context-window, token-tracking, compaction, strategies, tdd]
dependency_graph:
  requires: []
  provides: [context_window_package, compaction_strategies, token_tracker]
  affects: [agent_runtime, orchestrator]
tech_stack:
  added: [litellm.token_counter]
  patterns: [Protocol-based strategy, Pydantic ConfigDict extra=forbid, TDD red-green]
key_files:
  created:
    - src/zeroth/core/context_window/__init__.py
    - src/zeroth/core/context_window/models.py
    - src/zeroth/core/context_window/errors.py
    - src/zeroth/core/context_window/tracker.py
    - src/zeroth/core/context_window/strategies.py
    - tests/context_window/__init__.py
    - tests/context_window/test_models.py
    - tests/context_window/test_tracker.py
    - tests/context_window/test_strategies.py
  modified: []
decisions:
  - "ProviderAdapter.ainvoke used (not invoke) -- matches existing codebase Protocol signature"
  - "Tracker stores last_compaction_strategy name for state reporting"
  - "_split_messages shared helper used by all three strategies for consistent system/middle/recent splitting"
metrics:
  duration: 5m
  completed: "2026-04-13"
  tasks_completed: 2
  tasks_total: 2
  test_count: 49
  files_created: 9
---

# Phase 37 Plan 01: Context Window Models, Tracker & Strategies Summary

Context window management package with litellm-based token tracking, ratio-based compaction triggering, and three Protocol-implementing compaction strategies (truncation, observation masking, LLM summarization) -- all via TDD.

## Task Results

| Task | Name | Commit(s) | Key Files |
|------|------|-----------|-----------|
| 1 | Context window models, errors, and tracker | 51b53ed (RED), 29a2536 (GREEN) | models.py, errors.py, tracker.py, __init__.py |
| 2 | Three built-in compaction strategies | f249a27 (RED), 381b216 (GREEN) | strategies.py, __init__.py updated |

## What Was Built

### Models (models.py)
- **ContextWindowSettings**: 5 configurable fields -- max_context_tokens (128K default), summary_trigger_ratio (0.8), compaction_strategy ("observation_masking"), preserve_recent_messages_count (4), archive_originals (False). All with ConfigDict(extra="forbid") and Pydantic Field validators.
- **CompactionResult**: Captures compacted messages, counts, token metrics, strategy name, and optional archived originals.
- **CompactionState**: Tracker state snapshot with accumulated tokens, max, compaction count, last strategy.

### Errors (errors.py)
- **ContextWindowError** base exception
- **CompactionError** for strategy failures
- **TokenCountError** for litellm counting failures

### Tracker (tracker.py)
- **ContextWindowTracker**: Wraps litellm.token_counter with model-specific counting, threshold detection via ratio comparison, and async maybe_compact() that delegates to the configured CompactionStrategy. Normalizes Pydantic models to dicts for litellm compatibility.

### Strategies (strategies.py)
- **CompactionStrategy Protocol**: @runtime_checkable, async compact() method returning CompactionResult.
- **TruncationStrategy**: Drops middle messages, preserves system message at index 0 and last N messages.
- **ObservationMaskingStrategy**: Replaces tool output content with "[output omitted -- N tokens]" placeholders. Default strategy per requirements.
- **LLMSummarizationStrategy**: Condenses older messages into a single summary via ProviderAdapter.ainvoke() call. Wraps provider errors in CompactionError.
- All strategies use shared _split_messages() helper and return NEW lists (never mutate originals per T-37-01).

## Verification Results

- 49 tests passing (19 model/error + 12 tracker + 18 strategy)
- ruff check: 0 errors
- ruff format: all files formatted
- Import verification: all public symbols importable from zeroth.core.context_window

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] ProviderAdapter uses ainvoke not invoke**
- **Found during:** Task 2 implementation
- **Issue:** Plan referenced `provider.invoke()` but the actual ProviderAdapter Protocol in the codebase uses `ainvoke()`
- **Fix:** Used `self._provider.ainvoke(request)` to match codebase convention
- **Files modified:** src/zeroth/core/context_window/strategies.py

**2. [Rule 1 - Bug] Ruff B009 lint violation**
- **Found during:** Task 2 verification
- **Issue:** `getattr(msg, "tool_call_id")` with constant attribute flagged by ruff B009
- **Fix:** Changed to `msg.tool_call_id` direct attribute access
- **Files modified:** src/zeroth/core/context_window/strategies.py

## Threat Mitigations Applied

| Threat | Mitigation | Verified |
|--------|-----------|----------|
| T-37-01 (Tampering) | All strategies return new lists, never mutate originals | 5 test assertions on list identity |
| T-37-02 (Info Disclosure) | ObservationMaskingStrategy placeholder shows only token count, no content | Test verifies placeholder format |
| T-37-03 (DoS) | TokenCountError wraps litellm exceptions | Test verifies wrapping |
| T-37-04 (Privilege) | LLMSummarizationStrategy uses same ProviderAdapter as agent | Architecture enforced by constructor |

## Known Stubs

None -- all code is fully implemented with no placeholders or TODOs.

## Self-Check: PASSED
