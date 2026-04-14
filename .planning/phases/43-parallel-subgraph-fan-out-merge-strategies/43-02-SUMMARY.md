---
phase: 43-parallel-subgraph-fan-out-merge-strategies
plan: 02
subsystem: orchestration
tags: [orchestration, parallel, reducers, validation, security]
requirements: [ORCH-03, ORCH-04]
requirements_addressed: [ORCH-03, ORCH-04]
dependency_graph:
  requires:
    - src/zeroth/core/parallel/models.py
    - src/zeroth/core/parallel/executor.py
    - src/zeroth/core/graph/validation.py
    - src/zeroth/core/graph/repository.py
    - src/zeroth/core/service/bootstrap.py
    - src/zeroth/core/contracts/registry.py
  provides:
    - src/zeroth/core/parallel/reducers.py
    - GraphValidator publish-time parallel config validation
    - GraphRepository.publish validate_or_raise hook
  affects:
    - tests/graph/test_validation.py (async migration)
tech_stack:
  added:
    - importlib regex-guarded dotted-path resolver
  patterns:
    - strategy dispatch registry
    - pydantic model validator
    - publish-time validation gate
key_files:
  created:
    - src/zeroth/core/parallel/reducers.py
    - tests/_fixtures/__init__.py
    - tests/_fixtures/reducers.py
    - tests/parallel/test_merge_strategies.py
    - tests/graph/test_merge_strategy_validation.py
  modified:
    - src/zeroth/core/parallel/errors.py
    - src/zeroth/core/parallel/models.py
    - src/zeroth/core/parallel/executor.py
    - src/zeroth/core/graph/validation.py
    - src/zeroth/core/graph/validation_errors.py
    - src/zeroth/core/graph/repository.py
    - src/zeroth/core/service/bootstrap.py
    - tests/graph/test_validation.py
decisions:
  - id: D-04-literal
    summary: "reduce uses built-in _default_fold (last-wins); only custom requires reducer_ref"
  - id: D-02
    summary: "merge strategy is shallow dict.update in branch-index order"
  - id: D-01
    summary: "reduce/custom are sequential left-to-right folds with 2-arg reducer"
  - id: D-15
    summary: "GraphValidator.validate_or_raise wired into GraphRepository.publish"
  - id: D-16
    summary: "reducer_ref validated at publish via regex + importlib + callable check"
  - id: D-17
    summary: "merge strategy requires node output contract with type=object"
metrics:
  duration_minutes: ~40
  tasks_completed: 3
  tests_added: 60
  files_created: 5
  files_modified: 8
---

# Phase 43 Plan 02: Merge Strategies & Publish-Time Validation Summary

One-liner: Implemented the full `collect`/`reduce`/`merge`/`custom` fan-in
strategy registry with regex-guarded `importlib` reducer resolution, and
activated `GraphValidator.validate_or_raise()` at graph publish time — this
is the first production call site for the validator.

## What Was Built

### Task 1 — Merge strategy registry (commit `8a154d5`)

- `ParallelConfig.merge_strategy` literal expanded to
  `Literal["collect", "reduce", "merge", "custom"]` with default `"collect"`
  (D-22 backward compat).
- New `reducer_ref: str | None = None` field with pydantic `@model_validator`
  enforcing D-04 literal: only `custom` requires `reducer_ref`; `reduce` uses
  a built-in default fold (`_default_fold`, last-wins); all other strategies
  reject `reducer_ref`.
- New module `src/zeroth/core/parallel/reducers.py`:
  - `_reduce_collect` — returns outputs list unchanged (None preserved, D-19)
  - `_reduce_merge` — shallow `dict.update` in branch-index order, skipping
    None, raising `MergeStrategyError` on non-dict branch output (D-02)
  - `_reduce_fold` — sequential left-to-right fold with a 2-arg reducer,
    skipping None, handling single-element case, wrapping reducer exceptions
    in `MergeStrategyError` (D-01)
  - `resolve_reducer_ref` — regex-guarded (`_REDUCER_REF_PATTERN`) `importlib`
    resolution; rejects bad strings BEFORE any `import_module` call (D-16,
    threat T-43-02-01)
  - `_default_fold` — named last-wins reducer for `reduce` strategy
  - `_STRATEGY_REGISTRY` — dispatch map for `collect`/`merge`
  - `dispatch_strategy` — unified entry point
