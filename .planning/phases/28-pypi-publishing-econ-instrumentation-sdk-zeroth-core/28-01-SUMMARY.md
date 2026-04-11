---
phase: 28
plan: 01
subsystem: packaging
tags:
  - packaging
  - pyproject
  - extras
  - pypi
  - pep561
requires: []
provides:
  - "zeroth-core 0.1.1 pyproject with minimal base deps and six optional extras"
  - "PEP 561 py.typed marker shipped in wheel"
  - "Lazy Postgres import path so base install works without psycopg"
affects:
  - pyproject.toml
  - uv.lock
  - src/zeroth/core/__init__.py
  - src/zeroth/core/storage/__init__.py
  - src/zeroth/core/py.typed
tech-stack:
  added:
    - "hatchling>=1.27 (for PEP 639 SPDX license)"
  patterns:
    - "PEP 420 namespace package layout (src/zeroth has no __init__.py)"
    - "Lazy re-export via module-level __getattr__ for optional-dependency symbols"
key-files:
  created:
    - src/zeroth/core/py.typed
  modified:
    - pyproject.toml
    - uv.lock
    - src/zeroth/core/__init__.py
    - src/zeroth/core/storage/__init__.py
key-decisions:
  - "Version target is 0.1.1 (0.1.0 is already published to PyPI and is immutable)"
  - "[sandbox] extra is an empty list — sandbox_sidecar has no Python runtime deps beyond base (shells out to docker CLI)"
  - "Kept hatchling wheel target packages = [src/zeroth] (not src/zeroth/core) because the former already produces the correct zeroth/core/ wheel layout under PEP 420"
  - "AsyncPostgresDatabase re-exports are lazy (__getattr__) so base install does not require psycopg"
requirements-completed:
  - PKG-01
  - PKG-02
  - PKG-03
duration: "~8 min"
completed: "2026-04-11"
---

# Phase 28 Plan 01: pyproject metadata and extras Summary

Restructured `pyproject.toml` for the first trusted-publisher PyPI release: bumped version to `0.1.1` (0.1.0 is already on PyPI and immutable), carved the monolithic dependencies list into a minimal core plus six PKG-03-locked optional extras (`memory-pg`, `memory-chroma`, `memory-es`, `dispatch`, `sandbox`, `all`), added OSS-grade PyPI metadata (Apache-2.0 SPDX license, `[project.urls]`, keywords, classifiers), and shipped a PEP 561 `py.typed` marker inside the built wheel. Discovered and fixed a blocking issue where the eager `AsyncPostgresDatabase` re-exports in `zeroth.core` / `zeroth.core.storage` would have crashed every base install after psycopg moved into `[memory-pg]` — replaced with lazy `__getattr__` so public API remains unchanged.

## Metrics

- **Duration:** ~8 min
- **Tasks completed:** 2 / 2
- **Files created:** 1 (`src/zeroth/core/py.typed`)
- **Files modified:** 4 (`pyproject.toml`, `uv.lock`, `src/zeroth/core/__init__.py`, `src/zeroth/core/storage/__init__.py`)
- **Commits:** 2 (`e2c0699`, `e256a70`)
- **Dependencies resolved:** 135 packages (incl. all six extras via `uv sync --all-extras --all-groups`)

## pyproject.toml Delta

### Before

```toml
[project]
name = "zeroth-core"
version = "0.1.0"
# ... no license, no keywords, no classifiers, no project.urls
dependencies = [
    # 20-ish entries, monolithic, including psycopg, psycopg-pool, pgvector,
    # chromadb-client, elasticsearch[async], redis, arq — PLUS duplicate
    # litellm>=1.83.0 and duplicate psycopg[binary]>=3.1 entries.
]
# No [project.optional-dependencies]

[build-system]
requires = ["hatchling"]

[tool.hatch.build.targets.wheel]
packages = ["src/zeroth"]
```

### After

