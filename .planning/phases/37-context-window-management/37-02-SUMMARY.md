---
phase: 37-context-window-management
plan: 02
subsystem: agent-runtime, orchestrator, graph-models
tags: [context-window, compaction, integration, thread-persistence]
dependency_graph:
  requires: [37-01]
  provides: [context-window-runtime-integration]
  affects: [agent_runtime/runner, graph/models, orchestrator/runtime, service/bootstrap]
tech_stack:
  added: []
  patterns: [save-inject-restore, optional-tracker-injection, thread-state-persistence]
key_files:
  created:
    - tests/context_window/test_runner_integration.py
    - tests/context_window/test_orchestrator_integration.py
  modified:
    - src/zeroth/core/agent_runtime/runner.py
    - src/zeroth/core/graph/models.py
    - src/zeroth/core/orchestrator/runtime.py
    - src/zeroth/core/service/bootstrap.py
decisions:
  - "context_tracker parameter uses Any type to avoid hard import dependency (follows budget_enforcer pattern)"
  - "Strategy selection uses hardcoded switch in orchestrator (no code injection risk per T-37-10)"
  - "Bootstrap needs no explicit field -- orchestrator.context_window_enabled defaults True"
  - "Context window audit captured before finally-block restoration so state is accurate"
metrics:
  duration: 6m23s
  completed: 2026-04-13T00:36:40Z
  tasks_completed: 2
  tasks_total: 2
  tests_added: 19
  tests_total_passing: 84
  files_changed: 7
---

# Phase 37 Plan 02: Runtime Integration for Context Window Management Summary

Context window compaction wired into AgentRunner, RuntimeOrchestrator, AgentNodeData, and ServiceBootstrap with 19 integration tests covering all injection, persistence, and audit paths.

## Completed Tasks

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | AgentRunner integration with compaction and thread persistence | b2f3613 | runner.py, test_runner_integration.py |
| 2 | Graph models, orchestrator injection, and bootstrap wiring | c85bb72 | models.py, runtime.py, bootstrap.py, test_orchestrator_integration.py |

## Task Details

### Task 1: AgentRunner Integration

- Added `context_tracker: Any | None = None` parameter to `AgentRunner.__init__`
- Compaction via `maybe_compact()` runs before first LLM invocation
- Compaction runs between tool call re-invocations in `_resolve_tool_calls`
- Compacted messages restored from thread state on subsequent runs (`compacted_messages` key)
- Archived messages stored in thread state when `archive_originals=True`
- Compaction metadata (strategy, tokens before/after, message counts) added to audit record
- 9 integration tests: backward compat, compact-before-LLM, audit metadata, thread persistence, archived messages, cross-run restoration, no-compaction paths, tool-call compaction

### Task 2: Graph Models, Orchestrator, Bootstrap

- Added `context_window: ContextWindowSettings | None = None` field to `AgentNodeData`
- Added `context_window_enabled: bool = True` field to `RuntimeOrchestrator`
- Orchestrator creates `ContextWindowTracker` from node config at dispatch time
- Strategy selection: "truncation" -> TruncationStrategy, "observation_masking" -> ObservationMaskingStrategy, "llm_summarization" -> LLMSummarizationStrategy (uses runner's provider)
- Save/inject/restore pattern for `context_tracker` matches memory_resolver and budget_enforcer
- Context window state (accumulated_tokens, compaction_count) captured in audit `execution_metadata`
- Bootstrap documents context window enabled by default (per-node config, no global wiring needed)
- 10 integration tests: strategy injection (3 strategies), no-tracker cases (None, disabled), restore-after-dispatch, audit presence/absence, AgentNodeData field tests

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None -- all integration points are fully wired.

## Verification Results

- `uv run pytest tests/context_window/ -x -v` -- 68 tests pass (models + tracker + strategies + runner integration + orchestrator integration)
- `uv run pytest tests/agent_runtime/ -x -v` -- 16 tests pass (backward compatible)
- `uv run ruff check` on all modified files -- lint clean
- `python -c "...AgentNodeData(context_window=ContextWindowSettings())..."` prints 128000

## Self-Check: PASSED

All 7 files verified present. Both commit hashes (b2f3613, c85bb72) found in git log.
