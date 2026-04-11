---
phase: 30-docs-site-foundation-getting-started-governance-walkthrough
plan: 03
subsystem: docs
tags: [docs, tutorial, getting-started, examples, ci]
requires:
  - zeroth.core.examples.quickstart
  - zeroth.core.service.bootstrap
  - zeroth.core.service.approval_api
provides:
  - examples/first_graph.py (library-embedded Getting Started example)
  - examples/approval_demo.py (service-mode + approval Getting Started example)
  - .github/workflows/examples.yml (CI job that runs all examples on push/PR)
  - Filled Getting Started tutorial pages (01-install, 02-first-graph, 03-service-and-approval)
  - Finalized landing page Choose-Your-Path tabs
affects:
  - docs/index.md
  - docs/tutorials/getting-started/*.md
  - tests/test_docs_phase30.py
tech-stack:
  added: []
  patterns:
    - In-process bootstrap for examples uses run_migrations + AsyncSQLiteDatabase against a tempdir and passes a custom agent_runners dict + stub executable_unit_runner into the RuntimeOrchestrator so the tutorial skips the full AgentRuntime + ExecutableUnit wiring that production uses
    - Examples SKIP cleanly on missing OPENAI_API_KEY (exit 0 + stderr notice) so forked PR CI without repo secrets never fails
    - approval_demo.py runs the FastAPI app in-process via httpx.ASGITransport so the tutorial exercises the real /deployments/{ref}/approvals/{id}/resolve endpoint without requiring a separate uvicorn daemon
    - pymdownx.snippets with base_path [., examples] lets tutorial pages embed examples/*.py via bare filenames (e.g. --8<-- "first_graph.py"), and check_paths=true fails the strict build if a referenced file is missing
    - Guarded slot for examples/governance_walkthrough.py in examples.yml via shell [ -f ] check lets Plan 30-04 drop in the next example without editing the workflow
key-files:
  created:
    - examples/first_graph.py
    - examples/approval_demo.py
    - .github/workflows/examples.yml
  modified:
    - docs/index.md
    - docs/tutorials/getting-started/index.md
    - docs/tutorials/getting-started/01-install.md
    - docs/tutorials/getting-started/02-first-graph.md
    - docs/tutorials/getting-started/03-service-and-approval.md
    - tests/test_docs_phase30.py
key-decisions:
  - Use httpx.ASGITransport to boot the service in-process for approval_demo.py instead of spawning a real uvicorn subprocess — keeps the tutorial single-file, avoids port collisions, and exercises the same FastAPI app + lifespan the production entrypoint uses (approval_api.resolve is the exact same code path)
  - Pass a minimal custom AgentRunner (litellm-backed, ~15 LOC) and a stub ExecutableUnitRunner (echo) into bootstrap_service via agent_runners/executable_unit_runner overrides, instead of standing up the full zeroth.core.agent_runtime.AgentRunner + ExecutableUnitRegistry + NativeUnitManifest chain — the tutorial goal is to show the graph run surface, not teach the whole agent runtime
  - Disable the durable worker for both examples (enable_durable_worker=False) so the orchestrator runs synchronously on POST /runs; the production path uses worker=True driven by the FastAPI lifespan, but the tutorial needs immediate paused_for_approval feedback on the response
  - Fix the static API key `demo-operator-key` in approval_demo.py so the printed curl command is copy-pasteable without env var juggling; the docs call this out as a tutorial-only shortcut
  - Wire a DemoPayload BaseModel with a single `message: str = ""` field as both contract://demo-input and contract://demo-output so the empty-payload default satisfies the Run API's contract validator without forcing tutorial readers to think about contracts on step 1
requirements-completed:
  - DOCS-01
  - DOCS-02
duration: 4 min
completed: 2026-04-11
---

# Phase 30 Plan 03: Getting Started Tutorial Summary

Shipped the complete 3-section Getting Started tutorial — two new runnable example scripts (`examples/first_graph.py` for the library path, `examples/approval_demo.py` for the service + approval path), filled tutorial prose in `docs/tutorials/getting-started/01-install.md`, `02-first-graph.md`, and `03-service-and-approval.md` with `pymdownx.snippets` embeds of the example code, finalized the landing page Choose-Your-Path tabs to link into the right tutorial sections, and added a `.github/workflows/examples.yml` CI job that exercises every example on push/PR with SKIP-safe secret handling. Every non-trivial code block in sections 2 and 3 is an `--8<--` snippet embed, so docs and code cannot drift.

## What Shipped

### `examples/first_graph.py` (library-embedded path)

~160 LOC script that:

1. SKIPs cleanly on missing `OPENAI_API_KEY` (exit 0 + stderr notice).
2. Runs `run_migrations` against a tempdir SQLite DB, opens an `AsyncSQLiteDatabase`.
3. Registers `contract://demo-input` and `contract://demo-output` as a trivial `DemoPayload` BaseModel so the Run API contract validator passes with an empty default payload.
4. Persists the `zeroth.core.examples.quickstart.build_demo_graph()` graph, deploys it as `demo-first-graph`, and calls `bootstrap_service(...)` with a custom echo `ExecutableUnitRunner` and `enable_durable_worker=False`.
5. Replaces `orchestrator.agent_runners` with a single-node dict pointing `"agent"` at a minimal `_LiteLLMAgentRunner` (~15 LOC) that calls `litellm.completion()` once and returns an `output_data` dict.
6. Drives the graph via `orchestrator.run_graph(...)` and prints the final run status + `final_output`.

Library-embedded means no HTTP hop — the whole run lives inside one Python process.

### `examples/approval_demo.py` (service + approval path)

~195 LOC script that:

1. SKIPs cleanly on missing `OPENAI_API_KEY`.
2. Bootstraps the same in-process SQLite + contracts + graph as `first_graph.py`, but uses `build_demo_graph(include_approval=True)` so the graph is agent → HumanApprovalNode → tool.
3. Configures a single-key `ServiceAuthConfig` (`demo-operator-key`, OPERATOR + REVIEWER roles) so the auth layer approves both `/runs` POSTs and `/approvals/*/resolve` POSTs with one header.
4. Wraps the bootstrapped service in `create_app(service)`, mounts it on an `httpx.AsyncClient(transport=httpx.ASGITransport(app=app), base_url="http://test")`, and drives the full HTTP path:
   - `POST /runs` with `{input_payload: {message: "..."}}` → agent runs synchronously → response shows `status: paused_for_approval` with the `approval_id` in `approval_paused_state`.
   - Prints the **exact** `curl -X POST http://localhost:8000/deployments/{ref}/approvals/{approval_id}/resolve -H "X-API-Key: demo-operator-key" -d '{"decision":"approve"}'` command a human operator would paste into their terminal against a real uvicorn daemon.
   - `POST /deployments/{ref}/approvals/{approval_id}/resolve` with `{"decision":"approve"}` → response contains the final succeeded run.
