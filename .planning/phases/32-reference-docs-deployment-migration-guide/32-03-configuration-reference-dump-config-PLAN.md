---
phase: 32-reference-docs-deployment-migration-guide
plan: 03
type: execute
wave: 1
depends_on: []
files_modified:
  - scripts/dump_config.py
  - docs/reference/configuration.md
autonomous: true
requirements:
  - DOCS-09
must_haves:
  truths:
    - "scripts/dump_config.py introspects ZerothSettings and every nested sub-Settings class via pydantic model_fields"
    - "docs/reference/configuration.md is a generated markdown table with columns: Env Var | Type | Default | Secret | Description"
    - "scripts/dump_config.py --check exits non-zero if docs/reference/configuration.md would change on regeneration"
    - "Every nested section (database, redis, auth, regulus, memory, pgvector, chroma, elasticsearch, sandbox, webhook, approval_sla, dispatch, tls) appears with its fields and correct ZEROTH_<section>__<field> env var names"
  artifacts:
    - path: "scripts/dump_config.py"
      provides: "pydantic-settings introspection + drift check"
      contains: "model_fields"
    - path: "docs/reference/configuration.md"
      provides: "auto-generated env var reference tables"
      contains: "| Env Var |"
  key_links:
    - from: "scripts/dump_config.py"
      to: "zeroth.core.config.settings.ZerothSettings"
      via: "import and model_fields traversal"
      pattern: "ZerothSettings"
    - from: "scripts/dump_config.py --check"
      to: "docs/reference/configuration.md"
      via: "byte-diff after regeneration"
      pattern: "--check"
---

<objective>
Write `scripts/dump_config.py` that introspects `zeroth.core.config.settings.ZerothSettings` (and its nested sub-Settings `BaseModel` children) via `model_fields`, emits `docs/reference/configuration.md` as a set of markdown tables, and supports a `--check` drift mode. Replace the Phase 30 stub with the generated output.

Purpose: Closes DOCS-09. Every env var, default, type, and secret flag is documented automatically from the single source of truth (`settings.py`), so the reference cannot drift.

Output: New script `scripts/dump_config.py`, regenerated `docs/reference/configuration.md`.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/32-reference-docs-deployment-migration-guide/32-CONTEXT.md

@src/zeroth/core/config/settings.py
@docs/reference/configuration.md
@scripts/dump_openapi.py

<interfaces>
Settings structure (from src/zeroth/core/config/settings.py):

- `ZerothSettings(BaseSettings)` with `env_prefix="ZEROTH_"`, `env_nested_delimiter="__"`
- Nested `BaseModel` children (one section each):
  - database → DatabaseSettings (backend, sqlite_path, postgres_dsn [secret], postgres_pool_min, postgres_pool_max, encryption_key [secret])
  - redis → RedisSettings (mode, host, port, password [secret], key_prefix, db, tls)
  - auth → AuthSettings (api_keys_json, bearer_json)
  - regulus → RegulusSettings (imported from zeroth.core.econ.models — introspect at runtime, do not hard-code)
  - memory → MemorySettings
  - pgvector → PgvectorSettings
  - chroma → ChromaSettings
  - elasticsearch → ElasticsearchSettings
  - sandbox → SandboxSettings
  - webhook → WebhookSettings
  - approval_sla → ApprovalSLASettings
  - dispatch → DispatchSettings
  - tls → TLSSettings

Env var convention: `ZEROTH_<SECTION>__<FIELD>` (uppercase, double underscore between section and field). Example: `database.postgres_dsn` → `ZEROTH_DATABASE__POSTGRES_DSN`.

Secret detection: Any field whose annotation (or origin type) is `SecretStr` or `SecretStr | None` is a secret. Use `pydantic.SecretStr` identity check, not string name.

Description extraction: Prefer `FieldInfo.description`; fall back to empty string. Field descriptions are not currently populated for most sub-Settings — that's acceptable (empty description column) and is itself a hint to add them over time.

Type rendering: Use `field.annotation` and render with `typing.get_type_hints` or stringification. For `SecretStr`, render as `SecretStr`. For `str | None`, render as `str | None`. For `list[str]`, render as `list[str]`.

