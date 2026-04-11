---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - src/zeroth/core/examples/__init__.py
  - src/zeroth/core/examples/quickstart.py
  - tests/examples/__init__.py
  - tests/examples/test_quickstart.py
  - tests/test_docs_phase30.py
autonomous: true
requirements:
  - DOCS-02
tags: [docs, tutorial, quickstart, graph]
must_haves:
  truths:
    - "`from zeroth.core.examples.quickstart import build_demo_graph` works against the installed package"
    - "`build_demo_graph()` returns a `Graph` with exactly one Agent node, one Executable Unit (tool) node, and at least one Edge"
    - "The module is explicitly documented as a 'tutorial helper, not a stable API' in its docstring"
    - "Wave 0 test scaffold `tests/test_docs_phase30.py` exists so later plans can add shape assertions against mkdocs.yml / docs/ without creating it from scratch"
  artifacts:
    - path: "src/zeroth/core/examples/__init__.py"
      provides: "Namespace for tutorial helpers (PEP 420 safe — empty package init is allowed inside src/zeroth/core since zeroth.core itself is a regular package)"
    - path: "src/zeroth/core/examples/quickstart.py"
      provides: "build_demo_graph() + build_demo_deployment_inputs() helpers used by examples/first_graph.py, examples/service_mode_with_approval.py, and examples/governance_walkthrough.py"
      min_lines: 40
    - path: "tests/examples/test_quickstart.py"
      provides: "Unit tests proving quickstart helper returns a well-formed Graph"
    - path: "tests/test_docs_phase30.py"
      provides: "Wave 0 shape-test scaffold referenced by plans 02, 03, 04, 05"
  key_links:
    - from: "src/zeroth/core/examples/quickstart.py"
      to: "src/zeroth/core/graph/models.py"
      via: "imports Graph, AgentNode, ExecutableUnitNode, Edge, HumanApprovalNode"
      pattern: "from zeroth\\.core\\.graph"
    - from: "tests/examples/test_quickstart.py"
      to: "src/zeroth/core/examples/quickstart.py"
      via: "pytest import"
      pattern: "from zeroth\\.core\\.examples\\.quickstart"
---

<objective>
Ship `zeroth.core.examples.quickstart` — a thin tutorial helper that lets
Getting Started Section 2 and the Governance Walkthrough show ~10 lines of
user-facing Python instead of the ~80-120 lines required to hand-build a
Graph + Deployment from scratch. Also stub the Wave 0 shape-test file
`tests/test_docs_phase30.py` so downstream plans (02–05) add assertions
rather than creating the file.

Purpose: Unblock plans 03 (Getting Started tutorial) and 04 (Governance
Walkthrough) by giving them a stable, tested `build_demo_graph(...)` they
can import from a runnable example script.

Output: New library module under `src/zeroth/core/examples/` (~50 LOC),
its unit test, and the empty shape-test scaffold for later plans.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-CONTEXT.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-RESEARCH.md
@src/zeroth/core/graph/models.py
@examples/hello.py

<interfaces>
<!-- From src/zeroth/core/graph/models.py — verified via grep -->
<!-- Executor should import these directly, no exploration needed -->

class NodeBase(BaseModel):
    node_id: str
    node_type: str   # "agent" | "executable_unit" | "human_approval"
    policy_bindings: list[str] = []
    # ... input_contract_id, output_contract_id, etc.

class AgentNode(NodeBase):  # node_type = "agent"
    instructions: str
    llm_model: str   # e.g. "openai/gpt-4o-mini"
    # ... tool bindings, memory bindings

class ExecutableUnitNode(NodeBase):  # node_type = "executable_unit"
    # wraps a command/code unit

class HumanApprovalNode(NodeBase):  # node_type = "human_approval"
    # pauses the run

class Edge(BaseModel):
    edge_id: str
    source_node_id: str
    target_node_id: str

