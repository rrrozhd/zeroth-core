---
phase: 43-parallel-subgraph-fan-out-merge-strategies
verified: 2026-04-14T21:31:00Z
status: human_needed
score: 4/4 success criteria verified
overrides_applied: 0
re_verification:
  previous_status: none
  previous_score: n/a
human_verification:
  - test: "D-21 Scenario 2 — fan-out inside a subgraph (nested) end-to-end"
    expected: "Child subgraph containing its own ParallelExecutor.split_fan_out runs concurrently, shares parent step_tracker, and fans in correctly"
    why_human: "Executor explicitly deferred this as a follow-up. Production wiring is present and proven at unit level (step_tracker threading in runtime.py lines 433, 504, 662; _execute_parallel_fan_out reuses caller tracker at lines 615-616; SubgraphExecutor.execute forwards step_tracker into _drive at executor.py line 208), but no end-to-end integration test asserts the combined behavior under a shared step budget. Capability is verifiable by code inspection; behavioral guarantee requires a human-authored nested fixture."
  - test: "D-21 Scenario 3 — three-level nesting (fan-out → subgraph → inner fan-out)"
    expected: "All three levels of composition execute, step budget enforced across all levels, audit prefixes correctly nested"
    why_human: "Same as Scenario 2. namespace_subgraph recursive re-namespacing with branch_index is unit-tested (TestNamespaceSubgraphBranchIndex), but full three-level composition requires human-verified integration fixture."
  - test: "D-11 literal resume idempotency under execute/resume counter fixture"
    expected: "On a pause-then-resume cycle, child_run_execute_count == initial_branch_count and child_run_resume_count == 1 for the paused branch; completed siblings' BranchResult dicts are byte-identical pre/post pause"
    why_human: "Production wiring confirmed at runtime.py: _execute_parallel_fan_out_resume rehydrates completed_results via explicit dict-to-BranchResult reconstruction (lines 918-932), calls subgraph_executor.resume() exclusively for the paused branch (with resume_graph fallback for duck-typed test doubles), and never re-invokes .execute() on siblings. A counter-fixture test would lock this invariant formally but requires manual assertion design."
---

# Phase 43: Parallel Subgraph Fan-Out & Merge Strategies Verification Report

