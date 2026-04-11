---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 01
subsystem: docs
tags: [docs, tutorial, quickstart, graph, examples]
requires: [zeroth.core.graph.models, zeroth.core.policy.models]
provides:
  - zeroth.core.examples package
  - zeroth.core.examples.quickstart.build_demo_graph
  - zeroth.core.examples.quickstart.build_demo_graph_with_policy
  - tests/test_docs_phase30.py scaffold for plans 02-05
affects: []
tech-stack:
  added: []
  patterns:
    - Tutorial helpers live under src/ (installed package) not top-level examples/
    - Helpers marked unstable in docstring (no deprecation cycle required)
key-files:
  created:
    - src/zeroth/core/examples/__init__.py
    - src/zeroth/core/examples/quickstart.py
    - tests/examples/__init__.py
    - tests/examples/test_quickstart.py
    - tests/test_docs_phase30.py
  modified: []
key-decisions:
  - Place tutorial helpers under src/zeroth/core/examples/ (installed) instead of top-level examples/ so docs snippets can `from zeroth.core.examples.quickstart import build_demo_graph` against the published wheel without path hacks
  - Keep build_demo_graph_with_policy as a thin reference binder — it sets policy_bindings=["block-demo-caps"] and records denied capabilities in capability_bindings but does not persist a PolicyDefinition; tutorial scripts (plan 04) are responsible for registering the actual policy
  - Scaffold tests/test_docs_phase30.py with one always-passing placeholder so plans 02-05 add shape assertions instead of bootstrapping the file
requirements-completed: [DOCS-02]
duration: 5 min
completed: 2026-04-11
---

# Phase 30 Plan 01: Quickstart Helper Module Summary

Shipped `zeroth.core.examples.quickstart` — a ~150 LOC tutorial helper exposing `build_demo_graph(instruction, llm_model, include_approval)` and `build_demo_graph_with_policy(denied_capabilities)` so the Getting Started tutorial (Plan 30-03) and Governance Walkthrough (Plan 30-04) can build a valid agent+tool(+approval) `Graph` in a single import, and created the Wave 0 `tests/test_docs_phase30.py` scaffold that Plans 30-02 through 30-05 will populate with mkdocs/docs-tree shape assertions.

## What Shipped

### `zeroth.core.examples` package
- New installed package at `src/zeroth/core/examples/` with a docstring that explicitly marks it as a **tutorial helper, NOT a stable API surface**.
- Import path works against the installed wheel: `from zeroth.core.examples.quickstart import build_demo_graph`.

### `build_demo_graph(...)` API
```python
def build_demo_graph(
    instruction: str = "...",
    llm_model: str = "openai/gpt-4o-mini",
    *,
    include_approval: bool = False,
) -> Graph
```
- Returns a two-node linear `Graph`: one `AgentNode` (`node_id="agent"`) wired to one `ExecutableUnitNode` (`node_id="tool"`) via a single `Edge`.
- When `include_approval=True`, splices a `HumanApprovalNode` (`node_id="approval"`) between agent and tool — used by the Governance Walkthrough to demonstrate the approval gate.
- Agent instruction and LLM model are parameterized so the tutorial can show both OpenAI and other litellm-compatible providers.

### `build_demo_graph_with_policy(denied_capabilities)` API
- Returns the demo graph with the tool node's `policy_bindings=["block-demo-caps"]` and `capability_bindings` recording the denied capabilities.
- Does **not** persist a `PolicyDefinition` — Plan 30-04's example script is responsible for registering the actual policy. This keeps the helper pure and test-free from policy-store dependencies.

### `tests/test_docs_phase30.py`
- Scaffold file with one always-passing `test_phase30_scaffold_present` placeholder and a module docstring explaining "Populated across Plans 30-01 through 30-05".
- Plans 02-05 will add real assertions (mkdocs.yml shape, docs/ tree, CI workflow) without needing to bootstrap the file.

## Tasks & Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | RED: Create package + test scaffolds | `f877d9f` | `src/zeroth/core/examples/__init__.py`, `tests/examples/__init__.py`, `tests/examples/test_quickstart.py`, `tests/test_docs_phase30.py` |
| 2 | GREEN: Implement quickstart helpers | `08e8fe5` | `src/zeroth/core/examples/quickstart.py` |

## Verification Results

- `uv run pytest tests/examples/test_quickstart.py tests/test_docs_phase30.py -v` → **8 passed**
- `uv run python -c "from zeroth.core.examples.quickstart import build_demo_graph; g = build_demo_graph(); assert g.graph_id and len(g.nodes) >= 2"` → **exit 0**
- `uv run ruff check src/zeroth/core/examples/ tests/examples/` → **All checks passed**
- `uv run interrogate -c pyproject.toml src/zeroth/core/examples/` → **PASSED (100.0%, minimum 90.0%)**
- Full suite: `uv run pytest -q` → **670 passed, 8 deselected, 1 pre-existing warning** (no regressions)

## Success Criteria

- [x] `zeroth.core.examples.quickstart` importable from the installed package
- [x] `build_demo_graph()` returns a `Graph` with ≥1 agent node and ≥1 tool node (verified by `test_build_demo_graph_has_agent_and_tool_nodes`)
- [x] `build_demo_graph(include_approval=True)` returns a graph containing a `HumanApprovalNode`
- [x] `build_demo_graph_with_policy([...])` returns a graph with tool `policy_bindings` populated
- [x] Helper marked as tutorial-only (module + function docstrings both state "NOT a stable API")
- [x] `tests/test_docs_phase30.py` exists as the Wave 0 scaffold
- [x] `interrogate` docstring gate still passes (100.0%)
- [x] No regressions in existing suite (670 passed)

## Deviations from Plan

None - plan executed exactly as written. Ruff formatter collapsed one `Edge(...)` argument list onto a single line in Task 2 (~1 line change). Counted as cosmetic, not a deviation.

## Threat Flags

None — no new network endpoints, auth paths, or trust boundaries introduced. This is pure in-process model construction with no I/O.

## Ready for Next Plan

Plan 30-02 (docs site scaffold / mkdocs) can:
1. Import `build_demo_graph` from the installed package in tutorial snippets.
2. Append mkdocs shape assertions to `tests/test_docs_phase30.py`.

## Self-Check: PASSED

- [x] `src/zeroth/core/examples/__init__.py` exists on disk
- [x] `src/zeroth/core/examples/quickstart.py` exists on disk
- [x] `tests/examples/__init__.py` exists on disk
- [x] `tests/examples/test_quickstart.py` exists on disk
- [x] `tests/test_docs_phase30.py` exists on disk
- [x] Commit `f877d9f` present in git log
- [x] Commit `08e8fe5` present in git log
