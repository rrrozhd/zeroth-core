# Phase 43: Parallel Subgraph Fan-Out & Merge Strategies - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-14
**Phase:** 43-parallel-subgraph-fan-out-merge-strategies
**Areas discussed:** Merge strategies, SubgraphNode in parallel, Nested composition, Registration validation, Error propagation, Testing strategy, Backward compatibility

---

## Merge Strategies

| Option | Description | Selected |
|--------|-------------|----------|
| Sequential fold | Left-to-right fold with user-provided reducer, referenced by dotted path | ✓ |
| Pairwise tree reduce | Balanced binary tree reduction | |
| You decide | Claude picks | |

**User's choice:** Sequential fold

| Option | Description | Selected |
|--------|-------------|----------|
| Shallow dict merge | `dict.update()` in branch order, later branches overwrite | ✓ |
| Deep recursive merge | Nested dicts merged recursively, lists concatenated | |
| You decide | Claude picks | |

**User's choice:** Shallow dict merge

| Option | Description | Selected |
|--------|-------------|----------|
| Dotted import path | `reducer_ref` field, resolved via `importlib`, validated at registration | ✓ |
| Registry with named reducers | `ReducerRegistry` like `ContractRegistry` | |
| You decide | Claude picks | |

**User's choice:** Dotted import path

---

## SubgraphNode in Parallel

| Option | Description | Selected |
|--------|-------------|----------|
| Force isolated per branch | Each branch's subgraph gets its own thread regardless of config | ✓ |
| Honor the config | Respect `SubgraphNodeData.thread_participation` as-is | |
| You decide | Claude picks | |

**User's choice:** Force isolated per branch

| Option | Description | Selected |
|--------|-------------|----------|
| Roll up to parent branch | Child subgraph cost summed into `BranchResult.cost_usd` | ✓ |
| Separate child run tracking | Costs tracked via `parent_run_id` linkage | |
| You decide | Claude picks | |

**User's choice:** Roll up to parent branch

| Option | Description | Selected |
|--------|-------------|----------|
| Pause all branches | Parent run goes `WAITING_APPROVAL`, other branches cancelled/paused | ✓ |
| Continue other branches | Only the blocked branch pauses; parent waits for mixed state | |
| Reject approval in parallel subgraphs | Validation error if subgraph contains `HumanApprovalNode` | |

**User's choice:** Pause all branches

| Option | Description | Selected |
|--------|-------------|----------|
| Branch-prefixed node IDs | `branch:{i}:subgraph:{ref}:{depth}:` audit prefix | ✓ |
| Metadata tagging only | `branch_index` in audit metadata, keep standard IDs | |
| You decide | Claude picks | |

**User's choice:** Branch-prefixed node IDs

---

## Nested Composition

| Option | Description | Selected |
|--------|-------------|----------|
| Shared parent budget | All branches decrement the same `GlobalStepTracker` from parent | ✓ |
| Per-subgraph budget | Each subgraph invocation gets its own budget | |
| You decide | Claude picks | |

**User's choice:** Shared parent budget

| Option | Description | Selected |
|--------|-------------|----------|
| Reuse existing `max_depth` | `SubgraphNodeData.max_depth` already bounds recursion; fan-out is lateral | ✓ |
| Add fan-out nesting limit | New `max_parallel_depth` config | |
| You decide | Claude picks | |

**User's choice:** Reuse existing max_depth

| Option | Description | Selected |
|--------|-------------|----------|
| Standard branch isolation | Subgraph's internal fan-out uses normal `BranchContext`s | ✓ |
| Nested branch namespacing | Compound `outer_branch:inner_branch` IDs and audit prefixes | |

**User's choice:** Standard branch isolation

---

## Registration Validation

| Option | Description | Selected |
|--------|-------------|----------|
| At graph publish | Validate on `DRAFT → PUBLISHED` transition | ✓ |
| At deployment creation | Validate when `DeploymentService` creates a snapshot | |
| Both | Validate at publish AND deployment | |

**User's choice:** At graph publish

| Option | Description | Selected |
|--------|-------------|----------|
| Import + callable check | `importlib.import_module` and verify target is callable | ✓ |
| Syntax-only check | Validate dotted path is well-formed, no import attempt | |
| You decide | Claude picks | |

**User's choice:** Import + callable check

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, type check | `merge` requires dict-like output contract; `reduce`/`custom` check reducer resolves | ✓ |
| No, runtime only | Skip contract compatibility checks | |
| You decide | Claude picks | |

**User's choice:** Yes, type check

---

## Error Propagation

| Option | Description | Selected |
|--------|-------------|----------|
| `SubgraphExecutionError` becomes branch error | Existing wrapper, fail_fast/best_effort decide cancellation | ✓ |
| Unwrap child failure reason | Propagate child `RunFailureState.reason` directly | |
| You decide | Claude picks | |

**User's choice:** SubgraphExecutionError becomes branch error

| Option | Description | Selected |
|--------|-------------|----------|
| No partial output | Failed branch output is None | ✓ |
| Capture last checkpoint | Failed branch gets last checkpoint output | |

**User's choice:** No partial output

---

## Testing Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| In-memory `SubgraphResolver` stub | Test resolver with pre-built `Graph` dict registry, extends Phase 39 pattern | ✓ |
| Full integration with SQLite | Use real `SQLiteDatabase`, `GraphRepository`, `DeploymentService` | |
| You decide | Claude picks | |

**User's choice:** In-memory SubgraphResolver stub

| Option | Description | Selected |
|--------|-------------|----------|
| SubgraphNode in fan-out branch | Basic: fan-out splits, each branch runs subgraph | ✓ |
| Fan-out inside subgraph | Child workflow has its own fan-out node | ✓ |
| Nested: fan-out > subgraph > fan-out | Two levels deep | ✓ |
| Approval pause in parallel subgraph | Subgraph approval gate inside branch, pause + resume works | ✓ |

**User's choice:** All four scenarios selected

---

## Backward Compatibility

| Option | Description | Selected |
|--------|-------------|----------|
| Default unchanged, new strategies opt-in | `merge_strategy` default stays `"collect"`, new strategies explicit | ✓ |
| You decide | Claude picks | |

**User's choice:** Default unchanged, new strategies opt-in

| Option | Description | Selected |
|--------|-------------|----------|
| Remove block entirely | Delete `FanOutValidationError` for `SubgraphNode` | ✓ |
| Opt-in flag on `ParallelConfig` | `allow_subgraph_branches: bool = False` | |
| You decide | Claude picks | |

**User's choice:** Remove block entirely

---

## Claude's Discretion

- Reducer callable signature (2-arg vs 3-arg with branch index)
- Exact location of new validation code (new file vs extending existing validators)
- Optional telemetry metrics for composition scenarios

## Deferred Ideas

- Pairwise tree reduce (if a workload demonstrates need)
- Deep recursive merge as separate `merge_deep` strategy
- Named ReducerRegistry (layer on top of dotted-path resolution later)
- Allow-subgraph-branches opt-in flag (not adopted)
- Partial output capture from failed child runs
- Per-subgraph step budgets
- Fan-out nesting depth cap (covered by existing `max_depth`)
