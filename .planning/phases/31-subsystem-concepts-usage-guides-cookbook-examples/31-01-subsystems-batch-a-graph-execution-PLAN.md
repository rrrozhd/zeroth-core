---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
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
autonomous: true
requirements:
  - DOCS-03
  - DOCS-04
subsystem_map:
  graph: src/zeroth/core/graph/
  orchestrator: src/zeroth/core/orchestrator/
  agent_runtime: src/zeroth/core/agent_runtime/
  execution_units: src/zeroth/core/execution_units/
  conditions: src/zeroth/core/conditions/

must_haves:
  truths:
    - "Reader can open /concepts/graph.md and understand what a Zeroth graph is, why it exists, and how it relates to the orchestrator, without reading source"
    - "Reader can open /how-to/graph.md and copy a runnable minimal example that builds a graph with at least one node + edge"
    - "Same holds for orchestrator, agents, execution-units, conditions"
    - "Each Concept page links to its paired Usage Guide and at least one adjacent Concept (e.g. graph <-> orchestrator)"
    - "Each Usage Guide ends with a 'Reference cross-link' anchor pointing at the Phase 32 auto-generated API reference (stub link is acceptable)"
  artifacts:
    - path: docs/concepts/graph.md
      provides: "Concept page for zeroth.core.graph"
      min_lines: 40
      contains: "## What it is"
    - path: docs/concepts/orchestrator.md
      provides: "Concept page for zeroth.core.orchestrator"
      min_lines: 40
    - path: docs/concepts/agents.md
      provides: "Concept page for zeroth.core.agent_runtime (pedagogical name: agents)"
      min_lines: 40
    - path: docs/concepts/execution-units.md
      provides: "Concept page for zeroth.core.execution_units"
      min_lines: 40
    - path: docs/concepts/conditions.md
      provides: "Concept page for zeroth.core.conditions"
      min_lines: 40
    - path: docs/how-to/graph.md
      provides: "Usage guide for graph"
      min_lines: 50
      contains: "## Minimal example"
    - path: docs/how-to/orchestrator.md
      provides: "Usage guide for orchestrator"
      min_lines: 50
    - path: docs/how-to/agents.md
      provides: "Usage guide for agents"
      min_lines: 50
    - path: docs/how-to/execution-units.md
      provides: "Usage guide for execution_units"
      min_lines: 50
    - path: docs/how-to/conditions.md
      provides: "Usage guide for conditions"
      min_lines: 50
  key_links:
    - from: docs/concepts/graph.md
      to: docs/how-to/graph.md
      via: "'See also' section"
      pattern: "how-to/graph"
    - from: docs/concepts/orchestrator.md
      to: docs/concepts/graph.md
      via: "'Where it fits' cross-link to adjacent concept"
      pattern: "concepts/graph"
---

<objective>
Ship 10 docs pages covering the first batch of 5 core subsystems (graph, orchestrator, agents, execution_units, conditions) — one Concept + one Usage Guide per subsystem — synthesized directly from source code under `src/zeroth/core/`.

Purpose: Fill the Concepts and How-to quadrants for the execution/graph slice of Zeroth. This is the subsystem group users meet first when building a workflow (D-03 + D-04 from CONTEXT).
Output: 10 new markdown files under docs/concepts/ and docs/how-to/ that mkdocs build --strict accepts. No mkdocs.yml nav changes in this plan — plan 31-05 handles nav finalization.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-CONTEXT.md

Prior phase summaries (for site conventions, snippet embedding, strict build gate):
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-docs-site-scaffold-SUMMARY.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-03-getting-started-tutorial-SUMMARY.md

Source subsystems to read (the ONLY sources of truth for page content):
@src/zeroth/core/graph/
@src/zeroth/core/orchestrator/
@src/zeroth/core/agent_runtime/
@src/zeroth/core/execution_units/
@src/zeroth/core/conditions/