```toml
[project]
name = "zeroth-core"
version = "0.1.1"
description = "Governed medium-code platform for production-grade multi-agent systems"
readme = "README.md"
requires-python = ">=3.12"
license = "Apache-2.0"  # PEP 639 SPDX expression
keywords = ["agents", "multi-agent", "llm", "governance", "fastapi", "workflows"]
classifiers = [
    "Development Status :: 4 - Beta",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.12",
    "Framework :: FastAPI",
    "Topic :: Software Development :: Libraries :: Python Modules",
    "Operating System :: OS Independent",
    "Typing :: Typed",
]
dependencies = [
    "fastapi>=0.115", "httpx>=0.27", "pydantic>=2.10", "pydantic-settings>=2.13",
    "sqlalchemy>=2.0", "aiosqlite>=0.22", "alembic>=1.18", "PyJWT[crypto]>=2.10",
    "PyYAML>=6.0", "python-dotenv>=1.0", "governai>=0.2.3",
    "econ-instrumentation-sdk>=0.1.1", "tenacity>=8.2", "cachetools>=5.5",
    "litellm>=1.83,<2.0", "langchain-litellm>=0.3.4", "uvicorn>=0.30", "mcp>=1.7,<2.0",
]

[project.optional-dependencies]
memory-pg    = ["psycopg[binary]>=3.3", "psycopg-pool>=3.2", "pgvector>=0.4.2"]
memory-chroma = ["chromadb-client>=1.5.6"]
memory-es    = ["elasticsearch[async]>=8.0,<9"]
dispatch     = ["redis>=5.0.0", "arq>=0.27"]
sandbox      = []
all = [
    "zeroth-core[memory-pg]",
    "zeroth-core[memory-chroma]",
    "zeroth-core[memory-es]",
    "zeroth-core[dispatch]",
    "zeroth-core[sandbox]",
]

[project.urls]
Homepage  = "https://github.com/rrrozhd/zeroth-core"
Source    = "https://github.com/rrrozhd/zeroth-core"
Issues    = "https://github.com/rrrozhd/zeroth-core/issues"
Changelog = "https://github.com/rrrozhd/zeroth-core/blob/main/CHANGELOG.md"

[build-system]
requires = ["hatchling>=1.27"]  # bumped for PEP 639 SPDX support

[tool.hatch.build.targets.wheel]
# Namespace layout (PEP 420): src/zeroth has no __init__.py; hatchling packs
# src/zeroth/core/ under zeroth/core/ in the wheel. VERIFIED against 0.1.0 wheel.
# See 28-RESEARCH.md pitfall #2.
packages = ["src/zeroth"]
```

All other sections (`[tool.hatch.metadata]`, `[dependency-groups]`, `[tool.pytest.*]`, `[tool.ruff*]`, `[tool.interrogate]`) preserved byte-for-byte.

## Version Bump Rationale

CONTEXT.md D-05 specified `0.1.0` for the first PyPI release, but `zeroth-core==0.1.0` was already uploaded to PyPI on 2026-04-10 via a manual build. PyPI versions are immutable, so the next release must be `0.1.1` or higher. Resolved in the plan preamble and carried into `[project].version` and the wheel filename (`dist/zeroth_core-0.1.1-py3-none-any.whl`).

## Wheel Layout Verification

```text
$ unzip -l dist/zeroth_core-0.1.1-py3-none-any.whl | wc -l
164 entries

# Programmatic check (from Task 2 <verify>):
wheel layout OK, 164 entries

# Assertions that passed:
- ≥1 entry starts with "zeroth/core/"
- "zeroth/core/py.typed" present
- NO "zeroth/__init__.py" (namespace package guarantee)
- NO top-level "core/..." entry (Research pitfall #2 guard)
```

## py.typed Inclusion

`src/zeroth/core/py.typed` was created as an empty file and is automatically packed by hatchling because it lives under the wheel target package root. Verified present in the wheel listing and also verified on disk inside the clean-venv install:

```text
clean-venv import OK: /tmp/zc-smoke/lib/python3.12/site-packages/zeroth/core
# (os.path.exists(p/'py.typed') assertion passed)
```

## Clean-venv Install Verification

Python 3.12.12 venv at `/tmp/zc-smoke`:

1. **Base install** (`pip install dist/zeroth_core-0.1.1-py3-none-any.whl`)
   - Initially failed with `ModuleNotFoundError: No module named 'psycopg'` — this was the [Rule 3 - Blocking] issue documented below.
   - After the fix: `import zeroth.core` succeeds and `py.typed` is present on disk. ✅
2. **Full extras install** (`pip install "dist/zeroth_core-0.1.1-py3-none-any.whl[all]"`)
   - Resolved and installed all extras' transitive graph (psycopg, psycopg-pool, pgvector, chromadb-client, elasticsearch[async], redis, arq) from real PyPI.
   - Post-install imports of `zeroth.core.memory.pgvector_connector`, `zeroth.core.memory.chroma_connector`, and `zeroth.core.dispatch.worker` all succeeded. ✅

This satisfies the local smoke-check for PKG-02 (clean-venv install) and pre-validates PKG-03 extras resolution. Plan 03's CI matrix will be the authoritative gate.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Lazy Postgres imports in `zeroth.core` and `zeroth.core.storage`**

