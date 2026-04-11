---
phase: 31-subsystem-concepts-usage-guides-cookbook-examples
plan: 05
type: execute
wave: 2
depends_on:
  - 31-01
  - 31-02
  - 31-03
  - 31-04
files_modified:
  - docs/how-to/cookbook/index.md
  - docs/how-to/cookbook/approval-step.md
  - docs/how-to/cookbook/attach-memory.md
  - docs/how-to/cookbook/budget-cap.md
  - docs/how-to/cookbook/sandbox-tool.md
  - docs/how-to/cookbook/webhook-retry.md
  - docs/how-to/cookbook/policy-block.md
  - docs/how-to/cookbook/audit-query.md
  - docs/how-to/cookbook/agent-handoff.md
  - docs/how-to/cookbook/condition-branch.md
  - docs/how-to/cookbook/secret-injection.md
  - docs/concepts/index.md
  - docs/how-to/index.md
  - examples/approval_step.py
  - examples/attach_memory.py
  - examples/budget_cap.py
  - examples/sandbox_tool.py
  - examples/webhook_retry.py
  - examples/policy_block.py
  - examples/audit_query.py
  - examples/agent_handoff.py
  - examples/condition_branch.py
  - examples/secret_injection.py
  - .github/workflows/examples.yml
  - mkdocs.yml
autonomous: true
requirements:
  - DOCS-06
  - DOCS-12

cookbook_recipes:
  - slug: approval-step
    title: "Add a human approval step to a node"
    example: examples/approval_step.py
    subsystems: [approvals, graph]
  - slug: attach-memory
    title: "Attach memory to an agent"
    example: examples/attach_memory.py
    subsystems: [memory, agents]
  - slug: budget-cap
    title: "Cap a run's cost budget"
    example: examples/budget_cap.py
    subsystems: [econ, runs]
  - slug: sandbox-tool
    title: "Sandbox a tool call"
    example: examples/sandbox_tool.py
    subsystems: [execution_units, guardrails]
  - slug: webhook-retry
    title: "Retry a failing webhook with backoff"
    example: examples/webhook_retry.py
    subsystems: [webhooks, dispatch]
  - slug: policy-block
    title: "Block a tool call via policy"
    example: examples/policy_block.py
    subsystems: [policy, execution_units]
  - slug: audit-query
    title: "Query the audit trail for a run"
    example: examples/audit_query.py
    subsystems: [audit, runs]
  - slug: agent-handoff
    title: "Hand off between two agents mid-graph"
    example: examples/agent_handoff.py
    subsystems: [agents, orchestrator]
  - slug: condition-branch
    title: "Branch execution on a condition"
    example: examples/condition_branch.py
    subsystems: [conditions, graph]
  - slug: secret-injection
    title: "Inject a secret into an execution unit"
    example: examples/secret_injection.py
    subsystems: [secrets, execution_units]

autonomous_true_rationale: "No external credentials or manual ops — all 10 examples use the same SKIP-on-missing-env pattern as hello.py/first_graph.py, and the mkdocs strict build runs under uv."

must_haves:
  truths:
    - "A reader can open docs/how-to/cookbook/index.md and see all 10 recipes listed"
    - "Each of the 10 recipe pages has a runnable example and 'When to use' + 'When NOT to use' sections"
    - "Each of the 10 examples/*.py files runs with `uv run python examples/{file}.py` — SKIPs cleanly when needed env vars are missing"
    - "CI workflow .github/workflows/examples.yml runs every one of the 10 new example files on push to main"
    - "mkdocs.yml nav lists every Concept page (20), every Usage Guide (20), and every Cookbook recipe (10)"
    - "`uv run mkdocs build --strict` succeeds on the fully populated site"
    - "docs/concepts/index.md and docs/how-to/index.md are meaningful landing pages listing their children, not the current scaffolding stubs"
  artifacts:
    - path: docs/how-to/cookbook/index.md
      provides: "Cookbook landing page listing all 10 recipes"
      min_lines: 20
    - path: mkdocs.yml
      provides: "Nav covering all 40 subsystem pages + cookbook"
      contains: "Cookbook"
    - path: .github/workflows/examples.yml
      provides: "CI matrix covering all 14 example files (4 existing + 10 new)"
      contains: "examples/approval_step.py"
  key_links:
    - from: docs/how-to/cookbook/index.md
      to: "all 10 recipe pages"
      via: "bullet list with relative links"
      pattern: "approval-step.md"
    - from: mkdocs.yml
      to: docs/how-to/cookbook/
      via: "nav: Cookbook subsection"
      pattern: "cookbook"
    - from: .github/workflows/examples.yml
      to: "all 10 new examples"
      via: "one `run:` step per file"
      pattern: "examples/approval_step.py"