**Phase Goal:** Enable fan-out and subgraph composition to work together, and implement the full family of reduce merge strategies declared by the platform.
**Verified:** 2026-04-14T21:31:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (ROADMAP Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | SubgraphNode inside a fan-out branch executes child graph concurrently | VERIFIED | `graph/models.py:107` — `parallel_config: ParallelConfig \| None` on NodeBase, `_reject_parallel_config` grep returns zero matches. `parallel/executor.py:split_fan_out` no longer raises FanOutValidationError for SubgraphNode (only HumanApprovalNode branch remains). `orchestrator/runtime.py:654` — `branch_coro_factory` detects SubgraphNode and dispatches via `self.subgraph_executor.execute(..., branch_context=ctx, step_tracker=step_tracker)`. Integration exercised by `TestScenario1SubgraphInFanOutBranch::test_fan_out_to_subgraph_collect`. |
| 2 | Subgraph can itself contain fan-out without corrupting isolation, audit, or thread semantics | VERIFIED (wiring) / HUMAN_NEEDED (scenario 2/3 e2e) | D-06 isolated thread forced at `subgraph/executor.py:157-164` when `branch_context is not None`. D-08/D-12 shared `step_tracker` threaded: `runtime.py:615-616` (fresh tracker only when None passed), `subgraph/executor.py:208` (forwards into recursive `_drive`). D-10 branch-prefixed audit IDs: `subgraph/resolver.py:103` builds `branch:{branch_index}:subgraph:{graph_ref}:{depth}:` prefix. Production wiring is real; D-21 scenarios 2 & 3 deferred as human-verified integration (not capability gap). |
| 3 | Merge strategies `collect`, `reduce`, `merge`, custom produce deterministic typed outputs during fan-in | VERIFIED | `parallel/reducers.py` — full strategy registry: `_reduce_collect` (list preserving None), `_reduce_merge` (shallow dict.update branch-index order, D-02), `_reduce_fold` (sequential left-to-right, D-01), `_default_fold` (reduce built-in, D-04 literal), `resolve_reducer_ref` (regex-guarded importlib), `dispatch_strategy` entry point. `parallel/models.py:30` — `Literal["collect","reduce","merge","custom"]`. `parallel/executor.py:295` — `collect_fan_in` dispatches through `dispatch_strategy`. 39 tests in `tests/parallel/test_merge_strategies.py`. |
| 4 | Invalid merge-strategy / incompatible reducer-output contracts rejected at graph validation time | VERIFIED | `graph/validation.py:123` — `async validate_or_raise`. `:129` `_validate_parallel_configs` iterates all nodes with parallel_config. `:153` calls `resolve_reducer_ref` for custom (D-16). `:168` calls `_check_merge_dict_contract` for merge strategy, which fetches via injected `ContractRegistry.get` and asserts `json_schema.type == "object"` (D-17). `ValidationCode.INVALID_MERGE_STRATEGY` + `INVALID_REDUCER_REF` added. `graph/repository.py:99-100` — `publish()` calls `await self._validator.validate_or_raise(graph)` before state transition (D-15). 21 tests in `tests/graph/test_merge_strategy_validation.py`. |

**Score:** 4/4 success criteria verified in production code.

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `src/zeroth/core/parallel/reducers.py` | Strategy registry module | VERIFIED | 186 lines. All 5 helpers + dispatch_strategy present. |
| `src/zeroth/core/parallel/errors.py` | BranchApprovalPauseSignal + MergeStrategyError family | VERIFIED | Lines 48, 57, 66, 74. BranchApprovalPauseSignal is BaseException subclass (critical for asyncio.gather fail-fast). |
| `src/zeroth/core/parallel/models.py` | ParallelConfig with reducer_ref + 4-strategy Literal + model_validator | VERIFIED | Lines 30, 36, 50-66. FanInResult.pause_state at line 135. |
| `src/zeroth/core/parallel/executor.py` | split_fan_out unblock + collect_fan_in dispatch | VERIFIED | Imports BranchApprovalPauseSignal, FanInResult, dispatch_strategy. split_fan_out no longer raises for SubgraphNode. Fail-fast and best-effort both catch BranchApprovalPauseSignal and partition completed/cancelled siblings. |
| `src/zeroth/core/subgraph/executor.py` | execute(branch_context, step_tracker) + resume() | VERIFIED | `async def resume` at line 220. D-06 forced isolated thread lines 157-164. D-09 W-4 cost rollup at lines 213-216 (sole writer of total_cost_usd). |
| `src/zeroth/core/subgraph/resolver.py` | namespace_subgraph(branch_index=...) | VERIFIED | branch_index kwarg line 70, prefix build line 103. |
| `src/zeroth/core/orchestrator/runtime.py` | _drive(step_tracker), _execute_parallel_fan_out_resume, _handle_parallel_subgraph_pause, pending_parallel_subgraph loop detection | VERIFIED | Line 223 step_tracker kwarg on _drive. Line 267 pending_parallel_subgraph detection at loop top. Line 807 _handle_parallel_subgraph_pause. Line 865 _execute_parallel_fan_out_resume. Line 615-616 shared tracker reuse (fixes D-12 budget escape). |
| `src/zeroth/core/graph/models.py` | SubgraphNode without _reject_parallel_config | VERIFIED | grep `_reject_parallel_config` returns zero matches. parallel_config field inherited from NodeBase line 107. |
| `src/zeroth/core/graph/validation.py` | async validate_or_raise + parallel_config checks | VERIFIED | Lines 88, 93, 123, 129, 153, 168, 208. Async migration complete; contract_registry injection supported with WARNING degradation. |
| `src/zeroth/core/graph/repository.py` | publish calls validator | VERIFIED | Line 26 validator kwarg, line 99-100 awaits validate_or_raise before publish. |
| `tests/parallel/test_subgraph_in_parallel.py` | D-21 scenarios + pause tests | VERIFIED | 17 tests across 6 classes, including Scenario 1 e2e and approval-pause stash. |
| `tests/parallel/test_merge_strategies.py` | Strategy registry tests | VERIFIED | 39 tests (D-01/02/04/16/22 coverage including regex-before-importlib). |
| `tests/graph/test_merge_strategy_validation.py` | Publish-validation tests | VERIFIED | 21 tests (reducer_ref, merge contract, degraded mode, retroactive rejection). |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `GraphRepository.publish` | `GraphValidator.validate_or_raise` | bootstrap injection | WIRED | `bootstrap.py` constructs `GraphValidator(contract_registry=...)` and passes to `GraphRepository(database, validator=...)`. First production call site. |
| `GraphValidator._validate_parallel_configs` | `resolve_reducer_ref` | import | WIRED | `validation.py:31` imports; `:153` calls for custom strategy. |
| `GraphValidator._check_merge_dict_contract` | `ContractRegistry.get` | constructor injection | WIRED | Line 208 `await self._contract_registry.get(node.output_contract_ref)`. Degrades to WARNING when None. |
| `ParallelExecutor.collect_fan_in` | `dispatch_strategy` | import | WIRED | `executor.py:27` imports; `:295` dispatches with reducer_ref. |
| `RuntimeOrchestrator._drive` | `SubgraphExecutor.execute` | branch dispatch | WIRED | `runtime.py:654` and `:426` pass `step_tracker` and `branch_context`. |
| `branch_coro_factory` | `BranchApprovalPauseSignal` raise | isinstance WAITING_APPROVAL check | WIRED | Raised on child_run.status == WAITING_APPROVAL; caught by fail-fast/best-effort and propagated up through FanInResult.pause_state. |
| `RuntimeOrchestrator._execute_parallel_fan_out_resume` | `SubgraphExecutor.resume` | duck-typed dispatch | WIRED (D-11 literal) | Paused branch resumed exclusively via `subgraph_executor.resume()`; `resume_graph` direct-call fallback only for test-double executors lacking `.resume`. Never calls `.execute()` on siblings. Completed siblings rehydrated byte-identically at lines 918-932. Cancelled siblings recorded `BranchResult(output=None, error="cancelled_by_approval_pause")` per D-19. |
| `RuntimeOrchestrator._drive` loop top | `_execute_parallel_fan_out_resume` | `pending_parallel_subgraph` metadata short-circuit | WIRED | Line 267 detects metadata, line 269 dispatches resume, line 281 clears key. |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|--------------------|--------|
| `collect_fan_in` | `reduced_value` | `dispatch_strategy(strategy, outputs, reducer_ref)` | Yes — strategy registry returns real computed values (39 unit tests assert) | FLOWING |
| `SubgraphExecutor.execute` return metadata | `total_cost_usd` | `_sum_audit_cost(result.execution_history)` at W-4 return path | Yes — single writer; runtime.py `_sum_run_cost` reads it for branch-level rollup | FLOWING |
| `_execute_parallel_fan_out_resume` output | rehydrated + resumed + cancelled `BranchResult` list | explicit dict reconstruction + `subgraph_executor.resume()` + D-19 sentinel | Yes — three partitions assembled in branch-index order, passed through `collect_fan_in` | FLOWING |
| `pending_parallel_subgraph` metadata | D-11 stash dict | `_handle_parallel_subgraph_pause` | Yes — full BranchContext dump, paused_info, completed/cancelled partitions | FLOWING |

### Behavioral Spot-Checks

Smoke test per orchestrator: `uv run pytest tests/parallel tests/subgraph tests/graph tests/orchestrator -q` → **248 passed** (already run upstream). 17 + 39 + 21 = 77 Phase-43-specific tests present on disk.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| ORCH-01 | 43-01 | Fan-out + subgraph composition | SATISFIED | Truth 1, 2; see key-link table rows 5, 7. |
| ORCH-02 | 43-01 | Approval-pause resume in parallel subgraphs (D-11 literal) | SATISFIED | Truth 2; _execute_parallel_fan_out_resume D-11 literal verified by code inspection of runtime.py lines 865-1000. |
| ORCH-03 | 43-02 | Merge strategies (collect/reduce/merge/custom) | SATISFIED | Truth 3; reducers.py + dispatch_strategy + 39 tests. |
| ORCH-04 | 43-02 | Graph-validation-time rejection of invalid strategies | SATISFIED | Truth 4; validation.py + repository.py publish hook + 21 tests. |

No orphaned requirements — REQUIREMENTS.md ORCH-01..04 all claimed by plans.

### Decision Ledger Coverage (D-01 … D-23)

| Decision | Status | Evidence |
|----------|--------|----------|
| D-01 sequential 2-arg fold | VERIFIED | `_reduce_fold` reducers.py:69-94 |
| D-02 merge = shallow dict.update | VERIFIED | `_reduce_merge` reducers.py:48-66 |
| D-04-literal reduce uses built-in default | VERIFIED | `_default_fold` reducers.py:133; dispatch_strategy reduce branch line 172 |
| D-05 SubgraphNode reject removed | VERIFIED | grep `_reject_parallel_config` returns zero |
| D-06 forced isolated thread in fan-out subgraph | VERIFIED | subgraph/executor.py:157-164 |
| D-08/D-12 shared GlobalStepTracker | VERIFIED | runtime.py:223/615-616/662, executor.py:208 |
| D-09/W-4 cost rollup at SubgraphExecutor return | VERIFIED | subgraph/executor.py:213-216 (sole writer) |
| D-10 namespace_subgraph branch_index prefix | VERIFIED | resolver.py:70/103 |
| D-11-literal resume path | VERIFIED | runtime.py:865-1000; subgraph_executor.resume() exclusive, completed siblings rehydrated byte-identically, cancelled siblings get sentinel. Counter-fixture test deferred (human verification item). |
| D-15 publish hook | VERIFIED | repository.py:99-100; bootstrap.py wires validator |
| D-16 regex-guarded importlib | VERIFIED | reducers.py:36-38, `resolve_reducer_ref` line 108 rejects BEFORE import_module |
| D-17 merge requires object contract | VERIFIED | validation.py:`_check_merge_dict_contract` line 170 |
| D-19 cancelled = None + error sentinel | VERIFIED | runtime.py:995-1000 |
| D-22 default merge_strategy = "collect" | VERIFIED | parallel/models.py:30 |
| D-23 FanOutValidationError SubgraphNode branch removed | VERIFIED | parallel/executor.py split_fan_out no longer raises for SubgraphNode |

All 23 locked decisions either explicitly realized or are sub-cases of the above (D-03, D-07, D-13, D-14, D-18, D-20, D-21 are meta/design decisions; D-21 scenarios 2/3 deferred as follow-ups — see human verification).

### Anti-Patterns Found

None. No TODO/FIXME/placeholder/stub markers in modified files. All dataclass vs pydantic deviations were auto-fixed with explicit dict round-trip (schema-identical on wire).

### Deferred Items (Step 9b Filter)

All three human-verification items above are test-coverage gaps, not capability gaps. Production wiring is present and provable by code inspection. They are NOT filtered to later milestone phases because no later phase addresses them — they are acknowledged follow-ups within Phase 43's own scope.

### Gaps Summary

**No capability gaps.** Phase 43 meets all four ROADMAP success criteria in production code:

1. Fan-out + SubgraphNode composition — the validator that blocked it is gone, split_fan_out dispatches subgraph branches, branch_coro_factory invokes SubgraphExecutor.execute with branch_context + step_tracker, branch-prefixed audit IDs flow through namespace_subgraph.
2. Nested composition (fan-out inside subgraph) — step_tracker is shared across levels (verified by the `if step_tracker is None` guard at runtime.py:615 and forwarding at executor.py:208), isolated thread forced for child subgraphs.
3. Full strategy family (collect/reduce/merge/custom) — reducers.py provides a complete dispatch registry, collect_fan_in routes through it, ParallelConfig model-validates reducer_ref consistency per D-04 literal.
4. Publish-time validation — GraphValidator.validate_or_raise is now wired into GraphRepository.publish via bootstrap, with _validate_parallel_configs exercising resolve_reducer_ref + _check_merge_dict_contract.

**Deferred follow-ups (test-only, human-verifiable):**
- D-21 scenarios 2 & 3 full end-to-end integration tests (nested fan-out and three-level nesting)
- D-11 literal resume idempotency test with execute/resume counter fixtures
- W-1 subgraph-exception-in-branch end-to-end assertions for fail_fast + best_effort through collect_fan_in

These are acknowledged by the executor in 43-01-SUMMARY "Open Follow-ups" and do not hide any missing capability — the production wiring is in place and independently verified above.

### Known Executor Deviations (Context Verification)

1. **D-21 Scenarios 2 & 3 deferred** — verified NOT a capability gap. step_tracker threading present at runtime.py:223/615-616/662 and subgraph/executor.py:208. Shared-budget fix at `_execute_parallel_fan_out` confirmed (`if step_tracker is None: step_tracker = GlobalStepTracker(...)`). SubgraphExecutor.execute accepts branch_context kwarg. Capability is real; tests are missing. Routed to human verification.
2. **D-11 counter-based idempotency test deferred** — verified NOT a capability gap. `_execute_parallel_fan_out_resume` at runtime.py:865-1000 rehydrates completed siblings byte-identically (lines 918-932) and dispatches the paused branch exclusively through `subgraph_executor.resume()` (never `.execute()`). Fallback to `resume_graph` only when the injected executor lacks `.resume` (test-double duck typing). Routed to human verification for counter-fixture assertion.
3. **BranchResult/BranchContext dataclass vs pydantic** — verified: `_handle_parallel_subgraph_pause` serializes via explicit dict builder, `_execute_parallel_fan_out_resume` reconstructs via dataclass constructor + `RunHistoryEntry.model_validate` for history entries. On-wire dict shape is schema-identical. No regression.

---

_Verified: 2026-04-14T21:31:00Z_
_Verifier: Claude (gsd-verifier)_
