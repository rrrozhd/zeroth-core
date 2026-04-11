---
phase: 32-reference-docs-deployment-migration-guide
plan: 04
type: execute
wave: 2
depends_on:
  - "32-01-python-api-reference-mkdocstrings-PLAN"
  - "32-02-http-api-reference-swagger-PLAN"
  - "32-03-configuration-reference-dump-config-PLAN"
files_modified:
  - docs/how-to/deployment/index.md
  - docs/how-to/deployment/local-dev.md
  - docs/how-to/deployment/docker-compose.md
  - docs/how-to/deployment/standalone-service.md
  - docs/how-to/deployment/embedded-library.md
  - docs/how-to/deployment/with-regulus.md
autonomous: true
requirements:
  - DOCS-10
must_haves:
  truths:
    - "A 'Deployment' section under How-to Guides contains 5 mode-specific pages + an index"
    - "Each mode page has a runnable command block and prose covering prerequisites, install/run, verification, and common gotchas"
    - "The 5 modes covered are: local-dev, docker-compose, standalone-service, embedded-library, with-regulus"
    - "Each page cross-links to the Configuration Reference (from Plan 32-03) and HTTP API Reference (from Plan 32-02) where relevant"
  artifacts:
    - path: "docs/how-to/deployment/index.md"
      provides: "Deployment section landing page with mode comparison"
    - path: "docs/how-to/deployment/local-dev.md"
      provides: "Local dev mode (uv run zeroth-core serve)"
    - path: "docs/how-to/deployment/docker-compose.md"
      provides: "docker-compose mode using repo docker-compose.yml"
    - path: "docs/how-to/deployment/standalone-service.md"
      provides: "Standalone service mode (uvicorn + reverse proxy)"
    - path: "docs/how-to/deployment/embedded-library.md"
      provides: "Embedded-in-host-app mode"
    - path: "docs/how-to/deployment/with-regulus.md"
      provides: "Deployment alongside Regulus companion service"
  key_links:
    - from: "docs/how-to/deployment/index.md"
      to: "docs/how-to/deployment/local-dev.md"
      via: "relative markdown link"
      pattern: "local-dev.md"
    - from: "docs/how-to/deployment/docker-compose.md"
      to: "docker-compose.yml (repo root)"
      via: "snippet include via pymdownx.snippets"
      pattern: "docker-compose"
---

<objective>
Write the Deployment Guide: one landing page + 5 mode-specific pages under `docs/how-to/deployment/`, each ~400 words with runnable command blocks and verification steps. Covers local-dev, docker-compose, standalone-service, embedded-library, and with-regulus modes.

Purpose: Closes DOCS-10. Users shipping zeroth-core have an end-to-end playbook for every supported deployment model.

Output: 6 new markdown files under `docs/how-to/deployment/`. Nav wiring deferred to Plan 32-06 (finalize).
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-CONTEXT.md

@docker-compose.yml
@README.md
@pyproject.toml
@docs/how-to/service.md

<interfaces>
Established conventions from Phase 30/31 docs:
- Pages live under `docs/how-to/<area>/<slug>.md`
- Code fences use triple backticks with language tags (`bash`, `python`, `yaml`)
- Configuration env vars are referenced by their generated anchor: `../../reference/configuration.md#<section>`
- HTTP API cross-link: `../../reference/http-api.md`
- Python API cross-link: `../../reference/python-api/<subsystem>.md`

Install extras from README.md:
```
pip install zeroth-core
pip install "zeroth-core[memory-pg]"
pip install "zeroth-core[memory-chroma]"
pip install "zeroth-core[memory-es]"
pip install "zeroth-core[dispatch]"
pip install "zeroth-core[sandbox]"
pip install "zeroth-core[all]"
```

Service entry point (from pyproject.toml + src/zeroth/core/service/entrypoint.py): `zeroth-core serve` (console script) or `uv run zeroth-core serve`.

