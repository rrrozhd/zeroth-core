---
phase: 32-reference-docs-deployment-migration-guide
plan: 02
type: execute
wave: 1
depends_on: []
files_modified:
  - scripts/dump_openapi.py
  - docs/reference/http-api.md
  - docs/assets/openapi/zeroth-core-openapi.json
autonomous: true
requirements:
  - DOCS-08
must_haves:
  truths:
    - "docs/reference/http-api.md renders an interactive Swagger UI browsing the zeroth-core OpenAPI spec"
    - "scripts/dump_openapi.py --check exits non-zero when openapi/zeroth-core-openapi.json is stale versus the live FastAPI schema"
    - "The OpenAPI spec consumed by the docs page is the committed snapshot at openapi/zeroth-core-openapi.json (copied or referenced into the docs build)"
  artifacts:
    - path: "scripts/dump_openapi.py"
      provides: "drift-check mode via --check flag"
      contains: "--check"
    - path: "docs/reference/http-api.md"
      provides: "Swagger UI embed page"
      contains: "swagger-ui"
    - path: "docs/assets/openapi/zeroth-core-openapi.json"
      provides: "OpenAPI snapshot under docs/ so mkdocs can serve it as a static asset"
  key_links:
    - from: "docs/reference/http-api.md"
      to: "docs/assets/openapi/zeroth-core-openapi.json"
      via: "Swagger UI script tag url attribute"
      pattern: "assets/openapi/zeroth-core-openapi.json"
    - from: "scripts/dump_openapi.py --check"
      to: "openapi/zeroth-core-openapi.json"
      via: "diff against freshly generated spec"
      pattern: "--check"
---

<objective>
Render the `zeroth-core` FastAPI OpenAPI spec as an interactive Swagger UI page under `docs/reference/http-api.md`, and extend `scripts/dump_openapi.py` with a `--check` drift mode so CI can fail when the committed snapshot diverges from the live schema.

Purpose: Closes DOCS-08. Gives API consumers an interactive, always-current HTTP reference without requiring a running uvicorn.

Output: Updated `scripts/dump_openapi.py` (adds `--check`), new Swagger-UI-based `docs/reference/http-api.md`, a copy of the OpenAPI JSON under `docs/assets/openapi/` so mkdocs serves it as a static asset.
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

@scripts/dump_openapi.py
@docs/reference/http-api.md
@mkdocs.yml

<interfaces>
Existing `scripts/dump_openapi.py` main signature:
```python
def main() -> int:
    parser = argparse.ArgumentParser(...)
    parser.add_argument("--out", type=Path, default=None, ...)
    args = parser.parse_args()
    # constructs stub bootstrap, calls create_app, dumps app.openapi() as JSON
```

The committed snapshot lives at `openapi/zeroth-core-openapi.json` (sibling to `src/`, not under `docs/`). mkdocs `docs_dir` defaults to `docs/`, so files outside `docs/` are not served. Solution: copy the snapshot to `docs/assets/openapi/zeroth-core-openapi.json` and make that path the source-of-truth for the Swagger UI embed. Drift check in Plan 32-06 CI will also copy-or-diff to keep both in sync.
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Extend scripts/dump_openapi.py with --check drift mode + copy snapshot into docs/assets</name>
  <files>scripts/dump_openapi.py, docs/assets/openapi/zeroth-core-openapi.json</files>
  <action>
    Per D-02 and D-07:

    1. In `scripts/dump_openapi.py`, add a new argparse flag:
       ```python
       parser.add_argument(
           "--check",
           action="store_true",
           help="Exit 1 if --out file is missing or differs from freshly generated spec (drift check for CI).",
       )
       ```

    2. Update `main()`:
       - Generate `text` (spec JSON) as today.
       - If `args.check`:
         - Require `args.out is not None` (error otherwise: "--check requires --out").
         - If `args.out` does not exist → print "DRIFT: {out} does not exist" to stderr, return 1.
         - Read existing file; if content != generated text → print "DRIFT: {out} is stale. Run `python scripts/dump_openapi.py --out {out}` to update." to stderr, return 1.
         - Else print "OK: {out} is up to date." and return 0.
       - Else, keep existing write-or-print behavior unchanged.

    3. Add the file `docs/assets/openapi/zeroth-core-openapi.json` by copying `openapi/zeroth-core-openapi.json`:
       ```bash
       mkdir -p docs/assets/openapi
       cp openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
       ```
       This is the asset the Swagger UI embed will fetch. It is an exact duplicate of the root snapshot — Plan 32-06 will add a CI step that regenerates both and diffs them.

    4. Quick sanity run (positive case):
       `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json`
       should print "OK: ..." and exit 0.

    5. Quick sanity run (negative case):
       ```bash
       echo "{}" > /tmp/stale.json
       uv run python scripts/dump_openapi.py --check --out /tmp/stale.json
       ```
       should print "DRIFT: ..." and exit 1.
  </action>
  <verify>
    <automated>uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json && echo "{}" > /tmp/gsd-stale.json && ! uv run python scripts/dump_openapi.py --check --out /tmp/gsd-stale.json</automated>
  </verify>
  <done>`--check` flag implemented, exits 0 on up-to-date snapshot, exits 1 on drift, docs/assets/openapi/zeroth-core-openapi.json exists and matches root snapshot byte-for-byte.</done>
