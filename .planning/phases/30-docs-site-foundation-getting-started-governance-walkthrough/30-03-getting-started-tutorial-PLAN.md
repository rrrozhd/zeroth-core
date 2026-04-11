---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 03
type: execute
wave: 2
depends_on: [01, 02]
files_modified:
  - examples/first_graph.py
  - examples/approval_demo.py
  - docs/index.md
  - docs/tutorials/getting-started/index.md
  - docs/tutorials/getting-started/01-install.md
  - docs/tutorials/getting-started/02-first-graph.md
  - docs/tutorials/getting-started/03-service-and-approval.md
  - .github/workflows/examples.yml
  - tests/test_docs_phase30.py
autonomous: true
requirements:
  - DOCS-01
  - DOCS-02
tags: [docs, tutorial, getting-started, examples, ci]
must_haves:
  truths:
    - "User can run `python examples/first_graph.py` in a clean venv and either see a real LLM call's terminal output or a clean SKIP message — no stack traces"
    - "User can run `python examples/approval_demo.py` against a locally-booted `python -m zeroth.core.service.entrypoint`, POST a run, approve via curl, and see the run transition to succeeded — with no stack traces and a clean SKIP path when OPENAI_API_KEY is missing"
    - "`docs/tutorials/getting-started/01-install.md` embeds `examples/hello.py` via pymdownx.snippets and describes the <5 minute install path"
    - "`docs/tutorials/getting-started/02-first-graph.md` embeds `examples/first_graph.py` and shows the quickstart helper usage"
    - "`docs/tutorials/getting-started/03-service-and-approval.md` embeds `examples/approval_demo.py` and shows both the Python and curl paths for approval resolve (POST /deployments/{ref}/approvals/{id}/resolve)"
    - "Landing page Choose Your Path tabs link to the correct library vs service pages"
    - "`.github/workflows/examples.yml` CI job runs all four example scripts (hello, first_graph, approval_demo, governance_walkthrough — the last may not exist yet, handle gracefully) on push to main and on PR, with the SKIP guard preventing fork PRs from failing"
  artifacts:
    - path: "examples/first_graph.py"
      provides: "Runnable Getting Started section 2 example using zeroth.core.examples.quickstart.build_demo_graph + in-process bootstrap_service"
      min_lines: 60
    - path: "examples/approval_demo.py"
      provides: "Runnable section 3 example: boots service in-process OR expects running uvicorn, POSTs a run that hits HumanApprovalNode, prints approval_id + curl command, auto-resolves via httpx, prints succeeded status"
      min_lines: 60
    - path: ".github/workflows/examples.yml"
      provides: "GHA workflow that runs each examples/*.py on push/PR; SKIP on missing OPENAI_API_KEY"
    - path: "docs/tutorials/getting-started/02-first-graph.md"
      provides: "Tutorial prose with embedded first_graph.py snippet"
      min_lines: 30
  key_links:
    - from: "examples/first_graph.py"
      to: "src/zeroth/core/examples/quickstart.py"
      via: "import build_demo_graph"
      pattern: "from zeroth\\.core\\.examples\\.quickstart import"
    - from: "examples/approval_demo.py"
      to: "src/zeroth/core/service/approval_api.py"
      via: "POST /deployments/{ref}/approvals/{id}/resolve"
      pattern: "approvals/.+/resolve"
    - from: "docs/tutorials/getting-started/02-first-graph.md"
      to: "examples/first_graph.py"
      via: "pymdownx.snippets --8<--"
      pattern: "--8<--.*first_graph"
    - from: ".github/workflows/examples.yml"
      to: "examples/"
      via: "job step running each script"
      pattern: "examples/first_graph.py"
---

<objective>
Ship the complete 3-section Getting Started tutorial: landing page update,
install/first-graph/service-and-approval pages, two new runnable example
scripts, and a CI workflow that exercises every example on every commit.
Every non-trivial code block in the tutorial is embedded from an
`examples/*.py` file via `pymdownx.snippets` so docs and code can never
drift.

Purpose: Satisfy DOCS-01 (landing page) finalization and DOCS-02 (3-section
tutorial, <5 min first output, <30 min full tutorial).

