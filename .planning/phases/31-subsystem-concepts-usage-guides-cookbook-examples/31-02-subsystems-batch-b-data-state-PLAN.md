---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - docs/concepts/mappings.md
  - docs/concepts/memory.md
  - docs/concepts/storage.md
  - docs/concepts/contracts.md
  - docs/concepts/runs.md
  - docs/how-to/mappings.md
  - docs/how-to/memory.md
  - docs/how-to/storage.md
  - docs/how-to/contracts.md
  - docs/how-to/runs.md
autonomous: true
requirements:
  - DOCS-03
  - DOCS-04
subsystem_map:
  mappings: src/zeroth/core/mappings/
  memory: src/zeroth/core/memory/
  storage: src/zeroth/core/storage/
  contracts: src/zeroth/core/contracts/
  runs: src/zeroth/core/runs/

must_haves:
  truths:
    - "Reader can open /concepts/{mappings,memory,storage,contracts,runs}.md and understand each subsystem's purpose"
    - "Reader can open /how-to/{mappings,memory,storage,contracts,runs}.md and copy a runnable minimal example"
    - "memory Usage Guide clarifies the pgvector/chroma/es connector split and points at the installable extras"
    - "contracts Concept page clarifies the distinction between Zeroth contracts and ordinary pydantic models"
    - "runs Usage Guide shows how to inspect a run's state after execution"
  artifacts:
    - path: docs/concepts/mappings.md
      min_lines: 40
      contains: "## What it is"
    - path: docs/concepts/memory.md
      min_lines: 40
    - path: docs/concepts/storage.md
      min_lines: 40
    - path: docs/concepts/contracts.md
      min_lines: 40
    - path: docs/concepts/runs.md
      min_lines: 40
    - path: docs/how-to/mappings.md
      min_lines: 50
      contains: "## Minimal example"
    - path: docs/how-to/memory.md
      min_lines: 50
    - path: docs/how-to/storage.md
      min_lines: 50
    - path: docs/how-to/contracts.md
      min_lines: 50
    - path: docs/how-to/runs.md
      min_lines: 50
  key_links:
    - from: docs/concepts/memory.md
      to: docs/concepts/storage.md
      via: "'Where it fits' cross-link (memory persists via storage)"
      pattern: "concepts/storage|storage.md"
    - from: docs/how-to/memory.md
      to: "pyproject.toml extras"
      via: "mention of `pip install 'zeroth-core[memory-pg]'` etc."
      pattern: "memory-pg|memory-chroma|memory-es"
---

<objective>
Ship 10 docs pages covering the second batch of 5 subsystems (mappings, memory, storage, contracts, runs) — the data/state slice of Zeroth. One Concept + one Usage Guide per subsystem.

Purpose: Fill the Concepts and How-to quadrants for everything related to moving data between nodes, persisting it, typing it, and inspecting completed runs (D-03 + D-04 from CONTEXT).
Output: 10 new markdown files. No mkdocs.yml changes — plan 31-05 handles nav.
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

@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-docs-site-scaffold-SUMMARY.md

Source subsystems (the ONLY sources of truth):
@src/zeroth/core/mappings/
@src/zeroth/core/memory/
@src/zeroth/core/storage/
@src/zeroth/core/contracts/
@src/zeroth/core/runs/

Example files to cite/embed:
@examples/first_graph.py

Installable extras (for memory page):
@pyproject.toml

<page_templates>
<!-- SAME templates as 31-01. Both MANDATORY. -->

Concept page (~300 words, 5 required H2 sections): `## What it is`, `## Why it exists`, `## Where it fits`, `## Key types`, `## See also`.

Usage Guide (~400-500 words, 5 required H2 sections): `## Overview`, `## Minimal example`, `## Common patterns`, `## Pitfalls`, `## Reference cross-link`.