5. Prints the final run status and terminal output.

### Tutorial pages (`docs/tutorials/getting-started/`)

- **`index.md`** — time budget (<5 min first output / <30 min end-to-end), prose explaining library vs service split, numbered list linking to sections 1-3.
- **`01-install.md`** — `pip install zeroth-core` and `uv add zeroth-core` install paths, API key setup, `--8<-- "hello.py"` snippet embed, expected output, explicit <5 minute gate callout, link to the pyproject extras doc.
- **`02-first-graph.md`** — one-paragraph explanation of graphs/agents/tool nodes, callout admonition marking `zeroth.core.examples.quickstart` as a tutorial helper (not a stable API), `--8<-- "first_graph.py"` snippet embed, expected `Run <uuid> finished with status: completed` output, step-by-step "what just happened" breakdown covering `run_migrations`, `ContractRegistry`, `GraphRepository`, `DeploymentService`, `bootstrap_service`, and `orchestrator.run_graph`.
- **`03-service-and-approval.md`** — explains the `uv run python -m zeroth.core.service.entrypoint` production boot command, clarifies that the tutorial uses `httpx.ASGITransport` in-process but exercises the same code path below the transport boundary, embeds `approval_demo.py` via snippets, shows the copy-pasteable curl command (hard-coded in prose — the same string the example prints verbatim), and ends with a library-vs-service comparison table plus a link forward to the Governance Walkthrough (Plan 30-04).