Output: Four updated docs pages, two new runnable examples, and a CI
workflow that keeps them honest.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-CONTEXT.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-RESEARCH.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-01-SUMMARY.md
@.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-02-SUMMARY.md
@src/zeroth/core/examples/quickstart.py
@src/zeroth/core/service/entrypoint.py
@src/zeroth/core/service/run_api.py
@src/zeroth/core/service/approval_api.py
@src/zeroth/core/service/bootstrap.py
@examples/hello.py
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write examples/first_graph.py and examples/approval_demo.py</name>
  <files>
    examples/first_graph.py
    examples/approval_demo.py
  </files>
  <action>
    1. Read `src/zeroth/core/service/bootstrap.py` enough to locate
       `bootstrap_service()` and the entrypoint/app-factory shape. Read
       `src/zeroth/core/service/entrypoint.py` for the env-var contract
       (`ZEROTH_DEPLOYMENT_REF`, `PORT`, `ZEROTH_DATABASE__BACKEND`).

    2. Create `examples/first_graph.py` (~60-100 LOC) that:
         * Starts with the hello.py-style SKIP guard on `OPENAI_API_KEY`
           (exits 0 with a SKIP stderr message).
         * Imports `build_demo_graph` from `zeroth.core.examples.quickstart`.
         * Calls `bootstrap_service()` in-process with an in-memory or
           temp-file SQLite DB.
         * Persists the demo Graph via `GraphRepository`, creates a
           `Deployment` via `DeploymentService`, submits a run via the
           in-process run service (NOT over HTTP — this example is the
           library-embedded path, the curl path belongs to approval_demo.py).
         * Polls the run until terminal and prints the final status +
           output payload.
         * Top docstring: "Getting Started Section 2 — first graph with
           agent + tool + LLM call. Runs end-to-end against an in-memory
           SQLite. SKIPs if OPENAI_API_KEY is unset."
         * MUST exit 0 under SKIP path, and 0 under success path, and
           non-zero only on real bugs.

    3. Create `examples/approval_demo.py` (~60-120 LOC) that:
         * SKIP guard on OPENAI_API_KEY.
         * Boots the service in-process (same pattern) with a graph built
           via `build_demo_graph(include_approval=True)`.
         * Uses httpx `AsyncClient(app=..., base_url="http://test")` to
           POST `/runs`, poll to `paused_for_approval`, fetch the
           approval list via `GET /deployments/{ref}/approvals?run_id=...`,
           extract the `approval_id`.
         * Prints the exact curl command the reader would run manually:
           ```
           curl -X POST http://localhost:8000/deployments/default/approvals/$APPROVAL_ID/resolve \
                -H "Authorization: Bearer $TOKEN" \
                -H "Content-Type: application/json" \
                -d '{"decision": "approve"}'
           ```
         * Then automatically POSTs the resolve for the reader via httpx
           so the script is runnable end-to-end.
         * Polls to `succeeded` and prints the final run status.
         * Docstring: "Getting Started Section 3 — service mode with
           approval gate. Demonstrates both the automated resolve AND
           prints the equivalent curl command for human-in-the-loop use."

    4. Run both locally WITHOUT `OPENAI_API_KEY` — both must exit 0 with
       a SKIP message. Then run WITH `OPENAI_API_KEY` if you have one in
       the session; otherwise note in SUMMARY that the happy-path test
       relies on the main-branch CI job's configured secret.

    5. `uv run ruff check examples/first_graph.py examples/approval_demo.py`
       and format.
  </action>
  <verify>
    <automated>uv run python examples/first_graph.py &amp;&amp; uv run python examples/approval_demo.py &amp;&amp; uv run ruff check examples/</automated>
  </verify>
  <done>
    - Both scripts SKIP cleanly with exit code 0 when OPENAI_API_KEY unset
    - Both scripts import only from the public `zeroth.core.*` surface
    - approval_demo.py POSTs to the real `/deployments/{ref}/approvals/{id}/resolve` endpoint
    - Ruff clean
  </done>
</task>

<task type="auto">
  <name>Task 2: Write Getting Started tutorial pages + landing page finalization</name>
  <files>
    docs/index.md
    docs/tutorials/getting-started/index.md
    docs/tutorials/getting-started/01-install.md
    docs/tutorials/getting-started/02-first-graph.md
    docs/tutorials/getting-started/03-service-and-approval.md
  </files>
  <action>
    1. `docs/index.md` — replace placeholders inside the tabbed split
       with real prose linking to the right tutorial sections. Keep the
       hello.py snippet embed. The tabbed block should be:
         * "Embed as library" → short blurb, link to 02-first-graph.md
         * "Run as service" → short blurb, link to 03-service-and-approval.md

    2. `docs/tutorials/getting-started/index.md` — replace placeholder
       with a short overview: time budget, what the reader will have at
       the end, link-list to the three numbered sections.

    3. `docs/tutorials/getting-started/01-install.md` — write the
       install section. Include:
         * `pip install zeroth-core` block (real, not a snippet).
         * Mention `uv add zeroth-core` as alternative.
         * Note about optional extras, link to pyproject extras.
         * Embed `examples/hello.py` via `--8<-- "hello.py"`.
         * Step-by-step verification: set OPENAI_API_KEY, run the file,
           expect a one-line LLM greeting.
         * Explicitly call out: "this is the <5-minute gate".

    4. `docs/tutorials/getting-started/02-first-graph.md` — write section 2:
         * One paragraph explaining graphs, agents, tool nodes, and how
           Zeroth differs from just calling an LLM directly.
         * Code: embed `examples/first_graph.py` via
           `--8<-- "first_graph.py"`.
         * A callout box noting `zeroth.core.examples.quickstart` is
           "a tutorial helper, not a stable API — see Phase 31+ for the
           real graph authoring guide."
         * Expected output block showing the status/payload the reader
           should see.

    5. `docs/tutorials/getting-started/03-service-and-approval.md` — write
       section 3:
         * Prose explaining service mode vs library mode, the approval
           gate, and when to pause a run for a human.
         * Show the command to boot the service:
           `uv run python -m zeroth.core.service.entrypoint`
         * Embed `examples/approval_demo.py` via `--8<-- "approval_demo.py"`.
         * A separate "Approve via curl" subsection that copies the
           curl command printed by the example (can be hard-coded in
           prose — the example prints it verbatim so it's the same
           string).
         * Link back to the service auth docs (stubbed — Phase 32).

    6. Run `uv run mkdocs build --strict` — MUST pass. If any snippet
       path fails, fix it. Do NOT disable `check_paths`.
  </action>
  <verify>
    <automated>uv run mkdocs build --strict</automated>
  </verify>
  <done>
    - Four tutorial pages contain real prose, not placeholders
    - Landing page tabbed split links to the correct pages
    - Every code block in sections 2 and 3 is a `pymdownx.snippets` embed, not inlined
    - `mkdocs build --strict` passes
  </done>