---

<objective>
Close out phase 31 by shipping: (a) 10 cookbook recipe pages under docs/how-to/cookbook/, (b) 10 runnable example files under examples/, (c) CI matrix extensions so every example runs on every main commit, (d) meaningful landing pages for docs/concepts/index.md and docs/how-to/index.md, (e) full mkdocs.yml nav covering all 40 subsystem pages and all 10 cookbook recipes, and (f) a passing `mkdocs build --strict` gate on the fully populated site.

Purpose: Deliver DOCS-06 (cookbook) and DOCS-12 (examples + CI smoke test) and finalize the site's navigation for phase 31. This plan depends on 31-01..04 having already written all 40 Concept + Usage Guide pages — without them, mkdocs --strict will fail on missing nav targets.
Output: 10 recipe pages, 10 example files, 2 rewritten index pages, extended CI workflow, finalized mkdocs.yml nav, and a green strict build.
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
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-01-SUMMARY.md
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-02-SUMMARY.md
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-03-SUMMARY.md
@.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-04-SUMMARY.md

@mkdocs.yml
@.github/workflows/examples.yml

Canonical example file templates — all 10 new examples MUST follow these SKIP-on-missing-env patterns:
@examples/hello.py
@examples/first_graph.py
@examples/approval_demo.py
@examples/governance_walkthrough.py

Existing scaffold pages to rewrite into meaningful landing pages:
@docs/concepts/index.md
@docs/how-to/index.md

<cookbook_page_template>
Every recipe file in this plan MUST follow this structure (~200 words + snippet):

```markdown
# {Recipe title}

## What this recipe does
{1-2 sentences.}

## When to use
- Bullet
- Bullet
- Bullet

## When NOT to use
- Bullet
- Bullet

## Recipe
```python
--8<-- "examples/{slug}.py"
```
*(Or embed a sliced snippet if the file is long.)*

## How it works
{2-3 sentences tying the snippet to the subsystems it touches.}

## See also
- [Usage Guide: {primary subsystem}](../{subsystem}.md)
- [Concept: {primary subsystem}](../../concepts/{subsystem}.md)
```

The `--8<-- "examples/..."` syntax is pymdownx.snippets which is already enabled in mkdocs.yml (`base_path: [".", "examples"]`).
</cookbook_page_template>

<example_file_template>
Every new file under examples/ MUST follow this shape (based on examples/hello.py and examples/first_graph.py):