- **Found during:** Task 2, step 6 (clean-venv wheel install smoke test)
- **Issue:** After carving `psycopg`, `psycopg-pool`, and `pgvector` out of base dependencies and into the `[memory-pg]` extra, a base `pip install zeroth-core` plus `python -c "import zeroth.core"` crashed immediately with `ModuleNotFoundError: No module named 'psycopg'`. Root cause: `src/zeroth/core/__init__.py` eagerly re-exports `AsyncPostgresDatabase`, and `src/zeroth/core/storage/__init__.py` in turn eagerly imports it from `zeroth.core.storage.async_postgres`, which top-level imports `psycopg` and `psycopg_pool`. Every downstream `from zeroth.core import ...` or `import zeroth.core` would have broken for any user who didn't install `[memory-pg]` — blocking PKG-02 entirely.
- **Fix:** Converted the Postgres symbol re-exports in both `__init__.py` files to module-level `__getattr__` lazy loaders. `AsyncPostgresDatabase` remains importable from `zeroth.core` (public API unchanged, dev mode still works, tests unaffected) but the actual `psycopg` import is deferred until first attribute access. Added `TYPE_CHECKING` imports so static type-checkers still see the symbol. Kept symbol in `__all__` so discoverability is preserved.
- **Files modified:** `src/zeroth/core/__init__.py`, `src/zeroth/core/storage/__init__.py`
- **Verification:** Clean py3.12 venv base install → `import zeroth.core` succeeds. Dev mode `uv run python -c "from zeroth.core import AsyncPostgresDatabase"` still resolves to the real class. Ruff clean on both files.
- **Commit:** `e256a70` (rolled into the Task 2 commit to keep the fix adjacent to the py.typed addition that uncovered it)

**Total deviations:** 1 auto-fixed (Rule 3). **Impact:** Without this fix, PKG-02 (clean-venv install + import works) would have failed in CI and for every real downstream user. The fix has zero impact on the public API or dev workflow.

## Authentication Gates

None — all operations were local.

## Issues Encountered

None beyond the single Rule 3 auto-fix documented above.

## Success Criteria

- [x] `pyproject.toml [project].version == "0.1.1"` (resolved Q1)
- [x] `pyproject.toml [project].license == "Apache-2.0"` (D-15, PEP 639 SPDX)
- [x] Base `[project].dependencies` contains no psycopg / pgvector / chromadb-client / elasticsearch / redis / arq entries (D-01)
- [x] `econ-instrumentation-sdk>=0.1.1` still pinned in base (PKG-01)
- [x] Six extras declared verbatim: `memory-pg`, `memory-chroma`, `memory-es`, `dispatch`, `sandbox`, `all` (PKG-03 / D-02)
- [x] `[sandbox]` is empty list (resolved Q2 / D-03)
- [x] `[all]` uses self-referencing `zeroth-core[...]` entries (D-03)
- [x] `[project.urls]` has Homepage, Source, Issues, Changelog (D-22)
- [x] `[project].keywords` and `[project].classifiers` populated (D-23)
- [x] `[tool.hatch.build.targets.wheel].packages` remains `["src/zeroth"]` with explanatory comment (resolved Q3)
- [x] `[build-system].requires == ["hatchling>=1.27"]` for PEP 639 support
- [x] `src/zeroth/core/py.typed` exists and is included in the built wheel
- [x] `uv build` produces `dist/zeroth_core-0.1.1-py3-none-any.whl` with correct `zeroth/core/` layout and no `zeroth/__init__.py`
- [x] Clean-venv `pip install` of the built wheel + `[all]` extra + key imports all succeed locally

## Next Steps

Ready for Plan 02 (`28-02-repo-metadata-and-hello-example`). Plan 02 adds LICENSE, CHANGELOG.md, CONTRIBUTING.md, and `examples/hello.py` — no dependency on the packaging bits finalized here beyond the `0.1.1` version anchor and the Apache-2.0 declaration, which CHANGELOG.md `[0.1.1]` section will match.

## Self-Check

- [x] `pyproject.toml` exists with version `0.1.1` (verified via tomllib script)
- [x] `src/zeroth/core/py.typed` exists on disk
- [x] `src/zeroth/core/__init__.py` and `src/zeroth/core/storage/__init__.py` modifications in place
- [x] Commit `e2c0699` (Task 1 — pyproject restructure) exists in `git log`
- [x] Commit `e256a70` (Task 2 — py.typed + lazy imports) exists in `git log`
- [x] `dist/zeroth_core-0.1.1-py3-none-any.whl` built and present

## Self-Check: PASSED
