---
phase: 32
plan: 03
subsystem: docs
tags: [docs, reference, configuration, pydantic-settings, drift-check]
dependency_graph:
  requires: [zeroth.core.config.settings.ZerothSettings]
  provides: [docs/reference/configuration.md, scripts/dump_config.py]
  affects: [CI drift gates]
tech_stack:
  added: []
  patterns: [pydantic model_fields introspection, argparse script with --check drift gate, mkdocs strict build]
key_files:
  created:
    - scripts/dump_config.py
  modified:
    - docs/reference/configuration.md
decisions:
  - Used pydantic SecretStr identity check via recursive annotation walk (robust against Union/Optional).
  - Escaped `|` inside rendered type annotations to prevent markdown table parsing issues.
  - Pulled one-line section blurb from each sub-Settings class __doc__ first non-empty line.
  - RegulusSettings introspected like every other sub-Settings — no hard-coded fields.
metrics:
  duration: ~10m
  completed: 2026-04-11
  tasks: 2
  files_touched: 2
---

# Phase 32 Plan 03: Configuration Reference (dump_config) Summary

Auto-generated the Configuration Reference page from `zeroth.core.config.settings.ZerothSettings` via a new `scripts/dump_config.py` introspection script, closing DOCS-09 with a CI drift gate.

## What Was Built

### scripts/dump_config.py
- Imports `ZerothSettings` and walks `model_fields` to find every nested `BaseModel` sub-section.
- For each section, walks its own `model_fields` and emits a markdown table row per field: ``| `ZEROTH_<SECTION>__<FIELD>` | `type` | default | ✓ | description |``.
- Secret detection uses recursive `SecretStr` identity matching (handles `SecretStr`, `SecretStr | None`, etc.).
- Type rendering strips `typing.` noise, expands Unions with `|`, escapes `|` for markdown.
- Default rendering handles `PydanticUndefined`, `default_factory`, redacts secrets to `***`.
- argparse: `--out` (default `docs/reference/configuration.md`), `--check` (drift gate exits 1 with `DRIFT:` prefix on diff, 0 with `OK:` when identical). Section blurbs pulled from sub-Settings class docstrings.

### docs/reference/configuration.md
- Replaces the Phase 30 stub.
- 13 sections in declaration order: Database, Redis, Auth, Regulus, Memory, Pgvector, Chroma, Elasticsearch, Sandbox, Webhook, Approval Sla, Dispatch, Tls.
- 4 secret flags: `ZEROTH_DATABASE__POSTGRES_DSN`, `ZEROTH_DATABASE__ENCRYPTION_KEY`, `ZEROTH_REDIS__PASSWORD`, `ZEROTH_REGULUS__API_KEY`.

## Verification

| Check | Result |
| ----- | ------ |
| `uv run python scripts/dump_config.py --out /tmp/...` runs clean | PASS |
| Output contains `ZEROTH_DATABASE__POSTGRES_DSN`, `ZEROTH_REDIS__PASSWORD`, `| ✓ |` | PASS |
| `--check` on fresh file exits 0 with `OK:` | PASS |
| `--check` on drifted file exits 1 with `DRIFT:` | PASS |
| `grep -c "^## " docs/reference/configuration.md` ≥ 13 | 13 — PASS |
| `grep -c "| ✓ |" docs/reference/configuration.md` ≥ 3 | 4 — PASS |
| `uv run mkdocs build --strict` | PASS (only pre-existing INFO notices about python-api anchors, out of scope for 32-03) |

## Commits

| Task | Hash | Message |
| ---- | ---- | ------- |
| 1 | f0dee7a | feat(32-03): add scripts/dump_config.py for pydantic-settings reference |
| 2 | 6b5a9c9 | docs(32-03): generate configuration reference from pydantic-settings |

## Deviations from Plan

None — plan executed exactly as written.

## Self-Check: PASSED

- FOUND: scripts/dump_config.py
- FOUND: docs/reference/configuration.md
- FOUND commit: f0dee7a
- FOUND commit: 6b5a9c9