docker-compose.yml is 73 lines — reference it via `pymdownx.snippets` (already enabled in mkdocs.yml with `base_path: [".", "examples"]`) or quote salient excerpts inline. Prefer an inline excerpt (snippets base_path does not include repo root).
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write deployment index + local-dev + docker-compose pages</name>
  <files>docs/how-to/deployment/index.md, docs/how-to/deployment/local-dev.md, docs/how-to/deployment/docker-compose.md</files>
  <action>
    Per D-04:

    1. **`docs/how-to/deployment/index.md`** — landing page (~250 words):
       - Heading: `# Deployment`
       - Intro paragraph: zeroth-core supports five deployment modes, from a single local process to a multi-container service with Regulus economics.
       - A comparison table with columns: Mode | When to use | Command | Prerequisites.
         - local-dev: "Hacking on graphs, tutorials" / `uv run zeroth-core serve` / Python 3.12
         - docker-compose: "Full-stack local with Postgres/Redis" / `docker compose up` / Docker
         - standalone-service: "Production single-node" / `uvicorn zeroth.core.service.entrypoint:app` / Postgres, TLS cert
         - embedded-library: "Importing zeroth.core in your own FastAPI/CLI" / `from zeroth.core import ...` / N/A
         - with-regulus: "Budget enforcement + economics" / Compose with regulus service / Regulus container
       - Cross-links to each mode page.
       - Cross-links to [Configuration Reference](../../reference/configuration.md) for env var tuning.

    2. **`docs/how-to/deployment/local-dev.md`** (~400 words):
       - `# Local development`
       - Use case: hacking on graphs, running tutorials, exercising examples.
       - Prerequisites: Python 3.12, `uv` installed.
       - Install block:
         ```bash
         pip install zeroth-core
         # Or, with optional extras:
         pip install "zeroth-core[memory-pg,dispatch]"
         ```
       - Run block:
         ```bash
         zeroth-core serve
         # Or via uv from a checkout:
         uv sync
         uv run zeroth-core serve
         ```
       - Verification: `curl http://localhost:8000/healthz` returns 200.
       - Default storage: SQLite at `./zeroth.db` (see `ZEROTH_DATABASE__SQLITE_PATH`).
       - Common gotchas:
         - Port conflict on 8000 → override via `--port` or `ZEROTH_SERVICE__PORT`.
         - Missing LLM keys → set `OPENAI_API_KEY` etc. in `.env`.
       - Cross-link: [Configuration Reference — database](../../reference/configuration.md#database) and [HTTP API Reference](../../reference/http-api.md).

    3. **`docs/how-to/deployment/docker-compose.md`** (~400 words):
       - `# Docker Compose`
       - Use case: full-stack local with Postgres + Redis + optional Regulus.
       - Prerequisites: Docker 24+, Docker Compose v2.
       - Step 1 — clone the repo (for the bundled compose file) or copy `docker-compose.yml` from [the repository root](https://github.com/rrrozhd/zeroth-core/blob/main/docker-compose.yml).
       - Step 2 — start services:
         ```bash
         docker compose up -d
         docker compose logs -f zeroth-core
         ```
       - Step 3 — verify: `curl http://localhost:8000/healthz` and `docker compose ps`.
       - Environment: the compose file wires `ZEROTH_DATABASE__POSTGRES_DSN` to the bundled Postgres; override any setting via a `.env` file in the same directory (see [Configuration Reference](../../reference/configuration.md)).
       - Teardown: `docker compose down -v` to drop volumes.
       - Common gotchas:
         - First boot: run Alembic migrations inside the container with `docker compose exec zeroth-core uv run alembic upgrade head`.
         - Stale image: rebuild with `docker compose build --no-cache zeroth-core`.
       - Cross-link: [with-regulus mode](with-regulus.md) for adding the Regulus companion.
  </action>
  <verify>
    <automated>test -f docs/how-to/deployment/index.md && test -f docs/how-to/deployment/local-dev.md && test -f docs/how-to/deployment/docker-compose.md && wc -w docs/how-to/deployment/local-dev.md docs/how-to/deployment/docker-compose.md</automated>
  </verify>
  <done>Three pages exist, each has runnable command blocks, cross-links to Configuration Reference, and is roughly the target word count (300-500).</done>
</task>

<task type="auto">
  <name>Task 2: Write standalone-service + embedded-library + with-regulus pages</name>
  <files>docs/how-to/deployment/standalone-service.md, docs/how-to/deployment/embedded-library.md, docs/how-to/deployment/with-regulus.md</files>
  <action>
    Per D-04:

    1. **`docs/how-to/deployment/standalone-service.md`** (~400 words):
       - `# Standalone service`
       - Use case: production single-node deploy fronted by nginx/Caddy.
       - Prerequisites: Python 3.12, Postgres, optional Redis, TLS termination.
       - Install:
         ```bash
         pip install "zeroth-core[memory-pg,dispatch]"
         ```
       - Run via uvicorn directly (for process managers / systemd):
         ```bash
         uvicorn zeroth.core.service.entrypoint:app \
           --host 0.0.0.0 \
           --port 8000 \
           --workers 4
         ```
       - Env config (excerpt — see [full reference](../../reference/configuration.md)):
         ```bash
         ZEROTH_DATABASE__POSTGRES_DSN=postgresql://zeroth:secret@db:5432/zeroth
         ZEROTH_AUTH__API_KEYS_JSON='{"ops":"..."}'
         ZEROTH_DISPATCH__ARQ_ENABLED=true
         ```
       - Reverse proxy: minimal nginx snippet proxying `/` to `127.0.0.1:8000`, terminating TLS. systemd unit example:
         ```ini
         [Service]
         ExecStart=/usr/local/bin/uvicorn zeroth.core.service.entrypoint:app --host 127.0.0.1 --port 8000 --workers 4
         EnvironmentFile=/etc/zeroth/zeroth.env
         Restart=on-failure
         ```
       - Verification: `curl -I https://api.example.com/healthz`.
       - Gotchas: run `alembic upgrade head` before first start; ensure the process can read `ZEROTH_DATABASE__ENCRYPTION_KEY` for secret storage.

    2. **`docs/how-to/deployment/embedded-library.md`** (~400 words):
       - `# Embedded in a host application`
       - Use case: you already run a FastAPI/CLI/worker and want to embed the zeroth-core runtime in-process (no separate service).
       - Prerequisites: `pip install zeroth-core` in the host project's virtualenv.
       - Minimal pattern:
         ```python
         from zeroth.core.orchestrator import Orchestrator
         from zeroth.core.graph import Graph
         from zeroth.core.service.bootstrap import build_bootstrap

         bootstrap = build_bootstrap()  # reads ZEROTH_* env vars
         graph = Graph(...)
         orch = bootstrap.orchestrator
         result = await orch.run(graph, inputs={"query": "hello"})
         ```
       - FastAPI host app integration: `app.include_router(zeroth_router)` if the host wants to expose HTTP endpoints, otherwise just import the Python surface.
       - No separate process needed; no migrations required for SQLite default; run `alembic upgrade head` from the host project if using Postgres.
       - Cross-link: [Python API Reference — service bootstrap](../../reference/python-api/service.md) and [Orchestrator](../../reference/python-api/orchestrator.md).
       - Gotchas:
         - Two event loops: call from inside an async context.
         - Logging: zeroth-core uses stdlib logging; configure via the host's root logger.

    3. **`docs/how-to/deployment/with-regulus.md`** (~400 words):
       - `# With the Regulus companion service`
       - Use case: cost-budget enforcement via Regulus (economics SDK published as `econ-instrumentation-sdk`).
       - Prerequisites: running Regulus service reachable over HTTP.
       - Install (Regulus is already a transitive dep of zeroth-core ≥0.1.1):
         ```bash
         pip install zeroth-core
         ```
       - Env config:
         ```bash
         ZEROTH_REGULUS__ENABLED=true
         ZEROTH_REGULUS__BASE_URL=http://regulus:9000
         ZEROTH_REGULUS__API_KEY=...
         ```
         Refer to [Configuration Reference — regulus](../../reference/configuration.md#regulus) for the full section.
       - docker-compose excerpt adding a `regulus` service alongside `zeroth-core` (show ~15 lines of yaml).
       - Verification: run a graph with a cost-capped contract and confirm the orchestrator halts when the budget is exceeded (link to [budget-cap cookbook recipe](../cookbook/budget-cap.md)).
       - Gotchas:
         - If Regulus is unreachable, `ZEROTH_REGULUS__ENABLED=true` fails closed on cost-checked nodes.
         - Verify SDK version pin: `econ-instrumentation-sdk>=0.1.1` in `pyproject.toml`.
       - Cross-link: [Economics concept page](../../concepts/econ.md) and [Python API Reference — econ](../../reference/python-api/econ.md).
  </action>
  <verify>
    <automated>test -f docs/how-to/deployment/standalone-service.md && test -f docs/how-to/deployment/embedded-library.md && test -f docs/how-to/deployment/with-regulus.md && grep -l "reference/configuration" docs/how-to/deployment/*.md | wc -l</automated>
  </verify>
  <done>All three pages exist, each cross-links to Configuration Reference at least once, code blocks are runnable, and content is roughly 300-500 words each.</done>
</task>

</tasks>

<verification>
- `ls docs/how-to/deployment/*.md | wc -l` == 6
- Every page has at least one fenced code block
- Every page contains a link to `../../reference/configuration.md` or `../../reference/python-api/`
- `uv run mkdocs build` (non-strict, since nav wiring lands in Plan 32-06) succeeds without errors
</verification>

<success_criteria>
DOCS-10 satisfied: Deployment Guide covers local development, docker-compose, standalone service mode, embedded-in-host-app mode, and deployments with/without the Regulus companion service. (Nav wiring finalized in Plan 32-06.)
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-04-SUMMARY.md`
</output>