Existing example files to link/embed from:
@examples/first_graph.py
@examples/approval_demo.py

Existing index pages to leave alone in this plan (31-05 will rewrite them):
@docs/concepts/index.md
@docs/how-to/index.md

<page_templates>
<!-- Both templates are MANDATORY. Every page in this plan MUST follow these shapes. -->

Concept page template (~300 words, 5 required H2 sections):

```markdown
# {Subsystem display name}

## What it is
{1-2 sentences. Plain English.}

## Why it exists
{1 paragraph. The design problem it solves. Cite the pain point it removes.}

## Where it fits
{1 paragraph. Relationship to adjacent subsystems. Link at least one adjacent Concept page, e.g. [orchestrator](orchestrator.md).}

## Key types
- **`ClassName`** — one-line description
- **`ClassName`** — one-line description
- (3-5 items total, all pulled from the real module)

## See also
- [Usage Guide: {subsystem}](../how-to/{subsystem}.md)
- [Concept: {adjacent}](./{adjacent}.md)
```

Usage Guide template (~400-500 words, 5 required H2 sections):

```markdown
# {Subsystem display name}: usage guide

## Overview
{1 paragraph recapping the problem this subsystem solves in concrete terms.}

## Minimal example
```python
# 10-20 line runnable snippet. Must actually import from zeroth.core.*.
# Prefer embedding via pymdownx.snippets if a matching examples/*.py exists.
```

## Common patterns
- **Pattern name** — one-line
- **Pattern name** — one-line
- (3-4 items)

## Pitfalls
1. Specific, actionable pitfall
2. Specific, actionable pitfall
3. (3-5 items, numbered)

## Reference cross-link
See the [Python API reference for `zeroth.core.{subsystem}`](../reference/python-api.md#zerothcore{subsystem}) (generated in Phase 32).
```

Filename convention:
- `zeroth.core.agent_runtime` → pedagogical name "agents" → `concepts/agents.md` + `how-to/agents.md`
- `zeroth.core.execution_units` → `concepts/execution-units.md` + `how-to/execution-units.md` (hyphenated)
- All others: `{last-module-segment}.md`
</page_templates>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write Concept + Usage Guide pages for graph, orchestrator, agents</name>
  <files>
    docs/concepts/graph.md,
    docs/concepts/orchestrator.md,
    docs/concepts/agents.md,
    docs/how-to/graph.md,
    docs/how-to/orchestrator.md,
    docs/how-to/agents.md
  </files>
  <action>
    For each of graph, orchestrator, agent_runtime:

    1. Read the module under src/zeroth/core/{module}/ — at minimum the package __init__.py and the main entrypoint file(s). Identify 3-5 key public types/classes and the subsystem's design purpose.

    2. Write docs/concepts/{slug}.md using the Concept template from <page_templates>. Exactly the 5 H2 sections. ~300 words. "Where it fits" MUST link to at least one adjacent Concept page in this batch (graph <-> orchestrator, orchestrator <-> agents).

    3. Write docs/how-to/{slug}.md using the Usage Guide template. Exactly the 5 H2 sections. ~400-500 words. The "Minimal example" code block MUST import from zeroth.core.* and MUST be runnable on its own — prefer reusing patterns from examples/first_graph.py. Cite the paired Concept page in "Overview". End with a "Reference cross-link" anchor pointing at ../reference/python-api.md (the target does not exist yet; the link is a forward reference — use a plain markdown link, not @file syntax).

    4. Slug mapping: graph → graph, orchestrator → orchestrator, agent_runtime → agents.

    Do NOT touch mkdocs.yml. Do NOT modify docs/concepts/index.md or docs/how-to/index.md (plan 31-05 handles nav and index pages).
    Do NOT add mermaid diagrams — the mermaid plugin is not enabled in mkdocs.yml.
    All code snippets MUST parse as valid Python 3.12. Prefer copying verbatim from examples/first_graph.py where applicable rather than inventing new APIs.
  </action>
  <verify>
    <automated>test -f docs/concepts/graph.md && test -f docs/concepts/orchestrator.md && test -f docs/concepts/agents.md && test -f docs/how-to/graph.md && test -f docs/how-to/orchestrator.md && test -f docs/how-to/agents.md && grep -l "## What it is" docs/concepts/graph.md docs/concepts/orchestrator.md docs/concepts/agents.md && grep -l "## Minimal example" docs/how-to/graph.md docs/how-to/orchestrator.md docs/how-to/agents.md</automated>
  </verify>
  <done>Six files exist. Each Concept page has all 5 required H2 sections. Each Usage Guide has all 5 required H2 sections. Every snippet imports from zeroth.core.* and no snippet uses the legacy `zeroth.*` path.</done>