Filename rule: last module segment, lowercased. All 10 files in this batch are single-word slugs.
</page_templates>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write Concept + Usage Guide pages for mappings, contracts, runs</name>
  <files>
    docs/concepts/mappings.md,
    docs/concepts/contracts.md,
    docs/concepts/runs.md,
    docs/how-to/mappings.md,
    docs/how-to/contracts.md,
    docs/how-to/runs.md
  </files>
  <action>
    For each of mappings, contracts, runs:

    1. Read src/zeroth/core/{module}/ — package __init__.py, main module files, key domain classes. Identify 3-5 key public types.

    2. Write docs/concepts/{slug}.md using the Concept template. ~300 words. "Where it fits":
       - mappings: cross-link to concepts/graph.md (mappings transport data along edges)
       - contracts: cross-link to concepts/mappings.md and concepts/agents.md (contracts type node I/O)
       - runs: cross-link to concepts/orchestrator.md (orchestrator produces runs)

    3. Write docs/how-to/{slug}.md using the Usage Guide template. ~400-500 words. Minimal examples MUST be runnable with zeroth.core.* imports.
       - mappings: show declaring a mapping between two nodes
       - contracts: show defining a contract and attaching it to a node
       - runs: show executing a graph and reading back a run's state (status, outputs)

    4. Every Usage Guide ends with `## Reference cross-link` pointing at `../reference/python-api.md`.

    Do NOT touch mkdocs.yml. Do NOT modify index.md files. Do NOT add mermaid diagrams.
  </action>
  <verify>
    <automated>for f in docs/concepts/mappings.md docs/concepts/contracts.md docs/concepts/runs.md docs/how-to/mappings.md docs/how-to/contracts.md docs/how-to/runs.md; do test -f "$f" || exit 1; done && grep -l "## What it is" docs/concepts/mappings.md docs/concepts/contracts.md docs/concepts/runs.md && grep -l "## Minimal example" docs/how-to/mappings.md docs/how-to/contracts.md docs/how-to/runs.md</automated>
  </verify>
  <done>Six files exist with all required H2 sections. All code imports from zeroth.core.*. Cross-links to Batch A pages (graph, orchestrator, agents) use relative paths.</done>
</task>

<task type="auto">
  <name>Task 2: Write Concept + Usage Guide pages for memory, storage</name>
  <files>
    docs/concepts/memory.md,
    docs/concepts/storage.md,
    docs/how-to/memory.md,
    docs/how-to/storage.md
  </files>
  <action>
    For memory and storage (closely related — memory connectors persist via storage backends):

    1. Read src/zeroth/core/memory/ — especially pgvector_connector.py, chroma_connector.py, and any ES connector present. Read src/zeroth/core/storage/ — especially the repository/session module(s).

    2. Write docs/concepts/memory.md. Key types: the memory connector interface plus the concrete connectors. "Where it fits" cross-links to concepts/storage.md AND concepts/agents.md. Explain the three installable flavours (pgvector, chroma, ES) at a conceptual level.

    3. Write docs/concepts/storage.md. Focus on the persistence layer for runs/approvals/audit. Cross-link to concepts/runs.md and concepts/memory.md.

    4. Write docs/how-to/memory.md. Minimal example MUST use an in-memory/sqlite fallback that runs without extras installed. In "Common patterns" mention that the pgvector/chroma/es connectors require the matching extras — provide the exact install commands:
       - `pip install 'zeroth-core[memory-pg]'`
       - `pip install 'zeroth-core[memory-chroma]'`
       - `pip install 'zeroth-core[memory-es]'`
       Verify these three extra names by reading pyproject.toml before writing.

    5. Write docs/how-to/storage.md. Minimal example: open a session, query runs, close. Pitfalls: connection pooling, Alembic migrations, sqlite vs postgres differences.

    Do NOT touch mkdocs.yml. Do NOT modify index.md files.
  </action>
  <verify>
    <automated>test -f docs/concepts/memory.md && test -f docs/concepts/storage.md && test -f docs/how-to/memory.md && test -f docs/how-to/storage.md && grep -q "memory-pg" docs/how-to/memory.md && grep -q "memory-chroma" docs/how-to/memory.md && grep -q "memory-es" docs/how-to/memory.md</automated>
  </verify>
  <done>Four files exist. memory Usage Guide documents all three extras with exact install commands matching pyproject.toml. storage pages cross-link to runs + memory.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build` (non-strict) succeeds
- All 10 files exist at expected paths
- Each Concept page has 5 required H2 sections; each Usage Guide has 5 required H2 sections
- Memory Usage Guide contains all three `memory-*` extras
- No occurrences of `from zeroth.` or `import zeroth.` that aren't `zeroth.core.`
</verification>

<success_criteria>
- 10 new markdown files committed under docs/concepts/ and docs/how-to/
- mkdocs build passes
- memory page correctly documents the three pgvector/chroma/es extras
- Cross-links to plan 31-01 pages resolve correctly
</success_criteria>

<output>
After completion, create `.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-02-SUMMARY.md` listing the 10 files shipped, key types per subsystem, and any cross-connector pitfalls discovered (especially Alembic/migration gotchas).
</output>