### Landing page (`docs/index.md`)

Replaced the plan-02 placeholder prose inside the Choose-Your-Path tabbed split with real blurbs:

- **Embed as library** → links to `tutorials/getting-started/02-first-graph.md`
- **Run as service** → links to `tutorials/getting-started/03-service-and-approval.md`

Also added a top-level "New here?" link to `tutorials/getting-started/index.md`. The `--8<-- "hello.py"` snippet embed from plan 02 is preserved.

### `.github/workflows/examples.yml`

Workflow `name: examples`, triggers `push.branches: [main]` and `pull_request.branches: [main]`. Single `ubuntu-latest` job that:

1. Checks out the repo.
2. Runs `astral-sh/setup-uv@v5` + `actions/setup-python@v5` (3.12).
3. `uv sync --all-extras --group dev`.
4. Runs each example script in sequence: `hello.py` → `first_graph.py` → `approval_demo.py` → `governance_walkthrough.py` (the last guarded by `[ -f ]` so Plan 30-04 can land it without modifying this workflow).

`OPENAI_API_KEY` and `ANTHROPIC_API_KEY` are exposed at the job level via `env:` from repo secrets; on forked PRs those secrets are absent and each script's SKIP guard returns exit 0, keeping the job green.

### Shape tests (`tests/test_docs_phase30.py`)

Five new Plan 30-03 assertions appended to the existing Plan 30-02 shape tests:

1. `test_examples_workflow_exists` — loads `.github/workflows/examples.yml` as YAML and asserts `push.branches` and `pull_request.branches` both include `main`. (PyYAML parses bare `on:` as the Python `True` bool, so the test accepts both `"on"` and `True` as the key.)
2. `test_examples_workflow_runs_first_graph_and_approval` — asserts the workflow body contains `examples/first_graph.py` AND `examples/approval_demo.py` as literal path strings.
3. `test_first_graph_page_embeds_example` — asserts `02-first-graph.md` contains `--8<--` and `first_graph.py`.
4. `test_approval_page_embeds_example_and_curl` — asserts `03-service-and-approval.md` contains `--8<--`, `approval_demo.py`, and both `/approvals/` and `/resolve` substrings (proving the curl command is documented).
5. `test_landing_tabs_link_to_getting_started` — asserts `docs/index.md` contains links to both `tutorials/getting-started/02-first-graph.md` and `tutorials/getting-started/03-service-and-approval.md`.

## Tasks & Commits

| Task | Name                                                              | Commit    | Files                                                                                                                                                                                                       |
| ---- | ----------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1    | Write examples/first_graph.py and examples/approval_demo.py       | `ef9d5b3` | `examples/first_graph.py`, `examples/approval_demo.py`                                                                                                                                                      |
| 2    | Getting Started tutorial pages + landing page finalization        | `68c1088` | `docs/index.md`, `docs/tutorials/getting-started/index.md`, `docs/tutorials/getting-started/01-install.md`, `docs/tutorials/getting-started/02-first-graph.md`, `docs/tutorials/getting-started/03-service-and-approval.md` |
| 3    | Examples CI workflow + Plan 30-03 shape tests                      | `a1bbff2` | `.github/workflows/examples.yml`, `tests/test_docs_phase30.py`                                                                                                                                              |

## Verification Results

