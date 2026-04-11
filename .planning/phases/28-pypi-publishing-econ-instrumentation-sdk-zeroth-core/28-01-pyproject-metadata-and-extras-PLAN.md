---
phase: 28
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - pyproject.toml
  - src/zeroth/core/py.typed
autonomous: true
requirements:
  - PKG-01
  - PKG-02
  - PKG-03
tags:
  - packaging
  - pyproject
  - extras
  - pypi

must_haves:
  truths:
    - "pyproject.toml [project].version is 0.1.1 (0.1.0 is already live on PyPI, immutable)"
    - "Base dependencies list contains ONLY minimal core — no psycopg, pgvector, chromadb-client, elasticsearch, redis, arq"
    - "[project.optional-dependencies] declares exactly six extras: memory-pg, memory-chroma, memory-es, dispatch, sandbox, all (names verbatim per PKG-03)"
    - "[sandbox] is declared as an empty list (marker extra; sandbox_sidecar has no runtime deps beyond base)"
    - "[all] is the union of the other five extras via self-referencing zeroth-core[...] entries"
    - "uv build produces a wheel whose entries are rooted at zeroth/core/... with NO top-level zeroth/__init__.py"
    - "uv build produces a wheel that includes src/zeroth/core/py.typed (PEP 561 marker)"
    - "pyproject.toml declares license = \"Apache-2.0\" (PEP 639 SPDX expression)"
    - "[project.urls] block has Homepage, Source, Issues, Changelog entries"
    - "[project].classifiers and [project].keywords are set for PyPI metadata"
    - "pip install -e . (dev install) and pip install 'zeroth-core[all]' (from built wheel in clean venv) both succeed"
    - "econ-instrumentation-sdk>=0.1.1 remains pinned in base dependencies (PKG-01 verification)"
  artifacts:
    - path: "pyproject.toml"
      provides: "Updated project metadata, version 0.1.1, six optional-dependency extras, [project.urls], classifiers, keywords, Apache-2.0 license, py.typed-inclusive wheel config"
      contains: "0.1.1"
    - path: "src/zeroth/core/py.typed"
      provides: "PEP 561 type-hint marker for downstream consumers"
  key_links:
    - from: "pyproject.toml [project.optional-dependencies].all"
      to: "other extras (memory-pg, memory-chroma, memory-es, dispatch, sandbox)"
      via: "self-referencing zeroth-core[...] entries"
      pattern: "zeroth-core\\[memory-pg\\]"
    - from: "pyproject.toml [tool.hatch.build.targets.wheel]"
      to: "built wheel layout"
      via: "hatchling package resolution"
      pattern: "packages\\s*=\\s*\\[.src/zeroth.\\]"
---

<objective>
Restructure `pyproject.toml` for the first trusted-publisher PyPI release: bump version to 0.1.1 (0.1.0 is already live and immutable), carve the monolithic dependencies list into a minimal base plus six optional extras (names locked verbatim per PKG-03), and add all OSS-grade project metadata (license, URLs, classifiers, keywords). Ship the PEP 561 py.typed marker so downstream users get type hints.

Purpose: PKG-02 requires `zeroth-core` installable from PyPI; PKG-03 requires every extra declared and installable; PKG-01 requires the econ SDK consumed via PyPI constraint (already done — this plan verifies it stays that way).

Output: A `pyproject.toml` ready for `uv build` + trusted-publisher upload, plus a py.typed marker. Release workflow lives in Plan 03.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-CONTEXT.md
@.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-RESEARCH.md
@.planning/REQUIREMENTS.md
@pyproject.toml
@CLAUDE.md

<resolved_open_questions>
Three questions from research were resolved by the user before this plan was written — treat as locked:

1. **Version target is `0.1.1`, not `0.1.0`.** CONTEXT.md D-05 said "0.1.0, no bump" but `zeroth-core==0.1.0` is already on PyPI (uploaded 2026-04-10, immutable). This plan uses `0.1.1` throughout. Every mention of "D-05 / 0.1.0" in CONTEXT.md should be read as 0.1.1.
2. **`[sandbox]` extra is an empty list marker.** `zeroth.core.sandbox_sidecar` imports only fastapi + pydantic (already in base) + stdlib, and shells out to the `docker` CLI via `asyncio.create_subprocess_exec`. No Python runtime dep exists. Declare `sandbox = []` so `pip install "zeroth-core[sandbox]"` succeeds as a no-op while preserving PKG-03 verbatim.
3. **Do NOT change the hatchling wheel target.** CONTEXT.md D-21 said to change `packages = ["src/zeroth"]` → `["src/zeroth/core"]`. Research (pitfall #2) proved this is wrong: the current setting already produces the correct `zeroth/core/...` wheel layout (VERIFIED by unzipping the existing 0.1.0 wheel), and changing it to `src/zeroth/core` would make hatchling root files at `core/` instead of `zeroth/core/`. This plan keeps `packages = ["src/zeroth"]` and adds a short comment explaining why.
</resolved_open_questions>

<interfaces>
<!-- Current pyproject.toml [project] shape (to be modified). -->
<!-- Executor should edit this file in place — do NOT rewrite from scratch. -->

Current `pyproject.toml [project]` has:
  - name = "zeroth-core"
  - version = "0.1.0"                           ← bump to "0.1.1"
  - description = "Governed medium-code platform for production-grade multi-agent systems"
  - readme = "README.md"
  - requires-python = ">=3.12"
  - dependencies = [ ... 20-ish entries, monolithic ... ]   ← carve into base + extras
  - NO [project.optional-dependencies]          ← add
  - NO [project.urls]                            ← add
  - NO license field                             ← add "Apache-2.0"
  - NO classifiers                               ← add
  - NO keywords                                  ← add

Current `[tool.hatch.build.targets.wheel]`:
  - packages = ["src/zeroth"]   ← KEEP as-is, add comment explaining why (per resolved Q3)

Current `src/zeroth/` layout (verified this session):
  - src/zeroth/core/__init__.py                  ← exists
  - NO src/zeroth/__init__.py                    ← correctly absent (namespace pkg)

Subpackages under `src/zeroth/core/` that drive extras mapping:
  - memory/pgvector_connector.py      → [memory-pg]
  - memory/chroma_connector.py        → [memory-chroma]
  - memory/ (elasticsearch backend)   → [memory-es]
  - dispatch/worker.py, dispatch/*    → [dispatch]   (arq + redis)
  - sandbox_sidecar/*                 → [sandbox]    (empty — shells out to docker CLI)
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Rewrite pyproject.toml with base deps, six extras, and PyPI metadata</name>
  <files>pyproject.toml</files>
  <behavior>
    After this task:
    - `[project].version` is `"0.1.1"` (per resolved Q1; overrides CONTEXT.md D-05)
    - `[project].license` is `"Apache-2.0"` (SPDX string, PEP 639 form; per D-15)
    - `[project].dependencies` contains ONLY the base list (per D-01):
        fastapi>=0.115, httpx>=0.27, pydantic>=2.10, pydantic-settings>=2.13,
        sqlalchemy>=2.0, aiosqlite>=0.22, alembic>=1.18, PyJWT[crypto]>=2.10,
        PyYAML>=6.0, python-dotenv>=1.0, governai>=0.2.3, econ-instrumentation-sdk>=0.1.1,
        tenacity>=8.2, cachetools>=5.5, litellm>=1.83,<2.0, langchain-litellm>=0.3.4,
        uvicorn>=0.30, mcp>=1.7,<2.0
      and NO entries for psycopg, psycopg-pool, pgvector, chromadb-client, elasticsearch, redis, arq.
      Also remove the duplicate `litellm>=1.83.0` and duplicate `psycopg[binary]>=3.1` entries currently in the file.
    - `[project.optional-dependencies]` is a new table declaring exactly six extras with names VERBATIM per PKG-03 / D-02:
        memory-pg    = ["psycopg[binary]>=3.3", "psycopg-pool>=3.2", "pgvector>=0.4.2"]      # D-03
        memory-chroma = ["chromadb-client>=1.5.6"]                                           # D-03
        memory-es    = ["elasticsearch[async]>=8.0,<9"]                                       # D-03 — keep <9 cap; ES9 removed [async] shape (pitfall #7)
        dispatch     = ["redis>=5.0.0", "arq>=0.27"]                                          # D-03
        sandbox      = []                                                                     # resolved Q2: empty marker
        all          = [
                         "zeroth-core[memory-pg]",
                         "zeroth-core[memory-chroma]",
                         "zeroth-core[memory-es]",
                         "zeroth-core[dispatch]",
                         "zeroth-core[sandbox]",
                       ]                                                                      # D-03
    - `[project.urls]` is a new table (per D-22):
        Homepage  = "https://github.com/rrrozhd/zeroth-core"
        Source    = "https://github.com/rrrozhd/zeroth-core"
        Issues    = "https://github.com/rrrozhd/zeroth-core/issues"
        Changelog = "https://github.com/rrrozhd/zeroth-core/blob/main/CHANGELOG.md"
    - `[project].keywords` is set (per D-23): ["agents", "multi-agent", "llm", "governance", "fastapi", "workflows"]
    - `[project].classifiers` is set (per D-23) with at minimum:
        "Development Status :: 4 - Beta",
        "License :: OSI Approved :: Apache Software License",
        "Programming Language :: Python :: 3",
        "Programming Language :: Python :: 3.12",
        "Framework :: FastAPI",
        "Topic :: Software Development :: Libraries :: Python Modules",
        "Operating System :: OS Independent",
        "Typing :: Typed"
    - `[build-system].requires` is `["hatchling>=1.27"]` (needed for PEP 639 SPDX license support)
    - `[tool.hatch.build.targets.wheel]` retains `packages = ["src/zeroth"]` (per resolved Q3 — do NOT change) AND a comment line immediately above the packages key reading:
        `# Namespace layout (PEP 420): src/zeroth has no __init__.py; hatchling packs src/zeroth/core/ under zeroth/core/ in the wheel. VERIFIED against 0.1.0 wheel. See 28-RESEARCH.md pitfall #2.`
    - `[tool.hatch.metadata].allow-direct-references = true` is preserved (currently there; keep it — harmless even after PKG-01 no longer needs it)
    - Existing `[dependency-groups].dev`, `[tool.pytest.ini_options]`, `[tool.ruff*]`, `[tool.interrogate]` blocks are preserved BYTE-FOR-BYTE — this task only touches `[project]`, `[build-system]`, and `[tool.hatch.*]`.
  </behavior>
  <action>
    1. Read current `pyproject.toml` to confirm line-level structure.
    2. Rewrite the `[project]` table to match the behavior spec above. Preserve field ordering convention: name, version, description, readme, requires-python, license, keywords, classifiers, authors (omit if not present today), dependencies. Then append `[project.optional-dependencies]` and `[project.urls]` tables.
    3. Bump `[build-system].requires` from `["hatchling"]` to `["hatchling>=1.27"]` so PEP 639 license SPDX is supported (per RESEARCH.md).
    4. Leave `[tool.hatch.build.targets.wheel].packages` as `["src/zeroth"]` (resolved Q3). Add the explanatory comment line immediately above it.
    5. Do NOT add `authors` or `maintainers` fields unless already present — defer to follow-up if user wants.
    6. Do NOT touch `[dependency-groups]`, `[tool.pytest.ini_options]`, `[tool.ruff*]`, `[tool.interrogate]`, `[tool.hatch.metadata]`.
    7. Run `uv lock` to regenerate the lockfile against the new deps shape (fails loudly if any dep name is wrong).
    8. Run `uv sync --all-extras --all-groups` to install every extra locally and ensure the full graph resolves.
    9. Run `uv run python -c "import zeroth.core; print(zeroth.core.__path__)"` to confirm the package still imports in dev mode.
    10. Run `uv run ruff check src tests` to catch any lint regression introduced by the sync (there shouldn't be any — this is a config-only change).
  </action>
  <verify>
    <automated>uv lock && uv sync --all-extras --all-groups && uv run python -c "import zeroth.core; import tomllib; d=tomllib.load(open('pyproject.toml','rb')); assert d['project']['version']=='0.1.1', d['project']['version']; assert d['project']['license']=='Apache-2.0', d['project']['license']; extras=d['project']['optional-dependencies']; assert set(extras.keys())=={'memory-pg','memory-chroma','memory-es','dispatch','sandbox','all'}, extras.keys(); assert extras['sandbox']==[], extras['sandbox']; assert all('zeroth-core[' in x for x in extras['all']), extras['all']; print('OK')"</automated>
  </verify>
  <done>
    `uv lock` succeeds; `uv sync --all-extras --all-groups` succeeds; `zeroth.core` still imports; the inline assertion script prints "OK" confirming version 0.1.1, Apache-2.0 license, all six extras present with sandbox empty and `all` self-referencing.
  </done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Add py.typed marker and verify built wheel layout end-to-end</name>
  <files>src/zeroth/core/py.typed, pyproject.toml</files>
  <behavior>
    After this task:
    - `src/zeroth/core/py.typed` exists as an empty file (PEP 561 marker).
    - `uv build` produces a fresh `dist/zeroth_core-0.1.1-py3-none-any.whl` and `dist/zeroth_core-0.1.1.tar.gz`.
    - `unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl` shows:
        - At least one entry under `zeroth/core/...`
        - `zeroth/core/py.typed` is present
        - NO entry named `zeroth/__init__.py` (namespace package guarantee)
        - NO entry rooted at `core/...` without the `zeroth/` prefix (pitfall #2 guard)
    - A clean venv can `pip install` the built wheel and import `zeroth.core`.
    - A clean venv can `pip install 'zeroth-core[all] @ file://.../dist/zeroth_core-0.1.1-py3-none-any.whl'` (or install the wheel then `pip install 'zeroth-core[all]==0.1.1'` — if the latter hits PyPI for non-local deps, that's fine; the point is the extras resolve).
  </behavior>
  <action>
    1. Create empty file `src/zeroth/core/py.typed` (use the Write tool with empty content; if the tool rejects empty content, write a single newline).
    2. Verify hatchling will include it: since `src/zeroth/core/py.typed` is under the wheel's package root (`src/zeroth` with the `zeroth/core/` subtree), it's included automatically. No `[tool.hatch.build.targets.wheel].include` rule needed. If the wheel check in step 5 shows it missing, add an explicit `include = ["src/zeroth/core/py.typed"]` to the wheel target as a fallback.
    3. Clean any stale `dist/` artifacts: `rm -rf dist/`.
    4. Run `uv build`. Must succeed and produce both sdist and wheel at version 0.1.1.
    5. Run `unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl` and inspect:
        - Confirm presence of `zeroth/core/__init__.py`
        - Confirm presence of `zeroth/core/py.typed`
        - Confirm absence of `zeroth/__init__.py` (the empty namespace dir must NOT get an __init__)
        - Confirm no entries begin with `core/` at the top level
    6. Create a clean throw-away venv and install the wheel:
        ```
        rm -rf /tmp/zc-smoke && python3 -m venv /tmp/zc-smoke
        /tmp/zc-smoke/bin/pip install --upgrade pip
        /tmp/zc-smoke/bin/pip install dist/zeroth_core-0.1.1-py3-none-any.whl
        /tmp/zc-smoke/bin/python -c "import zeroth.core; p=zeroth.core.__path__[0]; import os; assert os.path.exists(os.path.join(p,'py.typed')); print('py.typed OK:', p)"
        ```
    7. In that same clean venv, install the `[all]` extra (this will hit real PyPI for psycopg/pgvector/chromadb-client/elasticsearch/redis/arq — that's fine, we need to verify resolution):
        ```
        /tmp/zc-smoke/bin/pip install "dist/zeroth_core-0.1.1-py3-none-any.whl[all]"
        /tmp/zc-smoke/bin/python -c "import zeroth.core.memory.pgvector_connector; import zeroth.core.memory.chroma_connector; import zeroth.core.dispatch.worker; print('[all] imports OK')"
        ```
       This satisfies PKG-02 clean-venv install and pre-validates PKG-03 extras resolution locally. Plan 03's CI matrix is the full PKG-03 gate; this is the local smoke check.
    8. Leave `dist/` in place — Plan 03 does not use it (workflow rebuilds in CI), but keeping it helps the phase checker verify.
  </action>
  <verify>
    <automated>rm -rf dist && uv build && unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl | awk '{print $NF}' | python3 -c "import sys; entries=[l.strip() for l in sys.stdin if l.strip()]; assert any(e.startswith('zeroth/core/') for e in entries), 'no zeroth/core/ entries'; assert 'zeroth/core/py.typed' in entries, 'py.typed missing from wheel'; assert 'zeroth/__init__.py' not in entries, 'namespace pollution: top-level zeroth/__init__.py found'; assert not any(e.startswith('core/') and not e.startswith('zeroth/') for e in entries), 'pitfall #2: files rooted at core/ instead of zeroth/core/'; print('wheel layout OK,', len(entries), 'entries')" && rm -rf /tmp/zc-smoke && python3 -m venv /tmp/zc-smoke && /tmp/zc-smoke/bin/pip install --quiet --upgrade pip && /tmp/zc-smoke/bin/pip install --quiet dist/zeroth_core-0.1.1-py3-none-any.whl && /tmp/zc-smoke/bin/python -c "import zeroth.core, os; p=zeroth.core.__path__[0]; assert os.path.exists(os.path.join(p,'py.typed')); print('clean-venv import OK:', p)"</automated>
  </verify>
  <done>
    `dist/zeroth_core-0.1.1-py3-none-any.whl` exists; wheel listing check passes (zeroth/core/ rooted, py.typed present, no top-level zeroth/__init__.py, no core/-rooted entries); clean-venv wheel install imports `zeroth.core` and finds `py.typed` on disk. Extras resolution smoke check (step 7 of action) prints "[all] imports OK".
  </done>
</task>

</tasks>

<verification>
Overall plan verification (both tasks combined):

1. `uv lock && uv sync --all-extras --all-groups` — full dep graph resolves with the new extras split
2. `uv build` — produces 0.1.1 sdist + wheel
3. `python -c "import tomllib; d=tomllib.load(open('pyproject.toml','rb')); print(d['project']['version'], d['project']['license'], sorted(d['project']['optional-dependencies'].keys()))"` — prints `0.1.1 Apache-2.0 ['all', 'dispatch', 'memory-chroma', 'memory-es', 'memory-pg', 'sandbox']`
4. `unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl | grep -E '(zeroth/core/|py.typed|zeroth/__init__)'` — shows zeroth/core/py.typed, shows zeroth/core/* entries, shows NO zeroth/__init__.py line
5. Clean-venv wheel install + import `zeroth.core` succeeds
6. Clean-venv `[all]` install + import of `zeroth.core.memory.pgvector_connector`, `zeroth.core.memory.chroma_connector`, `zeroth.core.dispatch.worker` succeeds (local pre-check for PKG-03; CI matrix in Plan 03 is the authoritative gate)
</verification>

<success_criteria>
- [ ] `pyproject.toml [project].version == "0.1.1"` (per resolved Q1)
- [ ] `pyproject.toml [project].license == "Apache-2.0"` (per D-15)
- [ ] Base `[project].dependencies` contains no psycopg, pgvector, chromadb-client, elasticsearch, redis, or arq entries (per D-01)
- [ ] `econ-instrumentation-sdk>=0.1.1` still pinned in base (PKG-01 verification per resolved context)
- [ ] Six extras declared with exact names `memory-pg`, `memory-chroma`, `memory-es`, `dispatch`, `sandbox`, `all` (per PKG-03 / D-02)
- [ ] `[sandbox]` is empty list (per resolved Q2 / D-03)
- [ ] `[all]` uses self-referencing `zeroth-core[...]` entries (per D-03)
- [ ] `[project.urls]` has Homepage, Source, Issues, Changelog (per D-22)
- [ ] `[project].keywords` and `[project].classifiers` populated (per D-23)
- [ ] `[tool.hatch.build.targets.wheel].packages` remains `["src/zeroth"]` with explanatory comment (per resolved Q3 / D-21 override)
- [ ] `[build-system].requires == ["hatchling>=1.27"]` for PEP 639 support
- [ ] `src/zeroth/core/py.typed` exists and is included in the built wheel
- [ ] `uv build` produces `dist/zeroth_core-0.1.1-py3-none-any.whl` with correct `zeroth/core/` layout and no `zeroth/__init__.py`
- [ ] Clean-venv `pip install` of the built wheel + `[all]` extra + key imports all succeed locally
</success_criteria>

<output>
After completion, create `.planning/phases/28-pypi-publishing-econ-instrumentation-sdk-zeroth-core/28-01-SUMMARY.md` covering: pyproject delta (before/after), wheel layout verification output, clean-venv install result, py.typed inclusion confirmation, and a short note that version was bumped to 0.1.1 because 0.1.0 is immutable on PyPI.
</output>
