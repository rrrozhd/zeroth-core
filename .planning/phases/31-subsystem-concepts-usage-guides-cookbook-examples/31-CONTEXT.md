# Phase 31: Subsystem Concepts, Usage Guides, Cookbook & Examples - Context

**Gathered:** 2026-04-11
**Status:** Ready for planning
**Mode:** Inline discuss (autonomous workflow)

<domain>
## Phase Boundary

Fill the Diátaxis Concepts and How-to quadrants of the docs site (scaffolded in Phase 30) with complete coverage of every major `zeroth.core.*` subsystem, plus a Cookbook of 10 cross-subsystem recipes and a runnable `examples/` directory smoke-tested in CI.

This phase is pure content creation — no library code changes except (optionally) small tweaks to make existing modules tutorialable. The site infrastructure, landing page, and Getting Started + Governance Walkthrough tutorials from Phase 30 remain unchanged.

</domain>

<decisions>
## Implementation Decisions

### D-01 Scope — 20 subsystems, full coverage
Ship all 20 subsystems from the phase spec with **paired Concept + Usage Guide** pages each. Keep each page concise: ~300 words for Concept, ~400-500 words for Usage Guide with one minimal runnable example.

**Canonical subsystem list** (from ROADMAP phase 31):
1. graph
2. orchestrator
3. agents (→ `agent_runtime`)
4. execution units (`execution_units`)
5. memory
6. contracts
7. runs
8. conditions
9. mappings
10. policy
11. approvals
12. audit
13. secrets
14. identity
15. guardrails
16. dispatch
17. economics (→ `econ`)
18. storage
19. service
20. threads (if a threads module doesn't exist, substitute with `deployments` or `webhooks` — planner decides based on code)

### D-02 Content source — Claude writes from code + docstrings
Each subsystem page is synthesized by reading the actual module under `src/zeroth/core/<subsystem>/`, extracting the purpose, key types, and one realistic use case. Not mkdocstrings-only — Claude writes pedagogical prose, but cross-links to auto-generated reference (Phase 32).

### D-03 Concept page structure (~300 words)
Fixed template so readers know what to expect:
1. **What it is** (1-2 sentences)
2. **Why it exists** (1 paragraph — the design problem it solves)
3. **Where it fits** (1 paragraph — relationship to other subsystems, with one link to each adjacent Concept)
4. **Key types** (3-5 named types from the module with one-line descriptions)
5. **See also** (cross-links to the paired Usage Guide + adjacent Concepts)

### D-04 Usage Guide page structure (~400-500 words)
1. **Overview** (1 paragraph recapping the problem)
2. **Minimal example** (10-20 line runnable snippet, embedded from `examples/` via pymdownx.snippets where practical)
3. **Common patterns** (3-4 named patterns with one-line each)
4. **Pitfalls** (3-5 numbered items, specific and actionable)
5. **Reference cross-link** (link to the Phase 32 auto-generated API reference stub)

### D-05 Cookbook — 10 recipes, Claude chooses
The 10 recipes cover the most common cross-subsystem Zeroth tasks. Initial list (planner can refine after reading code):
1. Add an approval step to a node
2. Attach memory to an agent
3. Cap a run's cost budget
4. Sandbox a tool call
5. Retry a failing webhook with backoff
6. Block a tool call via policy
7. Query the audit trail for a run
8. Hand off between two agents mid-graph
9. Branch execution on a condition
10. Inject a secret into an execution unit

Each recipe: ~200 words, 1 runnable snippet, "When to use" + "When NOT to use" sections.

### D-06 Examples directory — runnable .py files
Extend the existing `examples/` directory (from Phases 28 + 30) with one file per cookbook recipe (10 new files). Each file:
- Env-gated with SKIP fallback (matches `hello.py`, `first_graph.py`, `approval_demo.py`, `governance_walkthrough.py`)
- Uses OpenAI via litellm fallback for LLM calls
- Self-contained — runs against a single zeroth-core service or in-process
- ~50-100 lines each

### D-07 CI — run every example with SKIP fallback
Extend `.github/workflows/examples.yml` (created in Phase 30) to add the 10 new cookbook examples to its matrix. Each entry runs the file; files without needed env keys exit 0 with SKIP. Job is green on every commit to main.

### D-08 Organization / navigation in mkdocs.yml
- `docs/concepts/{subsystem}.md` — 20 concept pages
- `docs/how-to/{subsystem}.md` — 20 usage guide pages
- `docs/how-to/cookbook/{recipe-slug}.md` — 10 cookbook pages
- `mkdocs.yml` nav updated to list all pages under the Concepts, How-to Guides, and Cookbook sections
- Section index pages (`concepts/index.md`, `how-to/index.md`, `how-to/cookbook/index.md`) become meaningful landing pages listing their contents

### D-09 Wave strategy — subsystem batching
Given 40 subsystem pages + 10 recipes + 10 examples, the planner should batch subsystems into 3-4 waves of ~5-6 subsystems each, each wave dispatched as a single executor agent writing ~10-12 pages. Cookbook + examples is one final wave. This keeps per-agent token usage reasonable and allows some parallelism.

### Claude's Discretion
- Exact subsystem ordering within waves (group by domain similarity: graph/orchestrator/conditions/mappings together, memory/storage/secrets together, policy/approvals/audit/guardrails together, dispatch/econ/service/webhooks together, etc.)
- Exact cookbook recipe list (10 chosen after reading actual code — the list in D-05 is a starting point)
- Whether to add mermaid diagrams for the Concept "Where it fits" sections (nice-to-have if mermaid plugin is already in the mkdocs.yml)
- Threads module substitution if `zeroth.core.threads` doesn't exist
- Whether execution_units gets its own page or is merged into the orchestrator Concept

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- All 27 subsystem directories under `src/zeroth/core/` — read these to write the pages
- `examples/hello.py`, `first_graph.py`, `approval_demo.py`, `governance_walkthrough.py` — pattern templates for new examples
- `zeroth.core.examples.quickstart` helper (from Phase 30-01) — reusable graph builder
- `.github/workflows/examples.yml` — extend its matrix
- `mkdocs.yml` — extend nav section
- `docs/concepts/index.md` and `docs/how-to/index.md` — currently scaffold pages, make them meaningful landing pages

### Established Patterns
- pymdownx.snippets embedding from examples/ into docs/
- mkdocs build --strict as the gate
- uv for all Python ops
- OpenAI + litellm fallback for example LLM usage
- SKIP-on-missing-env convention

### Integration Points
- Each subsystem page cross-links to adjacent subsystems — don't write pages in isolation
- Reference quadrant (Phase 32) will be linked from each Usage Guide's "Reference cross-link" section — use consistent anchor format so Phase 32 doesn't have to rewrite links
- Cookbook recipes embed from examples/ files

</code_context>

<specifics>
## Specific Ideas

- Each page is ~300-500 words — quality over length
- Minimal examples must be runnable, not pseudo-code
- Concept pages describe the "why" clearly — this is Diátaxis's core Concepts quadrant value
- Cookbook recipes all follow the same "When to use / When NOT to use" structure for scannability
- Navigation must not exceed mkdocs-material's comfortable depth — group into sections if needed

</specifics>

<deferred>
## Deferred Ideas

- Auto-generated Python API reference (Phase 32)
- Auto-rendered HTTP API reference (Phase 32)
- Configuration reference (Phase 32)
- Deployment Guide (Phase 32)
- Migration Guide (Phase 32)
- Versioned docs (mike)
- Custom domain, PR previews, social cards
- Advanced tutorials beyond Getting Started + Governance Walkthrough
- Per-subsystem deep-dive pages (these stay at Concept + Usage Guide granularity for this phase)

</deferred>