</task>

<task type="auto">
  <name>Task 2: Write Concept + Usage Guide pages for execution_units, conditions</name>
  <files>
    docs/concepts/execution-units.md,
    docs/concepts/conditions.md,
    docs/how-to/execution-units.md,
    docs/how-to/conditions.md
  </files>
  <action>
    For execution_units and conditions:

    1. Read src/zeroth/core/execution_units/ and src/zeroth/core/conditions/ — package inits, main module files. Identify 3-5 key types each.

    2. Write docs/concepts/execution-units.md and docs/concepts/conditions.md using the Concept template. Cross-link "Where it fits" between these two and to graph/orchestrator from Task 1 (conditions branches a graph; execution_units runs inside a graph node).

    3. Write docs/how-to/execution-units.md and docs/how-to/conditions.md using the Usage Guide template. Each Minimal example must be runnable. For execution_units, show defining and wiring one unit. For conditions, show a branching condition on a run's state.

    4. Verify all inter-page links in this plan resolve to files that exist after Task 1 (graph, orchestrator, agents) or this task (execution-units, conditions). Use relative paths (`../how-to/{slug}.md`, `./{slug}.md`) so mkdocs link-checking passes.

    Do NOT touch mkdocs.yml. Do NOT modify index.md files.
  </action>
  <verify>
    <automated>test -f docs/concepts/execution-units.md && test -f docs/concepts/conditions.md && test -f docs/how-to/execution-units.md && test -f docs/how-to/conditions.md && grep -c "## " docs/concepts/execution-units.md docs/concepts/conditions.md docs/how-to/execution-units.md docs/how-to/conditions.md | grep -v ":[01234]$"</automated>
  </verify>
  <done>Four files exist. Each has at least 5 H2 sections. Cross-links to graph/orchestrator/agents use relative paths that resolve. No references to legacy `zeroth.*` imports.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build` (non-strict) succeeds — build failures here will block the strict gate in plan 31-05, so catch them early
- All 10 files exist at expected paths
- Every Concept page contains all 5 required H2 sections (`## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`)
- Every Usage Guide page contains all 5 required H2 sections (`## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`)
- Every code fence labelled `python` parses (manual spot-check via `python -c "compile(open('...').read(), '...', 'exec')"` on extracted snippets, or just careful eyeballing against examples/first_graph.py)
- Zero occurrences of `from zeroth.` or `import zeroth.` that are NOT `from zeroth.core.` / `import zeroth.core.` in any new file
</verification>

<success_criteria>
- 10 new markdown files committed under docs/concepts/ and docs/how-to/
- `uv run mkdocs build` completes without errors
- Every Concept page cross-links to its paired Usage Guide
- Every Usage Guide has a reference-cross-link pointing at ../reference/python-api.md
- No scope creep into plan 31-05 territory (no mkdocs.yml edits, no index.md rewrites, no cookbook content)
</success_criteria>

<output>
After completion, create `.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-01-SUMMARY.md` listing the 10 files shipped, the key types surfaced per subsystem, and any source-code surprises worth noting for sibling plans 31-02..04.
</output>