- Typed errors in `parallel/errors.py`: `MergeStrategyError`,
  `MergeStrategyValidationError`, `ReducerRefValidationError`
- `ParallelExecutor.collect_fan_in` refactored to dispatch through
  `dispatch_strategy` and write the reduced value at `split_path`
- `tests/_fixtures/reducers.py` with importable reducer fixtures
  (`sum_ints`, `sum_scores`, `NOT_CALLABLE`)
- 38 new tests in `tests/parallel/test_merge_strategies.py` covering model
  validation, reducers, regex guard, import resolution, and
  `collect_fan_in` dispatch integration

### Task 2 — Publish-time validation (commit `f0cf9f7`)

- `GraphValidator.__init__` now accepts `contract_registry: ContractRegistry | None`
- `validate` and `validate_or_raise` converted to `async` (only production
  callers are tests — 6 callsites in `tests/graph/test_validation.py`
  updated to `async`/`await`; no non-test production callers existed)
- New `_validate_parallel_configs` iterates nodes with `parallel_config`:
  - For `custom` strategies, calls `resolve_reducer_ref` to catch bad paths
  - For `merge` strategies, calls `_check_merge_dict_contract`
- `_check_merge_dict_contract` fetches the node's output contract via the
  injected `ContractRegistry.get` and asserts
  `json_schema.get("type") == "object"` (D-17). Degrades to a WARNING
  (not ERROR) when no `ContractRegistry` is wired, so tests and
  construction paths without a registry continue to work.
- New `ValidationCode.INVALID_MERGE_STRATEGY` and
  `ValidationCode.INVALID_REDUCER_REF`
- 15 new tests in `tests/graph/test_merge_strategy_validation.py` covering
  backward compat, reducer_ref resolution success/failure, array-contract
  rejection, regex-before-importlib mocking, degraded mode

### Task 3 — Publish hook wiring (commit `b39bec6`)

- `GraphRepository.__init__` accepts `validator: GraphValidator | None`
- `GraphRepository.publish` calls `await self._validator.validate_or_raise(graph)`
  before `graph.publish()` and `self.save()`. Failure keeps the graph in
  DRAFT with persisted state unchanged.
- `bootstrap_service` constructs
  `GraphValidator(contract_registry=contract_registry)` and threads it into
  `GraphRepository(database, validator=...)`. **This is the first production
  call site for `GraphValidator.validate_or_raise()`.**
- 7 new publish-path integration tests (SQLite backend): valid publish
  succeeds, invalid reducer_ref stays DRAFT, merge-array-contract stays
  DRAFT, legacy no-validator still works, validator called exactly once,
  no-parallel-config backward compat, retroactive-rejection documentation

## Decisions Made

### D-04 literal (decision ledger note)

`reduce` uses a built-in default fold (`_default_fold`, last-wins) and does
NOT require `reducer_ref`. Only `custom` requires `reducer_ref`. This is
D-04 interpreted **literally** — a prior plan revision (pre-B-2) had
collapsed `reduce` into `custom` by requiring `reducer_ref` for both; that
was rejected as contradicting D-04 literal text. The two strategies remain
semantically distinct: `reduce` = "platform default fold"; `custom` =
"user-supplied dotted-path reducer".

Concrete consequence: `ParallelConfig(merge_strategy="reduce")` with
`reducer_ref=None` constructs and runs. `reduce` with a non-None
`reducer_ref` is REJECTED by the model validator — authors who want a
custom reducer must pick `merge_strategy="custom"`.

### D-02

`merge` is shallow `dict.update` in branch-index order. Later branches
overwrite earlier keys. None (failed) branches are skipped per D-19. No
deep merge.

### D-15, D-16, D-17

- D-15: `GraphValidator.validate_or_raise` hooked into
  `GraphRepository.publish` BEFORE `graph.publish()` state transition. Net
  new production wiring.
- D-16: `reducer_ref` validation does full `importlib.import_module` +
  `hasattr` + `callable` check at publish time.