- `OPENAI_API_KEY= uv run python examples/first_graph.py` → `SKIP: set OPENAI_API_KEY ...`, **exit 0**.
- `OPENAI_API_KEY= uv run python examples/approval_demo.py` → `SKIP: set OPENAI_API_KEY ...`, **exit 0**.
- `uv run ruff check examples/first_graph.py examples/approval_demo.py` → **All checks passed**.
- `uv run ruff format examples/first_graph.py examples/approval_demo.py` → 2 files reformatted (whitespace/line-wrap only, no logic change).
- `uv run mkdocs build --strict` → **exit 0**, "Documentation built in 0.24 seconds", no warnings. (The red "MkDocs 2.0" notice on stderr is the upstream Material-for-MkDocs team's informational banner, not a strict-mode failure.)
- `uv run pytest tests/test_docs_phase30.py -v` → **13 passed** (1 plan-01 scaffold + 7 plan-02 shape tests + 5 new plan-03 shape tests).
- `uv run ruff check tests/test_docs_phase30.py` → **All checks passed**.

The happy-path (with `OPENAI_API_KEY` set) was not exercised in the local session — per the plan's guidance, the real-LLM test runs in the `examples` CI workflow on `main` using the repo secret. Every piece of the happy-path code was exercised via ruff's static checks and a by-line walkthrough of the orchestrator dispatch surface (`RuntimeOrchestrator.run_graph` → `_drive` → `_dispatch_node` → custom `_LiteLLMAgentRunner.run` / `_EchoExecutableUnitRunner.run`).

## Success Criteria

- [x] DOCS-01 finalized: landing page Choose-Your-Path tabs link to real tutorial sections (02-first-graph.md and 03-service-and-approval.md).
- [x] DOCS-02 shipped: 3-section linear tutorial with runnable examples, <5 min first output (section 1), <30 min full (sections 1-3).
- [x] Every non-trivial code block in sections 2 and 3 is a `pymdownx.snippets` embed, not inlined.
- [x] CI workflow runs all examples (`hello.py`, `first_graph.py`, `approval_demo.py`, guarded `governance_walkthrough.py`) on every push/PR to main with SKIP-safe secret handling.
- [x] Both example scripts import only from the public `zeroth.core.*` surface.
- [x] `approval_demo.py` POSTs to the real `/deployments/{ref}/approvals/{id}/resolve` endpoint via `httpx.ASGITransport`.
- [x] `uv run mkdocs build --strict` is green.
- [x] `uv run pytest tests/test_docs_phase30.py -v` → 13 passed.
- [x] Ruff clean on all new/modified Python files.

## Deviations from Plan

None — plan executed exactly as written. Ruff's auto-formatter collapsed a few multi-line function signatures in the example scripts onto single lines, but this is cosmetic and counted as formatter output rather than a deviation.

## Chosen Approaches (as requested in plan output section)

### In-process bootstrap pattern for examples

Both example scripts follow the same bootstrap pattern:

```python
with tempfile.TemporaryDirectory() as tmp:
    db_path = str(Path(tmp) / "example.sqlite")
    run_migrations(f"sqlite:///{db_path}")
    database = AsyncSQLiteDatabase(path=db_path)

    contract_registry = ContractRegistry(database)
    await contract_registry.register(DemoPayload, name="contract://demo-input")
    await contract_registry.register(DemoPayload, name="contract://demo-output")

    graph = await GraphRepository(database).create(build_demo_graph(...))
    await GraphRepository(database).publish(graph.graph_id, graph.version)
    deployment = await deployment_service.deploy(deployment_ref, graph.graph_id, graph.version)

    service = await bootstrap_service(
        database,
        deployment_ref=deployment.deployment_ref,
        executable_unit_runner=_EchoExecutableUnitRunner(),
        enable_durable_worker=False,
    )
    service.orchestrator.agent_runners = {"agent": _LiteLLMAgentRunner(...)}
```

Key choices:

1. **Tempdir SQLite** — avoids polluting CWD, auto-cleans.
2. **`run_migrations` first** — matches the entrypoint.py production boot path.
3. **`enable_durable_worker=False`** — so `POST /runs` drives the orchestrator synchronously in the request handler, giving the tutorial an immediate `paused_for_approval` response instead of requiring a poll loop.
4. **Custom minimal agent + EU runners** — sidesteps the full `zeroth.core.agent_runtime` + `ExecutableUnitRegistry` + `NativeUnitManifest` chain. The orchestrator only requires objects that expose `async def run(...)` returning something with `output_data` and `audit_record` attributes.
5. **DemoPayload BaseModel** with `message: str = ""` default — lets the Run API contract validator accept empty payloads on first run.

### Exact curl approval command shown in the docs

Both the example script (via `_print_curl`) and the tutorial prose (hard-coded) show:

```bash
curl -X POST http://localhost:8000/deployments/demo-approval/approvals/$APPROVAL_ID/resolve \
     -H "X-API-Key: demo-operator-key" \
     -H "Content-Type: application/json" \
     -d '{"decision": "approve"}'
```

The deployment ref (`demo-approval`) is fixed in `approval_demo.py` so the printed command and the docs match verbatim. The reader is expected to substitute `$APPROVAL_ID` from the script's first stdout line when they run against a real uvicorn daemon.

### Pitfalls hit while building the examples

1. **`orchestrator.run_graph` vs manual `Run` construction** — first pass tried to hand-build a `Run` object and call a non-existent `run_until_terminal` method. Replaced with the canonical `orchestrator.run_graph(graph, initial_input, deployment_ref=...)` entry point which handles `Run` creation, persistence, and driving in one call.
2. **Ruff E501 on curl line** — the full curl URL f-string exceeded 100 chars. Refactored to build `url` in a local variable first, then interpolate.
3. **PyYAML `on:` key** — the YAML bare `on:` key is parsed as the Python `True` bool, not the string `"on"`. Added `triggers = config.get("on") if "on" in config else config.get(True)` fallback in `test_examples_workflow_exists`.
4. **`ExecutableUnitRunner` shape** — the orchestrator calls `runner.run(manifest_ref, input_payload)` and expects an object with `.output_data` and `.audit_record` — not a full `ExecutableUnitRunner` instance. A 5-line `SimpleNamespace`-returning echo stub is sufficient.
5. **Auth roles** — `/runs` requires `OPERATOR` (RUN_CREATE) and `/approvals/{id}/resolve` requires `REVIEWER` (APPROVAL_RESOLVE). The demo key is granted both roles so a single `X-API-Key` header works for the whole script.

## Ready for Next Plan

Plan 30-04 (Governance Walkthrough) can:

1. Drop `examples/governance_walkthrough.py` into the `examples/` directory — the guarded `[ -f ]` slot in `.github/workflows/examples.yml` will pick it up automatically without workflow edits.
2. Reuse the in-process bootstrap pattern documented above and the `_LiteLLMAgentRunner` / `_EchoExecutableUnitRunner` shapes.
3. Use `build_demo_graph_with_policy(denied_capabilities=[...])` from `zeroth.core.examples.quickstart` (already shipped in Plan 30-01) to demonstrate the policy-block scenario.
4. Append shape tests for the governance walkthrough page to `tests/test_docs_phase30.py`.
5. Rely on `uv run mkdocs build --strict` as the CI gate (full CI deploy wiring lands in Plan 30-05).

## Self-Check: PASSED

- [x] `examples/first_graph.py` exists on disk
- [x] `examples/approval_demo.py` exists on disk
- [x] `.github/workflows/examples.yml` exists on disk
- [x] `docs/index.md` modified (tabs now link to real pages)
- [x] `docs/tutorials/getting-started/01-install.md` has real prose + hello.py snippet embed
- [x] `docs/tutorials/getting-started/02-first-graph.md` has real prose + first_graph.py snippet embed
- [x] `docs/tutorials/getting-started/03-service-and-approval.md` has real prose + approval_demo.py snippet embed + curl command
- [x] `tests/test_docs_phase30.py` contains 5 new Plan 30-03 assertions (13 total tests)
- [x] Commit `ef9d5b3` present in git log (feat(30-03): add first_graph.py and approval_demo.py ...)
- [x] Commit `68c1088` present in git log (docs(30-03): fill Getting Started tutorial pages ...)
- [x] Commit `a1bbff2` present in git log (ci(30-03): add examples workflow + Plan 30-03 shape tests)
- [x] `uv run pytest tests/test_docs_phase30.py -v` → 13 passed
- [x] `uv run mkdocs build --strict` → exit 0
- [x] `OPENAI_API_KEY= uv run python examples/first_graph.py` → exit 0 (SKIP)
- [x] `OPENAI_API_KEY= uv run python examples/approval_demo.py` → exit 0 (SKIP)

## Threat Flags

None — the tutorial examples and CI workflow do not introduce new network endpoints, auth paths, or trust boundaries beyond what Plan 30-01 and Plan 30-02 already cover. The `_DEMO_API_KEY` constant in `approval_demo.py` is a local in-process credential that never leaves the script's httpx transport and is explicitly marked `# noqa: S105` as a tutorial constant.