```python
"""{Recipe title} — runnable example for docs/how-to/cookbook/{slug}.md."""

from __future__ import annotations

import os
import sys


def main() -> int:
    # Early SKIP if the example needs a credential that is not available.
    required_env = [...]  # empty list if no creds needed
    missing = [k for k in required_env if not os.environ.get(k)]
    if missing:
        print(f"SKIP: missing env vars: {', '.join(missing)}")
        return 0

    # Real example body. Must use zeroth.core.* imports only.
    # Aim for 50-100 lines total including imports and main().

    print("OK")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

Files that do not need any credentials (e.g. in-process examples) MUST still use this shape but with `required_env = []` — the SKIP branch is then unreachable but the scaffold is consistent for CI and readers.
</example_file_template>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write the 10 example files under examples/</name>
  <files>
    examples/approval_step.py,
    examples/attach_memory.py,
    examples/budget_cap.py,
    examples/sandbox_tool.py,
    examples/webhook_retry.py,
    examples/policy_block.py,
    examples/audit_query.py,
    examples/agent_handoff.py,
    examples/condition_branch.py,
    examples/secret_injection.py
  </files>
  <action>
    Create one runnable Python file per recipe in the `cookbook_recipes` frontmatter list. Each file:

    1. Follows the `<example_file_template>` exactly: module docstring naming its cookbook page, `main()` returning int, SKIP-on-missing-env pattern, zeroth.core.* imports only.
    2. Is 50-100 lines including imports.
    3. Uses the real module APIs — read the relevant subsystem directory in src/zeroth/core/ before writing anything. Do not invent APIs. Prefer reusing patterns from examples/first_graph.py, examples/approval_demo.py, and examples/governance_walkthrough.py.
    4. Uses `required_env = ["OPENAI_API_KEY"]` only if an LLM call is actually required for the recipe (e.g. agent-handoff, attach-memory). Pure plumbing recipes (condition-branch, secret-injection, policy-block, audit-query) should set `required_env = []` and run fully in-process.
    5. Exits 0 in all normal paths. Prints a short success string on the happy path (matches existing examples' conventions).
    6. Is formatted to pass `uv run ruff check examples/`.

    Verify each new file compiles and runs:
    ```
    uv run ruff check examples/
    for f in examples/approval_step.py examples/attach_memory.py examples/budget_cap.py examples/sandbox_tool.py examples/webhook_retry.py examples/policy_block.py examples/audit_query.py examples/agent_handoff.py examples/condition_branch.py examples/secret_injection.py; do
      uv run python "$f" || { echo "FAILED: $f"; exit 1; };
    done
    ```

    If any file can't SKIP cleanly and would fail without live services (e.g. webhook_retry needs Redis), print SKIP and exit 0 when the service is not reachable. Match the existing CONNECTION-REFUSED -> SKIP pattern in approval_demo.py.
  </action>
  <verify>
    <automated>uv run ruff check examples/ && for f in examples/approval_step.py examples/attach_memory.py examples/budget_cap.py examples/sandbox_tool.py examples/webhook_retry.py examples/policy_block.py examples/audit_query.py examples/agent_handoff.py examples/condition_branch.py examples/secret_injection.py; do uv run python "$f" > /dev/null || exit 1; done</automated>
  </verify>
  <done>10 example files exist, pass ruff, and all run to exit 0 locally (happy path or SKIP). Each uses zeroth.core.* only.</done>
</task>

<task type="auto">
  <name>Task 2: Write the 10 cookbook recipe pages and the cookbook index</name>
  <files>
    docs/how-to/cookbook/index.md,
    docs/how-to/cookbook/approval-step.md,
    docs/how-to/cookbook/attach-memory.md,
    docs/how-to/cookbook/budget-cap.md,
    docs/how-to/cookbook/sandbox-tool.md,
    docs/how-to/cookbook/webhook-retry.md,
    docs/how-to/cookbook/policy-block.md,
    docs/how-to/cookbook/audit-query.md,
    docs/how-to/cookbook/agent-handoff.md,
    docs/how-to/cookbook/condition-branch.md,
    docs/how-to/cookbook/secret-injection.md
  </files>
  <action>
    1. Create docs/how-to/cookbook/index.md — the Cookbook landing page. List all 10 recipes with one-line descriptions and relative links. Brief intro paragraph explaining that each recipe is a runnable cross-subsystem task-oriented recipe.

    2. For each of the 10 recipes, create docs/how-to/cookbook/{slug}.md using the `<cookbook_page_template>`. Every page MUST:
       - Have all 6 required H2 sections
       - Embed the matching examples/{slug}.py via pymdownx.snippets: ```--8<-- "{slug}.py"``` (note: snippets base_path is `[".", "examples"]` so the path is relative to examples/)
       - Link "See also" to the appropriate subsystem Concept + Usage Guide pages from plans 31-01..04 using relative paths (e.g. `../../concepts/approvals.md`)

    3. The snippets syntax is `--8<--` (pymdownx.snippets). Double-check by reading mkdocs.yml's markdown_extensions section — it's already configured with base_path = [".", "examples"].

    Do NOT modify mkdocs.yml yet (Task 4 does that).
  </action>
  <verify>
    <automated>test -f docs/how-to/cookbook/index.md && ls docs/how-to/cookbook/*.md | wc -l | grep -q "^11$" && grep -l "When to use" docs/how-to/cookbook/approval-step.md docs/how-to/cookbook/attach-memory.md docs/how-to/cookbook/budget-cap.md docs/how-to/cookbook/sandbox-tool.md docs/how-to/cookbook/webhook-retry.md docs/how-to/cookbook/policy-block.md docs/how-to/cookbook/audit-query.md docs/how-to/cookbook/agent-handoff.md docs/how-to/cookbook/condition-branch.md docs/how-to/cookbook/secret-injection.md</automated>
  </verify>
  <done>11 files exist under docs/how-to/cookbook/. Every recipe page has "When to use", "When NOT to use", a snippet embed, and a "See also" block with resolved links.</done>
</task>

<task type="auto">
  <name>Task 3: Rewrite docs/concepts/index.md and docs/how-to/index.md as meaningful landing pages</name>
  <files>
    docs/concepts/index.md,
    docs/how-to/index.md
  </files>
  <action>
    1. Rewrite docs/concepts/index.md. Currently a 2-line stub. New content:
       - 1 paragraph explaining the Diátaxis Concepts quadrant and how Zeroth uses it
       - A categorized list of all 20 subsystems, grouped as: Execution (graph, orchestrator, agents, execution-units, conditions), Data & state (mappings, memory, storage, contracts, runs), Governance (policy, approvals, audit, guardrails, identity), Platform (secrets, dispatch, econ, service, webhooks)
       - Each list item = bullet link to the concept page with a one-line summary
       - Note that webhooks was substituted for the original "threads" slot (cross-reference the substitution from plan 31-04)

    2. Rewrite docs/how-to/index.md. Currently a 2-line stub. New content:
       - 1 paragraph explaining task-oriented Usage Guides vs Concepts
       - Two H2 sections: "Subsystem usage guides" (20 items, same grouping as concepts/index.md) and "Cookbook" (link to cookbook/index.md)

    Use relative links (`./graph.md`, `./cookbook/index.md`).
  </action>
  <verify>
    <automated>grep -q "Execution\|Data" docs/concepts/index.md && grep -q "Cookbook\|cookbook" docs/how-to/index.md && wc -l docs/concepts/index.md docs/how-to/index.md | grep -v "^ *[0-9] "</automated>
  </verify>
  <done>Both index pages are meaningful landing pages with categorized links to all 20 subsystems. Concepts index notes the threads->webhooks substitution.</done>
</task>

<task type="auto">
  <name>Task 4: Finalize mkdocs.yml nav and extend .github/workflows/examples.yml</name>
  <files>
    mkdocs.yml,
    .github/workflows/examples.yml
  </files>
  <action>
    1. Update mkdocs.yml nav block. Keep everything above "How-to Guides" as-is. Replace the How-to Guides and Concepts sections with the full finalized shape:

       ```yaml
       - How-to Guides:
         - how-to/index.md
         - Subsystems:
           - Graph: how-to/graph.md
           - Orchestrator: how-to/orchestrator.md
           - Agents: how-to/agents.md
           - Execution units: how-to/execution-units.md
           - Conditions: how-to/conditions.md
           - Mappings: how-to/mappings.md
           - Memory: how-to/memory.md
           - Storage: how-to/storage.md
           - Contracts: how-to/contracts.md
           - Runs: how-to/runs.md
           - Policy: how-to/policy.md
           - Approvals: how-to/approvals.md
           - Audit: how-to/audit.md
           - Guardrails: how-to/guardrails.md
           - Identity: how-to/identity.md
           - Secrets: how-to/secrets.md
           - Dispatch: how-to/dispatch.md
           - Economics: how-to/econ.md
           - Service: how-to/service.md
           - Webhooks: how-to/webhooks.md
         - Cookbook:
           - how-to/cookbook/index.md
           - Add an approval step: how-to/cookbook/approval-step.md
           - Attach memory to an agent: how-to/cookbook/attach-memory.md
           - Cap a run's cost budget: how-to/cookbook/budget-cap.md
           - Sandbox a tool call: how-to/cookbook/sandbox-tool.md
           - Retry a failing webhook: how-to/cookbook/webhook-retry.md
           - Block a tool call via policy: how-to/cookbook/policy-block.md
           - Query the audit trail: how-to/cookbook/audit-query.md
           - Hand off between agents: how-to/cookbook/agent-handoff.md
           - Branch on a condition: how-to/cookbook/condition-branch.md
           - Inject a secret: how-to/cookbook/secret-injection.md
       - Concepts:
         - concepts/index.md
         - Graph: concepts/graph.md
         - Orchestrator: concepts/orchestrator.md
         - Agents: concepts/agents.md
         - Execution units: concepts/execution-units.md
         - Conditions: concepts/conditions.md
         - Mappings: concepts/mappings.md
         - Memory: concepts/memory.md
         - Storage: concepts/storage.md
         - Contracts: concepts/contracts.md
         - Runs: concepts/runs.md
         - Policy: concepts/policy.md
         - Approvals: concepts/approvals.md
         - Audit: concepts/audit.md
         - Guardrails: concepts/guardrails.md
         - Identity: concepts/identity.md
         - Secrets: concepts/secrets.md
         - Dispatch: concepts/dispatch.md
         - Economics: concepts/econ.md
         - Service: concepts/service.md
         - Webhooks: concepts/webhooks.md
       ```

    2. Extend .github/workflows/examples.yml. After the existing `Run examples/governance_walkthrough.py (if present)` step, append ten new steps — one per new example file — using the same shape as the existing steps. Example:

       ```yaml
       - name: Run examples/approval_step.py
         run: uv run python examples/approval_step.py
       ```

       Do this for all 10 new files. Do NOT remove or alter the existing 4 steps.

    3. Keep `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` env forwarding as-is. The SKIP pattern in each example file handles forked-PR runs without secrets.
  </action>
  <verify>
    <automated>grep -c "concepts/graph.md\|concepts/orchestrator.md\|concepts/agents.md\|concepts/execution-units.md\|concepts/conditions.md\|concepts/mappings.md\|concepts/memory.md\|concepts/storage.md\|concepts/contracts.md\|concepts/runs.md\|concepts/policy.md\|concepts/approvals.md\|concepts/audit.md\|concepts/guardrails.md\|concepts/identity.md\|concepts/secrets.md\|concepts/dispatch.md\|concepts/econ.md\|concepts/service.md\|concepts/webhooks.md" mkdocs.yml | awk '$1<20 {exit 1}' && grep -q "examples/approval_step.py" .github/workflows/examples.yml && grep -q "examples/secret_injection.py" .github/workflows/examples.yml</automated>
  </verify>
  <done>mkdocs.yml nav lists all 20 concept pages, all 20 usage guides, and all 10 cookbook recipes. examples.yml runs all 14 example files (4 existing + 10 new).</done>
</task>

<task type="auto">
  <name>Task 5: Run `mkdocs build --strict` gate and fix any link/snippet errors</name>
  <files>
    (fixes land wherever the strict build reports a missing link or missing snippet path)
  </files>
  <action>
    Run:
    ```
    uv run mkdocs build --strict 2>&1 | tee /tmp/mkdocs-strict.log
    ```

    The strict build fails on: missing nav targets, broken links, missing snippet source files, orphaned pages. Fix every reported error.

    Common fixes expected:
    - Relative links in subsystem pages that use bad `../` depth — adjust path
    - `--8<-- "{slug}.py"` where the examples/ base_path means you wrote the wrong prefix — it should be just `{slug}.py` with the snippets base_path set to `[".", "examples"]`
    - Missing cross-references — fix the source page
    - Orphaned pages (not in nav) — add to mkdocs.yml or delete the file

    Do NOT disable strict mode. Do NOT remove pages to dodge errors — fix the underlying issue. Rerun until exit 0.
  </action>
  <verify>
    <automated>uv run mkdocs build --strict</automated>
  </verify>
  <done>`uv run mkdocs build --strict` exits 0 on the fully populated site. Output is logged for the SUMMARY.</done>
</task>

</tasks>

<verification>
- `uv run mkdocs build --strict` exits 0
- All 10 new example files run to exit 0 locally (happy path or SKIP)
- `uv run ruff check examples/` passes
- `.github/workflows/examples.yml` references all 14 example files (4 existing + 10 new)
- docs/how-to/cookbook/ contains exactly 11 files (index + 10 recipes)
- mkdocs.yml nav covers all 40 subsystem pages + 10 cookbook recipes + both index pages
- docs/concepts/index.md and docs/how-to/index.md are meaningful landing pages, not 2-line stubs
</verification>

<success_criteria>
- DOCS-06 satisfied: cookbook has ≥10 cross-subsystem recipes
- DOCS-12 satisfied: examples/ has runnable .py files and CI smoke-tests each on main
- Phase 31 goal satisfied: Concepts + How-to quadrants fully populated with 40 subsystem pages, 10 cookbook recipes, and meaningful index pages
- Strict mkdocs build passes on the fully populated site
- No regressions: existing 4 example files still run; existing tutorial pages untouched
</success_criteria>

<output>
After completion, create `.planning/phases/31-subsystem-concepts-usage-guides-cookbook-examples/31-05-SUMMARY.md` with:
- The 10 example files shipped and their SKIP conditions
- The 10 recipe pages and their subsystem pairings
- mkdocs.yml nav diff summary
- examples.yml diff summary
- The strict build output (exit 0 line + page count)
- Any fixes made in Task 5 that should inform Phase 32 (broken anchor patterns, missing reference/ targets, etc.)
</output>