- D-17: `merge` strategy requires node output contract to have top-level
  `json_schema.type == "object"` via injected `ContractRegistry`.

## Deviations from Plan

### Auto-fixed items

**1. [Rule 3 - Blocking] ContractRegistry.get uses `.get(name)` not `.get_contract_version(ref)`**

- **Found during:** Task 2 implementation
- **Issue:** Plan used placeholder method name `get_contract_version`; actual
  method on `ContractRegistry` is `get(name: str | ContractReference, version=None)`
- **Fix:** Call `await self._contract_registry.get(node.output_contract_ref)`
- **Files:** `src/zeroth/core/graph/validation.py`

**2. [Rule 3 - Blocking] publish-path tests used `build_graph()` which fails
new validation**

- **Found during:** Task 3 test run
- **Issue:** `tests/graph/test_models.build_graph()` produces a Graph with
  empty `output_contract_ref` fields that pass the old no-op publish path
  but fail `GraphValidator.validate`. This is exactly the retroactive-rejection
  behavior the plan calls out as desirable, but it broke the "valid publish"
  baseline test.
- **Fix:** Publish-path tests now use `build_valid_graph()` from
  `tests/graph/test_validation.py` which passes all checks.
- **Files:** `tests/graph/test_merge_strategy_validation.py`

### None - other

Plan executed as written. Decisions D-01/02/04/15/16/17/22 realized exactly.

## Authentication Gates

None. Fully automated execution.

## Retroactive Rejection Note

Per research Risks row 1: pre-existing DRAFT graphs with invalid
`parallel_config` (e.g. `custom` with a bad reducer_ref, `merge` on an
array contract) will now fail `publish()`. This is desirable — catches
bugs that were previously hidden. No automated migration performed; graphs
must be hand-fixed by authors. Documented in
`test_retroactive_rejection_of_previously_unvalidated_draft`.

Existing PUBLISHED graphs are unaffected (they do not re-publish).

## Security Trust Assumption

`reducer_ref` `importlib` surface is trusted because graph publish is an
authenticated write operation — graph authors already have equivalent
power via `ExecutableUnitNode.manifest_ref`. Regex `_REDUCER_REF_PATTERN`
rejects paths with spaces, colons, relative imports, or single-segment
module names BEFORE any `import_module` call (defense in depth, threat
T-43-02-01). The `test_regex_rejects_before_importlib` test patches
`importlib.import_module` and asserts it is never called for bad paths.

## Verification Results

| Check | Result |
|-------|--------|
| `uv run pytest tests/parallel/test_merge_strategies.py` | 38 passed |
| `uv run pytest tests/parallel/` | 56 passed (including 18 Phase 38 regressions) |
| `uv run pytest tests/graph/test_merge_strategy_validation.py` | 22 passed |
| `uv run pytest tests/graph/` | all graph tests passed |
| `uv run pytest tests/` (full, excluding pre-existing env failures) | 1218 passed |
| `uv run ruff check src/` | clean |

**Pre-existing environmental failures (unrelated to Phase 43):**
`tests/memory/test_{chroma,elastic,pgvector}.py`, `tests/test_async_database.py`,
`tests/dispatch/test_arq_wakeup.py`, `tests/test_postgres_backend.py`,
`tests/test_docs_phase30.py::test_readme_links_to_live_docs`. All are
missing optional deps (psycopg, arq) or broken expectations on project
metadata (README link check) that pre-date this plan.

## Commits

- `8a154d5` — feat(43-02): merge strategy registry with custom reducers
- `f0cf9f7` — feat(43-02): publish-time parallel config validation
- `b39bec6` — feat(43-02): wire GraphValidator into GraphRepository.publish()

## Open Follow-ups

None. All three tasks complete, ORCH-03 and ORCH-04 fully satisfied.

## Self-Check: PASSED

- All files listed as created exist on disk.
- All three commits (`8a154d5`, `f0cf9f7`, `b39bec6`) present in
  `git log --oneline`.
- 60 new tests passing (38 merge-strategies + 22 publish validation).
- `GraphValidator.validate_or_raise` is now called from
  `GraphRepository.publish` (verified by
  `test_publish_calls_validator_exactly_once`).