</task>

<task type="auto">
  <name>Task 3: Add .github/workflows/examples.yml + shape tests</name>
  <files>
    .github/workflows/examples.yml
    tests/test_docs_phase30.py
  </files>
  <action>
    1. Create `.github/workflows/examples.yml`:
         * Name: `examples`
         * Triggers: `push: branches: [main]` and `pull_request: branches: [main]`
         * Job matrix optional — a single job on ubuntu-latest is fine.
         * Steps:
           - checkout
           - `astral-sh/setup-uv@v5`
           - `actions/setup-python@v5` with `python-version: "3.12"`
           - `uv sync --all-extras --group dev`
           - Run each example in sequence:
             `uv run python examples/hello.py`
             `uv run python examples/first_graph.py`
             `uv run python examples/approval_demo.py`
             (if `examples/governance_walkthrough.py` exists, run it too —
              use a shell `if [ -f ... ]; then ... fi` so plan 04 can add
              it later without changing this workflow)
         * Pass the `OPENAI_API_KEY` secret via `env:` only on the
           `push: main` event — forked PRs will exit via SKIP. Use:
           `env: { OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }} }` at
           the job level; the SKIP guard inside each script handles the
           missing-secret case.

    2. Extend `tests/test_docs_phase30.py` with:
         * `test_examples_workflow_exists`: assert file exists and has
           trigger `push.branches: [main]` and `pull_request.branches: [main]`.
         * `test_examples_workflow_runs_first_graph_and_approval`:
           assert the workflow string contains `examples/first_graph.py`
           AND `examples/approval_demo.py`.
         * `test_first_graph_page_embeds_example`: assert
           `docs/tutorials/getting-started/02-first-graph.md` contains
           `--8<--` and `first_graph.py`.
         * `test_approval_page_embeds_example_and_curl`: assert
           `docs/tutorials/getting-started/03-service-and-approval.md`
           contains `--8<--`, `approval_demo.py`, and the substring
           `/approvals/` + `/resolve` (to prove the curl command is shown).
         * `test_landing_tabs_link_to_getting_started`: assert
           `docs/index.md` contains links to
           `tutorials/getting-started/02-first-graph.md` AND
           `tutorials/getting-started/03-service-and-approval.md`.

    3. Run `uv run pytest tests/test_docs_phase30.py -v` — all green.
    4. Run `uv run ruff check tests/test_docs_phase30.py` and format.
  </action>
  <verify>
    <automated>uv run pytest tests/test_docs_phase30.py -v &amp;&amp; uv run mkdocs build --strict</automated>
  </verify>
  <done>
    - `.github/workflows/examples.yml` exists with both triggers and runs first_graph.py + approval_demo.py
    - New shape tests green
    - Strict mkdocs build green
    - Governance walkthrough example slot is handled gracefully (`[ -f ]` guard)
  </done>
</task>

</tasks>

<verification>
- `uv run python examples/first_graph.py` → SKIP or success, exit 0
- `uv run python examples/approval_demo.py` → SKIP or success, exit 0
- `uv run mkdocs build --strict` → green
- `uv run pytest tests/test_docs_phase30.py -v` → all tests green
</verification>

<success_criteria>
- DOCS-01 finalized: landing page Choose Your Path tabs point to the real tutorial sections
- DOCS-02 shipped: 3-section linear tutorial with runnable examples, <5 min first output, <30 min full
- Every non-trivial code block embedded via pymdownx.snippets (no drift risk)
- CI workflow runs all examples on every commit with SKIP-safe secret handling
</success_criteria>

<output>
After completion, create `.planning/phases/30-docs-site-foundation-getting-started-governance-walkthrough/30-03-SUMMARY.md` documenting: the chosen in-process bootstrap pattern for examples, the exact curl approval command shown in the docs, and any pitfalls hit while running the examples locally.
</output>