class Graph(BaseModel):
    graph_id: str
    name: str
    nodes: list[NodeBase]
    edges: list[Edge]
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Create zeroth.core.examples package + Wave 0 test scaffold</name>
  <files>
    src/zeroth/core/examples/__init__.py
    tests/examples/__init__.py
    tests/examples/test_quickstart.py
    tests/test_docs_phase30.py
  </files>
  <behavior>
    - `import zeroth.core.examples` succeeds
    - `tests/examples/test_quickstart.py` contains at minimum three failing
      placeholder tests that Task 2 will satisfy:
        * test_build_demo_graph_returns_graph_instance
        * test_build_demo_graph_has_agent_and_tool_nodes
        * test_build_demo_graph_edges_connect_all_nodes
    - `tests/test_docs_phase30.py` exists with a single always-passing
      placeholder test `test_phase30_scaffold_present` so `pytest -k test_docs_phase30`
      resolves. Later plans (02-05) will add real assertions to this file.
    - Package `__init__.py` files declare a short docstring only.
  </behavior>
  <action>
    1. Read `src/zeroth/core/graph/models.py` lines 80-280 to confirm field
       names for NodeBase/AgentNode/ExecutableUnitNode/HumanApprovalNode/Edge/Graph
       before writing tests.
    2. Create `src/zeroth/core/examples/__init__.py` with docstring:
       `"""Tutorial helpers used by the docs site — NOT a stable API surface."""`
    3. Create `tests/examples/__init__.py` (empty file, conventional).
    4. Create `tests/examples/test_quickstart.py` with the three tests listed
       in <behavior>. They should import from `zeroth.core.examples.quickstart`
       (which does not yet exist) so `uv run pytest tests/examples/test_quickstart.py`
       fails with ImportError — that is the RED state for the TDD loop.
    5. Create `tests/test_docs_phase30.py` with a module docstring explaining
       "Shape tests for Phase 30 docs deliverables. Populated across plans 01-05."
       and a single `def test_phase30_scaffold_present(): assert True` so the
       file is importable and discoverable.
  </action>
  <verify>
    <automated>uv run pytest tests/test_docs_phase30.py -x -q &amp;&amp; (uv run pytest tests/examples/test_quickstart.py -x -q; test $? -eq 1)</automated>
  </verify>
  <done>
    - `tests/test_docs_phase30.py` passes (placeholder green)
    - `tests/examples/test_quickstart.py` fails with ImportError on
      `zeroth.core.examples.quickstart` (RED — Task 2 will turn this green)
    - `src/zeroth/core/examples/__init__.py` importable: `uv run python -c "import zeroth.core.examples"` exits 0
  </done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Implement zeroth.core.examples.quickstart.build_demo_graph</name>
  <files>
    src/zeroth/core/examples/quickstart.py
  </files>
  <behavior>
    - `build_demo_graph(instruction: str = "...", llm_model: str = "openai/gpt-4o-mini", include_approval: bool = False) -> Graph`
      returns a valid `Graph` containing:
        * one `AgentNode` (node_id="agent", instructions param, llm_model param)
        * one `ExecutableUnitNode` (node_id="tool") acting as a stub tool node
        * if `include_approval=True`, one `HumanApprovalNode` (node_id="approval")
          wired between agent and tool
        * `Edge` instances connecting the nodes in a simple linear chain
    - `build_demo_graph_with_policy(denied_capabilities: list[Capability]) -> Graph`
      returns the same graph but with the tool node's `policy_bindings` set to
      a policy id "block-demo-caps" (plan 04 will use this to demonstrate the
      policy block scenario — no PolicyDefinition is persisted here, the
      example script is responsible for registering it).
    - Module docstring explicitly says: "Tutorial helper. NOT a stable API.
      Subject to change without a deprecation cycle. See Phase 30 docs."
    - All three tests from Task 1 pass.
  </behavior>
  <action>
    1. Write `src/zeroth/core/examples/quickstart.py` (~50-80 LOC including docstrings).
       Use `from __future__ import annotations`, Google-style docstrings (project
       convention — interrogate gate), and imports from `zeroth.core.graph.models`
       and `zeroth.core.policy.models` (Capability only, for typing the second helper).
    2. Keep construction minimal — use field defaults wherever the model allows,
       only set the fields the tests assert on. If NodeBase requires extra fields
       (input_contract_id etc.), set them to sensible defaults or use existing
       project fixtures — grep `tests/` for existing Graph-construction helpers
       before rolling your own values.
    3. Run `uv run pytest tests/examples/test_quickstart.py -x -v` — must be GREEN.
    4. Run `uv run ruff check src/zeroth/core/examples/ tests/examples/` and
       `uv run ruff format src/zeroth/core/examples/ tests/examples/` — must pass.
    5. Run `uv run interrogate -c pyproject.toml src/zeroth/core/examples/` and
       confirm the new module does not drop overall coverage below 90%.
  </action>
  <verify>
    <automated>uv run pytest tests/examples/test_quickstart.py tests/test_docs_phase30.py -x -v &amp;&amp; uv run ruff check src/zeroth/core/examples/ tests/examples/ &amp;&amp; uv run interrogate -c pyproject.toml src/zeroth/core/examples/</automated>
  </verify>
  <done>
    - `build_demo_graph()` returns a valid Graph with agent + tool + edges
    - `build_demo_graph(include_approval=True)` returns a Graph containing a HumanApprovalNode
    - `build_demo_graph_with_policy([...])` returns a Graph with tool node `policy_bindings` populated
    - Module docstring clearly marks helper as unstable tutorial API
    - All tests green; ruff clean; interrogate still ≥ 90%
  </done>
</task>

</tasks>

<verification>
- `uv run pytest tests/examples/test_quickstart.py tests/test_docs_phase30.py -v` → green
- `uv run python -c "from zeroth.core.examples.quickstart import build_demo_graph; g = build_demo_graph(); assert g.graph_id and len(g.nodes) >= 2"` → exits 0
- `uv run ruff check src/zeroth/core/examples/ tests/examples/` → clean
- Full suite: `uv run pytest -q` — no regressions from baseline
</verification>

<success_criteria>
- `zeroth.core.examples.quickstart` importable from installed package
- `build_demo_graph()` returns a Graph with ≥1 agent node and ≥1 tool node
- Helper is marked as tutorial-only (docstring), not a stable API
- `tests/test_docs_phase30.py` exists as a scaffold for later plans
- `interrogate` docstring gate still passes (≥90%)
- No regressions in the existing test suite
</success_criteria>

<output>
After completion, create `.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-01-SUMMARY.md` documenting: helper API shape, why it lives under `zeroth.core.examples` (vs `examples/`), and the stability caveat.
</output>