</task>

<task type="auto">
  <name>Task 2: Write Swagger UI embed page at docs/reference/http-api.md</name>
  <files>docs/reference/http-api.md</files>
  <action>
    Per D-02 (Claude's discretion: use static Swagger UI via CDN rather than neoteroi-mkdocs plugin — one fewer dependency, works under `mkdocs build --strict`, and `attr_list + md_in_html` are already enabled in mkdocs.yml).

    Replace the current single-line TBD content with:

    ````markdown
    # HTTP API Reference

    Interactive reference for the `zeroth-core` FastAPI service, rendered from the committed OpenAPI spec at [`openapi/zeroth-core-openapi.json`](https://github.com/rrrozhd/zeroth-core/blob/main/openapi/zeroth-core-openapi.json). The spec is regenerated from the FastAPI app on every commit via `scripts/dump_openapi.py`, and CI fails if it drifts.

    <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5.17.14/swagger-ui.css" />
    <div id="swagger-ui"></div>
    <script src="https://unpkg.com/swagger-ui-dist@5.17.14/swagger-ui-bundle.js" charset="UTF-8"></script>
    <script>
      window.addEventListener("load", function () {
        window.ui = SwaggerUIBundle({
          url: "../assets/openapi/zeroth-core-openapi.json",
          dom_id: "#swagger-ui",
          deepLinking: true,
          presets: [SwaggerUIBundle.presets.apis],
          layout: "BaseLayout",
        });
      });
    </script>

    ## Regenerating the spec

    The spec is a committed snapshot, not fetched live. To refresh it locally:

    ```bash
    uv run python scripts/dump_openapi.py --out openapi/zeroth-core-openapi.json
    cp openapi/zeroth-core-openapi.json docs/assets/openapi/zeroth-core-openapi.json
    ```

    CI runs `python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` on every PR and fails if the committed snapshot is stale.

    ## Offline consumption

    The raw JSON is available at [`/assets/openapi/zeroth-core-openapi.json`](../assets/openapi/zeroth-core-openapi.json) for tooling that wants to consume it directly (e.g., `openapi-typescript`, Postman import, ReDoc).
    ````

    Notes:
    - `md_in_html` + `attr_list` are already in `markdown_extensions` (confirmed in mkdocs.yml), so raw HTML inside markdown works under `--strict`.
    - The CDN URLs are pinned (`@5.17.14`) for reproducibility.
    - Relative path `../assets/openapi/zeroth-core-openapi.json` resolves correctly from `reference/http-api.md` to `assets/openapi/zeroth-core-openapi.json` in the built site.
  </action>
  <verify>
    <automated>uv run mkdocs build --strict && test -f site/reference/http-api/index.html && grep -q "swagger-ui" site/reference/http-api/index.html && test -f site/assets/openapi/zeroth-core-openapi.json</automated>
  </verify>
  <done>docs/reference/http-api.md renders under strict mkdocs build, produced HTML contains Swagger UI script tags, and the OpenAPI JSON asset is copied into `site/assets/openapi/` by mkdocs.</done>
</task>

</tasks>

<verification>
- `uv run python scripts/dump_openapi.py --check --out openapi/zeroth-core-openapi.json` exits 0
- Deliberately stale file causes `--check` to exit 1 with "DRIFT:" message
- `uv run mkdocs build --strict` passes
- `docs/assets/openapi/zeroth-core-openapi.json` exists and equals `openapi/zeroth-core-openapi.json` byte-for-byte
- `site/reference/http-api/index.html` contains `swagger-ui-bundle.js` script tag
</verification>

<success_criteria>
DOCS-08 satisfied: HTTP API Reference is rendered from the `zeroth-core` FastAPI OpenAPI spec and published alongside the Python reference on the docs site. Drift is gated in CI.
</success_criteria>

<output>
After completion, create `.planning/phases/32-reference-docs-deployment-migration-guide/32-02-SUMMARY.md`
</output>
