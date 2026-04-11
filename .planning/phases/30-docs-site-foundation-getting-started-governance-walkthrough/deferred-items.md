# Phase 30 — Deferred Items

Out-of-scope issues discovered during plan execution. Not fixed by Phase 30.

## Plan 30-05

- **Pre-existing ruff I001 in `examples/hello.py`**: `from litellm import completion`
  triggers "Organize imports" when linted via `ruff check examples/`. The file
  originates from Phase 28-02 (commit cc833ee) and was not touched by Plan 30-05.
  `ruff check src/ tests/` is clean. Recommend a dedicated maintenance commit or
  sweep in Phase 31 to fix.
