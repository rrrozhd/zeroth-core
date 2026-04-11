---
phase: 32-reference-docs-deployment-migration-guide
plan: 05
type: execute
wave: 2
depends_on: []
files_modified:
  - docs/how-to/migration-from-monolith.md
autonomous: true
requirements:
  - DOCS-11
must_haves:
  truths:
    - "docs/how-to/migration-from-monolith.md walks an existing monolith user through the switch to zeroth.core.*"
    - "The guide covers all four topics: import rename, econ SDK path swap, env var changes, Docker image retag"
    - "The guide contains at least one concrete before/after code example per topic"
    - "An automated grep+sed recipe for the import rename is included and runnable on macOS and Linux"
  artifacts:
    - path: "docs/how-to/migration-from-monolith.md"
      provides: "Single comprehensive migration guide"
      contains: "from zeroth.core"
  key_links:
    - from: "docs/how-to/migration-from-monolith.md"
      to: "docs/how-to/deployment/docker-compose.md"
      via: "relative link in Docker retag section"
      pattern: "deployment/docker-compose"
---

<objective>
Write a single comprehensive migration guide at `docs/how-to/migration-from-monolith.md` covering every change needed to move an existing project from the pre-split monolithic `zeroth.*` layout to the published `zeroth-core` package under `zeroth.core.*`.

Purpose: Closes DOCS-11. Existing monolith users have a concrete, copy-pasteable upgrade path.

Output: One new markdown file, ~1200-1500 words, with before/after examples for each topic. Nav wiring deferred to Plan 32-06.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-CONTEXT.md

@pyproject.toml
@README.md

<interfaces>
Key facts to cite in the guide:

