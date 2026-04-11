---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 01
subsystem: docs
tags: [docs, concepts, how-to, diataxis, graph, orchestrator, agents, execution-units, conditions]
requires: [30-02]
provides:
  - Concept + Usage Guide pages for graph, orchestrator, agents, execution-units, conditions
affects:
  - docs/concepts/
  - docs/how-to/
tech-stack:
  added: []
  patterns:
    - "Diataxis 5-H2 Concept template (What it is / Why / Where it fits / Key types / See also)"
    - "Diataxis 5-H2 Usage Guide template (Overview / Minimal example / Common patterns / Pitfalls / Reference cross-link)"
    - "Relative markdown cross-links between adjacent Concept pages"
    - "Forward-reference stub links to Phase 32 ../reference/python-api.md"
key-files:
  created:
    - docs/concepts/graph.md
    - docs/concepts/orchestrator.md
    - docs/concepts/agents.md
    - docs/concepts/execution-units.md
    - docs/concepts/conditions.md
    - docs/how-to/graph.md
    - docs/how-to/orchestrator.md
    - docs/how-to/agents.md
    - docs/how-to/execution-units.md
    - docs/how-to/conditions.md
  modified: []
key-decisions:
  - "Used relative markdown links (./sibling.md, ../how-to/slug.md) everywhere so mkdocs link-checking resolves without nav edits"
  - "Mapped zeroth.core.agent_runtime -> pedagogical slug 'agents' and zeroth.core.execution_units -> hyphenated slug 'execution-units' per plan"
  - "Left mkdocs.yml and index.md files untouched — 31-05 owns nav and section landing pages"
  - "Reference cross-link anchors use kebab-free forms like #zerothcoregraph so Phase 32's mkdocstrings output is the single source of anchors"
requirements-completed:
  - DOCS-03
  - DOCS-04
duration: ~15 min
completed: 2026-04-11
---

# Phase 31 Plan 01: Subsystem Batch A (graph + execution) Summary

**One-liner:** Shipped 10 Diátaxis pages (5 Concepts + 5 Usage Guides) for graph, orchestrator, agents, execution-units, and conditions — synthesized directly from `src/zeroth/core/` modules, cross-linked, and validated by a non-strict `mkdocs build`.

## Scope

Five subsystems covered end-to-end with paired Concept + Usage Guide pages:

| Subsystem          | Concept page                       | Usage Guide page                 |
| ------------------ | ---------------------------------- | -------------------------------- |
| `graph`            | `docs/concepts/graph.md`           | `docs/how-to/graph.md`           |
| `orchestrator`     | `docs/concepts/orchestrator.md`    | `docs/how-to/orchestrator.md`    |
| `agent_runtime`    | `docs/concepts/agents.md`          | `docs/how-to/agents.md`          |
| `execution_units`  | `docs/concepts/execution-units.md` | `docs/how-to/execution-units.md` |
| `conditions`       | `docs/concepts/conditions.md`      | `docs/how-to/conditions.md`      |

Every page follows the mandatory 5-H2 template from the plan (verified via `grep -c '^## '`).

## Key types surfaced per subsystem

**graph** — `Graph`, `Node` (discriminated union of `AgentNode`/`ExecutableUnitNode`/`HumanApprovalNode`), `Edge`, `GraphStatus`, `GraphRepository`.

**orchestrator** — `RuntimeOrchestrator`, `OrchestratorError`, `NodeDispatcherError`, `Run` (from `zeroth.core.runs`), `AgentRunner`/`ExecutableUnitRunner` dispatch interfaces.

**agents** (`agent_runtime`) — `AgentRunner`, `AgentConfig`, `ProviderAdapter` (+ `LiteLLMProviderAdapter`/`GovernedLLMProviderAdapter`/`DeterministicProviderAdapter`), `PromptAssembler`/`PromptAssembly`, `ToolAttachmentRegistry`.

**execution-units** — `ExecutableUnitRunner`, `ExecutableUnitManifest` (union of `NativeUnitManifest`/`WrappedCommandUnitManifest`/`ProjectUnitManifest`), `ExecutableUnitRegistry`, `SandboxManager`/`SandboxConfig`, `AdmissionController`.

**conditions** — `NextStepPlanner`, `ConditionEvaluator`, `BranchResolver`, `ConditionBinder`/`ConditionBinding`, `ConditionResultRecorder`.

## Source-code surprises (for sibling plans 31-02..04)

1. **`zeroth.core.graph` already re-exports the full node/edge/graph model surface** from its `__init__.py` — batch B/C authors writing pages for adjacent modules (mappings, contracts) can import directly from `zeroth.core.graph` without diving into `models.py`.

2. **`RuntimeOrchestrator` is a `@dataclass(slots=True)`** with injected collaborators (run repo, policy guard, audit, approvals, memory, mappings, conditions, secrets). Docs for policy/approvals/audit should emphasize that those subsystems plug into the *orchestrator*, not into individual nodes.

3. **`ExecutableUnitNode` is defined in `zeroth.core.graph.models`, not in `zeroth.core.execution_units`** — the execution_units package owns manifests, runner, and sandbox; the graph package owns the *node* that references a manifest. Worth cross-linking explicitly in future pages.

4. **`ConditionContext` / `TraversalState` live in `zeroth.core.conditions.models`** and are imported by the orchestrator — condition expressions execute in that context, so the mappings/runs doc pages can reuse the same mental model.

5. **Agent runtime has a `DeterministicProviderAdapter`** — ideal for example snippets and tests that can't hit a real provider. Cookbook recipes in plan 31-06 should prefer it over mocking.

6. **`examples/first_graph.py` uses stub `_LiteLLMAgentRunner` / `_EchoExecutableUnitRunner` classes** rather than the real `AgentRunner` — flagged as "tutorial helper" in the docstring. The `docs/how-to/agents.md` minimal example shows the production `AgentRunner` shape directly so readers don't confuse the tutorial shortcut with the real API.

## Deviations from Plan

None — plan executed exactly as written. No bugs, missing functionality, or blocking issues found. `mkdocs build` (non-strict) succeeds with only pre-existing warnings from unrelated subsystem pages (identity, storage, audit, webhooks) that are out of scope for this plan and belong to sibling batches 31-02..04.

## Verification

- `uv run mkdocs build` — succeeded in 0.58s (warnings only, none from files introduced by this plan)
- All 10 target files exist on disk
- Every Concept page contains all 5 required H2 sections (verified via `grep -c '^## '` → 5 for each)
- Every Usage Guide page contains all 5 required H2 sections (verified the same way)
- Every code fence labelled `python` imports only from `zeroth.core.*` (no legacy `zeroth.*` paths)
- Every Usage Guide ends with a stub link to `../reference/python-api.md` for Phase 32 resolution
- No `mkdocs.yml` or `docs/**/index.md` edits — scope respected for plan 31-05

## Commits

- `71d792d` — docs(31-01): add concept + usage guide pages for graph, orchestrator, agents
- `b487c68` — docs(31-01): add concept + usage guide pages for execution-units, conditions

## Issues Encountered

None.

## Next Phase Readiness

Ready for plan **31-02** (subsystem batch B) — no follow-ups blocking. Sibling plans can reuse the template shape and cross-link pattern verbatim. When Phase 32 lands, the `../reference/python-api.md#zerothcore*` anchors in every Usage Guide should resolve automatically.

## Self-Check: PASSED

- All 10 created files exist on disk (verified via prior `grep` and `mkdocs build` output listing each file)
- Both task commits present in `git log` (`71d792d`, `b487c68`)
- `mkdocs build` succeeded; no errors originating from files in this plan