Default rendering: `field.default` if not `PydanticUndefined`; otherwise `field.default_factory()` if present; otherwise `—`. For `SecretStr` with a default, render as `***` (do not leak).
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Write scripts/dump_config.py with introspection + --check</name>
  <files>scripts/dump_config.py</files>
  <action>
    Per D-03 and D-07:

    Create `scripts/dump_config.py` modeled on `scripts/dump_openapi.py` (argparse + main() returning int + `--out` + `--check`). The script must:

    1. Import `ZerothSettings` from `zeroth.core.config.settings`.

    2. Walk `ZerothSettings.model_fields` to find nested `BaseModel` sections. For each top-level field whose annotation is a subclass of `pydantic.BaseModel`, emit a `## <Section name>` heading and a table.

    3. For each nested section, walk its `model_fields`. For each field emit a row:
       ```
       | `ZEROTH_<SECTION>__<FIELD>` | `<type>` | `<default>` | <secret_yes_or_blank> | <description> |
       ```
       - `<SECTION>` = parent field name uppercased.
       - `<FIELD>` = leaf field name uppercased.
       - `<type>`: stringified annotation. For `SecretStr | None` render as `SecretStr | None`.
       - `<default>`: `field.default` if set and not `PydanticUndefined`; else call `default_factory` if present; stringify. For `SecretStr`, render `***`. For `None`, render `None`. For `list`/`dict`, use `repr`.
       - Secret column: `✓` if annotation contains `SecretStr`, else blank.
       - Description: `field.description or ""`.

    4. Each table is preceded by a short prose line explaining the section (one sentence hand-written per section, derived from the docstring of the sub-Settings class via `cls.__doc__`).

    5. Top-of-file preamble (static):
       ```markdown
       # Configuration Reference

       Every Zeroth setting is loaded from (in priority order): environment variables (`ZEROTH_` prefix, nested via `__`), a local `.env` file, then `zeroth.yaml`. This reference is auto-generated from `zeroth.core.config.settings` via `scripts/dump_config.py` — **do not edit by hand**.

       CI runs `python scripts/dump_config.py --check` on every PR and fails if this file is stale.
       ```

    6. Sort order: use the field declaration order on `ZerothSettings` (model_fields preserves it in Pydantic v2). Within each section, also preserve declaration order.

    7. argparse:
       - `--out PATH` (default: `docs/reference/configuration.md`)
       - `--check` — regenerate in-memory and diff against `--out`; exit 1 with `DRIFT: ...` message if different, exit 0 with `OK: ...` if identical.
       - Default behavior (no flags): write to `--out` path.

    8. Handle `RegulusSettings` by introspection like every other sub-Settings (don't hard-code its fields — it lives in `zeroth.core.econ.models` and may evolve).

    9. Keep the script dependency-free beyond what `zeroth-core` already installs (pydantic, pydantic-settings). Add minimal docstrings per ruff.
  </action>
  <verify>
    <automated>uv run python scripts/dump_config.py --out /tmp/gsd-config.md && test -s /tmp/gsd-config.md && grep -q "ZEROTH_DATABASE__POSTGRES_DSN" /tmp/gsd-config.md && grep -q "ZEROTH_REDIS__PASSWORD" /tmp/gsd-config.md && grep -q "| ✓ |" /tmp/gsd-config.md</automated>
  </verify>
  <done>scripts/dump_config.py exists, runs without errors, produces a markdown file with a section per nested sub-Settings, all env vars correctly prefixed, SecretStr fields flagged with ✓.</done>
</task>

<task type="auto">
  <name>Task 2: Generate docs/reference/configuration.md + verify --check semantics</name>
  <files>docs/reference/configuration.md</files>
  <action>
    Per D-03:

    1. Run the generator to write the real file:
       ```bash
       uv run python scripts/dump_config.py --out docs/reference/configuration.md
       ```

    2. Inspect the produced file. It should contain:
       - The static preamble.
       - One `## <Section>` heading per top-level ZerothSettings field (database, redis, auth, regulus, memory, pgvector, chroma, elasticsearch, sandbox, webhook, approval_sla, dispatch, tls).
       - One table per section with columns: `Env Var | Type | Default | Secret | Description`.
       - `ZEROTH_DATABASE__POSTGRES_DSN` and `ZEROTH_DATABASE__ENCRYPTION_KEY` marked as secret (✓).
       - `ZEROTH_REDIS__PASSWORD` marked as secret (✓).

    3. Verify `--check` is idempotent: re-run `uv run python scripts/dump_config.py --check --out docs/reference/configuration.md` — must exit 0 with "OK".

    4. Verify drift detection: append a junk line to the file, run `--check`, confirm exit 1 and "DRIFT" message, then regenerate.

    5. Verify `uv run mkdocs build --strict` still passes (no broken headings, the existing `Configuration: reference/configuration.md` nav entry now has real content).
  </action>
  <verify>
    <automated>uv run python scripts/dump_config.py --out docs/reference/configuration.md && uv run python scripts/dump_config.py --check --out docs/reference/configuration.md && uv run mkdocs build --strict</automated>
  </verify>
  <done>docs/reference/configuration.md contains auto-generated tables for all 13 sub-Settings sections, --check exits 0 on a fresh file, strict mkdocs build passes.</done>
</task>

</tasks>

<verification>
- `uv run python scripts/dump_config.py --check --out docs/reference/configuration.md` exits 0
- Deliberately editing the file and rerunning `--check` exits 1 with DRIFT message
- `grep -c "^## " docs/reference/configuration.md` ≥ 13
- `grep -c "| ✓ |" docs/reference/configuration.md` ≥ 3 (at minimum: postgres_dsn, encryption_key, redis.password)
- `uv run mkdocs build --strict` passes
</verification>

<success_criteria>
DOCS-09 satisfied: Configuration Reference is auto-generated from the pydantic-settings schemas and documents every env var, its default, and whether it is a secret. Drift is gated in CI.
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-03-SUMMARY.md`
</output>