- Published package name: `zeroth-core` (on PyPI ≥ 0.1.1 from Phase 28)
- Namespace: PEP 420 package `zeroth.core.*` — no top-level `zeroth/__init__.py`
- Monolith was at `zeroth.*` (e.g., `from zeroth.orchestrator import Orchestrator`)
- econ SDK is now `econ-instrumentation-sdk>=0.1.1` from PyPI (not a local path dep)
- Namespace change covered in Phase 27; PyPI publishing covered in Phase 28
- Future `FUTURE-01` tracks a LibCST codemod — not yet shipped, so this guide uses a grep+sed recipe
- Env vars: the prefix is `ZEROTH_` in both monolith and zeroth.core layouts — there are no renamed env vars in the rename phase; the guide should document this explicitly to prevent confusion
- Docker image: monolith image path should be documented as "retag to match your new registry/tag for zeroth-core" — the guide notes that zeroth-core does not currently publish an official image, so users rebuild from their own Dockerfile or the repo's `docker-compose.yml`
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Write docs/how-to/migration-from-monolith.md</name>
  <files>docs/how-to/migration-from-monolith.md</files>
  <action>
    Per D-05:

    Create a single comprehensive page. Structure:

    ```markdown
    # Migration from the monolith layout

    If you have a codebase that imports from the pre-split monolithic `zeroth.*` namespace, this guide walks you through the one-time upgrade to the published `zeroth-core` package under `zeroth.core.*`. The change is a pure rename — no runtime semantics change, no API breaks.

    The rename was introduced in Phase 27 of the v3.0 milestone and formalized on PyPI in Phase 28.

    ## TL;DR

    1. `pip install "zeroth-core>=0.1.1"` (drop any local/path dependency on `zeroth`)
    2. Rewrite imports: `from zeroth.X` → `from zeroth.core.X`
    3. Drop any local path dependency on `econ-instrumentation-sdk` — it's now a transitive PyPI dep of `zeroth-core`
    4. Keep all `ZEROTH_*` env vars as-is — no renames
    5. Rebuild your Docker image against the new package name

    Most small projects complete this migration in under 10 minutes.

    ## 1. Install the published package

    **Before** (monolith, path-installed):
    ```bash
    pip install -e /path/to/zeroth-monolith
    ```

    **After** (PyPI):
    ```bash
    pip install "zeroth-core>=0.1.1"
    # Or with extras matching your backend:
    pip install "zeroth-core[memory-pg,dispatch]>=0.1.1"
    ```

    If your project pins zeroth in `pyproject.toml`, change:

    ```toml
    # Before
    dependencies = [
      "zeroth @ file:///path/to/zeroth-monolith",
    ]

    # After
    dependencies = [
      "zeroth-core>=0.1.1",
    ]
    ```

    Note the distribution name is `zeroth-core` but you import it as `zeroth.core.*` (PEP 420 namespace package).

    ## 2. Rewrite imports

    Every import from `zeroth.<subsystem>` becomes `zeroth.core.<subsystem>`.

    **Before:**
    ```python
    from zeroth.orchestrator import Orchestrator
    from zeroth.graph import Graph, Node
    from zeroth.memory import EphemeralMemory
    import zeroth.policy as policy
    ```

    **After:**
    ```python
    from zeroth.core.orchestrator import Orchestrator
    from zeroth.core.graph import Graph, Node
    from zeroth.core.memory import EphemeralMemory
    import zeroth.core.policy as policy
    ```

    ### Grep + sed recipe

    For a quick in-place rename across a project, the following works on macOS (BSD sed) and Linux (GNU sed — remove the empty `''` after `-i`):

    ```bash
    # macOS
    grep -rl "from zeroth\." src/ tests/ | xargs sed -i '' 's/from zeroth\./from zeroth.core./g'
    grep -rl "import zeroth\." src/ tests/ | xargs sed -i '' 's/import zeroth\./import zeroth.core./g'

    # Linux
    grep -rl "from zeroth\." src/ tests/ | xargs sed -i 's/from zeroth\./from zeroth.core./g'
    grep -rl "import zeroth\." src/ tests/ | xargs sed -i 's/import zeroth\./import zeroth.core./g'
    ```

    **Caveat:** this is a naive substitution. It will correctly rewrite `from zeroth.graph` and `import zeroth.runtime`, but will also match false positives like string literals (`"zeroth.something"`). Diff before committing.

    A future release will ship a LibCST-based codemod (`python -m zeroth.core.codemods.rename_from_monolith`) that handles edge cases correctly — tracked as `FUTURE-01`.

    ### Verify the rewrite

    ```bash
    # Should return zero matches after the rewrite.
    grep -rn "^from zeroth\." src/ tests/
    grep -rn "^import zeroth\." src/ tests/

    # Run your test suite.
    uv run pytest
    ```

    ## 3. econ SDK path swap

    The monolith depended on `econ-instrumentation-sdk` via a local file path. Zeroth-core pins it as a published PyPI dependency:

    ```toml
    # In zeroth-core's pyproject.toml (transitive — you don't need to declare it)
    "econ-instrumentation-sdk>=0.1.1",
    ```

    **Action:** remove any local path dep from your own `pyproject.toml`:

    ```toml
    # Before
    dependencies = [
      "econ-instrumentation-sdk @ file:///path/to/regulus/sdk",
      "zeroth @ file:///path/to/zeroth-monolith",
    ]

    # After
    dependencies = [
      "zeroth-core>=0.1.1",
    ]
    ```

    `zeroth-core` will install `econ-instrumentation-sdk` automatically. If you need to override the version for testing, pin it explicitly:

    ```toml
    dependencies = [
      "zeroth-core>=0.1.1",
      "econ-instrumentation-sdk==0.1.1",
    ]
    ```

    ## 4. Environment variables

    **No env vars were renamed during the rename phase.** All `ZEROTH_*` settings keep their names, their nesting conventions (`ZEROTH_<SECTION>__<FIELD>`), and their defaults. This includes:

    - `ZEROTH_DATABASE__POSTGRES_DSN`
    - `ZEROTH_REDIS__*`
    - `ZEROTH_REGULUS__*`
    - `ZEROTH_AUTH__API_KEYS_JSON`
    - etc.

    See the full [Configuration Reference](../reference/configuration.md) for every supported variable.

    If your deployment sets `ZEROTH_*` vars via `.env`, docker-compose, or a systemd unit file, **no changes are required**.

    ## 5. Docker image retag

    Zeroth-core does not currently publish an official Docker image — you build your own from the package, or use the `docker-compose.yml` in the `zeroth-core` repo as a reference.

    **If you had a Dockerfile for the monolith** that installed it in editable mode, replace the install step:

    ```dockerfile
    # Before
    COPY zeroth-monolith /src/zeroth-monolith
    RUN pip install -e /src/zeroth-monolith

    # After
    RUN pip install "zeroth-core[memory-pg,dispatch]>=0.1.1"
    ```

    **Retag** your image (the tag is arbitrary — pick one that matches your registry layout):

    ```bash
    docker build -t registry.example.com/myorg/myapp:zeroth-core-0.1.1 .
    docker push registry.example.com/myorg/myapp:zeroth-core-0.1.1
    ```

    For a ready-made compose file, see [docker-compose deployment](deployment/docker-compose.md).

    ## 6. Verify the migration

    Run your existing test suite. Because the rename is pure (Phase 27 guaranteed zero functional changes), all passing tests on the monolith should still pass on `zeroth-core` without edits:

    ```bash
    uv run pytest
    ```

    If a test fails with an `ImportError` mentioning `zeroth.<something>` (without `.core`), the rename missed that file — rerun the sed recipe or fix manually.

    ## Troubleshooting

    - **`ModuleNotFoundError: No module named 'zeroth'`** after install — PEP 420 namespace package: make sure nothing in your project creates a `zeroth/__init__.py` that would shadow the namespace.
    - **`ModuleNotFoundError: No module named 'zeroth.orchestrator'`** — rename missed; check for imports in `.pyi` stub files, `conftest.py`, and any YAML/TOML config referencing dotted module paths.
    - **Duplicate `econ-instrumentation-sdk` install** — remove the local path dep from your `pyproject.toml`; the PyPI version from `zeroth-core` is canonical.
    - **My CI is still using the monolith wheel** — clear your CI's pip cache and pin `zeroth-core>=0.1.1` explicitly.

    ## What's not covered

    This guide only covers the monolith → `zeroth.core.*` rename. Future migrations (e.g., between `zeroth-core` versions) will get their own per-release guides as they ship.
    ```

    Final page should end up 1200-1500 words.
  </action>
  <verify>
    <automated>test -f docs/how-to/migration-from-monolith.md && grep -q "from zeroth.core" docs/how-to/migration-from-monolith.md && grep -q "econ-instrumentation-sdk" docs/how-to/migration-from-monolith.md && grep -q "ZEROTH_" docs/how-to/migration-from-monolith.md && grep -q "docker build" docs/how-to/migration-from-monolith.md && test $(wc -w < docs/how-to/migration-from-monolith.md) -ge 800</automated>
  </verify>
  <done>File exists, covers all 4 required topics (import rename, econ SDK, env vars, Docker), contains at least one before/after code example per topic, and the grep+sed recipe is runnable on both macOS and Linux.</done>
</task>

</tasks>

<verification>
- File exists
- Contains all four required subsections (import rename, econ SDK, env vars, Docker retag)
- Contains working grep+sed recipe
- Word count 800-1600
- `uv run mkdocs build` (non-strict — nav wiring in 32-06) succeeds
</verification>

<success_criteria>
DOCS-11 satisfied: Migration Guide explains how to move from the monolithic `zeroth.*` layout to `zeroth.core.*` (import rename pattern, econ SDK path swap, env var changes, Docker image retag).
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-05-SUMMARY.md`
</output>
