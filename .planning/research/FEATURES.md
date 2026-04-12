# Feature Research

**Domain:** Python library packaging + in-depth documentation + frontend repo split (v3.0 Core Library Extraction, Studio Split & Documentation)
**Researched:** 2026-04-10
**Confidence:** HIGH (exemplars are well-known, actively maintained, and their documentation patterns are stable industry practice)

## Research Scope

This milestone is a packaging and documentation milestone for an existing mature Python backend (~22K LOC source, ~18K LOC tests, 280+ tests, 20+ subsystems). Feature research focuses exclusively on the NEW deliverables:

1. `zeroth-core` PyPI-installable library structure & metadata
2. In-depth documentation for every subsystem
3. `zeroth-studio` frontend repo separation
4. Migration support for users of the monolithic repo

Existing backend features (graph, orchestrator, agents, memory, etc.) are NOT re-researched.

## Exemplars Studied

| Exemplar | What We Learned | Link |
|----------|----------------|------|
| **FastAPI** | Tutorial-first narrative docs with progressive disclosure; hand-written reference; multi-language scaffolding; Swagger/ReDoc for HTTP API | [fastapi.tiangolo.com](https://fastapi.tiangolo.com/) |
| **Pydantic** | Unified docs site with strong "Concepts" section + auto-generated API reference; version dropdown; migration guide as first-class page | [docs.pydantic.dev](https://docs.pydantic.dev/latest/) |
| **SQLAlchemy** | Narrative Unified Tutorial + massive reference; explicitly treats reference as primary artifact; unified 1.4/2.0/2.1 tutorial reuse | [docs.sqlalchemy.org](https://docs.sqlalchemy.org/en/20/) |
| **LangChain** | Explicit adoption of Diátaxis (Tutorials / How-to / Conceptual / API Reference); [doc refresh blog post](https://blog.langchain.com/langchain-documentation-refresh/) | [python.langchain.com](https://python.langchain.com/) |
| **Django** | Pioneered the four-doc-types-per-topic model that became Diátaxis; topic-oriented TOC | (referenced in Diátaxis origin) |
| **Typer** | Tightly-scoped Getting Started: install → first command → progressive tutorial stepping through features in a single linear flow | typer.tiangolo.com |
| **Diátaxis framework** | Tutorials / How-to Guides / Reference / Explanation as orthogonal documentation quadrants — THE dominant IA pattern | [diataxis.fr](https://diataxis.fr/) |
| **mkdocstrings + Griffe** | Auto-generate API reference from docstrings; supports Google/NumPy/Sphinx styles; cross-references between narrative and reference | [mkdocstrings.github.io](https://mkdocstrings.github.io/) |
| **LibCST RenameCommand** | Automated codemod for package renames (e.g. `zeroth.*` → `zeroth.core.*`); rewrites imports in user codebases | [libcst.readthedocs.io](https://libcst.readthedocs.io/en/latest/codemods.html) |

## Common Pattern Extracted (The "Standard Shape")

Every exemplar converges on roughly this top-level IA:

```
1. Home / Why [Library]          (one-paragraph pitch + install + 10-line example)
2. Getting Started / Tutorial    (progressive, ONE linear path, ~30 minutes end-to-end)
3. Concepts / Explanation        (what it is, why it exists, mental model per subsystem)
4. How-to Guides / Recipes       (task-oriented, "how do I X")
5. API Reference                 (auto-generated, module-by-module, cross-linked)
6. Deployment / Operations       (production concerns)
7. Migration                     (from prior versions / prior names)
8. Contributing                  (for library developers)
9. Release notes / Changelog
```

Zeroth-specific additions (because it is simultaneously a library AND an HTTP service):

```
- HTTP API Reference             (alongside Python API Reference — OpenAPI-sourced)
- Service Mode guide             (running as a FastAPI service vs embedding as a library)
- Governance guide               (approvals, audit, policy — Zeroth's differentiator)
```

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features whose absence makes a 0.1.0 library release feel broken or unusable.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **`pip install zeroth-core` works and resolves cleanly** | Non-negotiable for "pip-installable library" | MEDIUM | PEP 621 `pyproject.toml` with optional-dependencies extras (`[memory-pg]`, `[memory-chroma]`, `[dispatch]`, `[all]`). Requires `econ-instrumentation-sdk` on PyPI first. |
| **PEP 420 namespace package `zeroth.core.*`** | Decision already logged; leaves room for `zeroth.studio`, `zeroth.ext.*` siblings | MEDIUM | No top-level `zeroth/__init__.py`. Entire rename is a rename-only (no re-export glue), per design decision. |
| **Landing page with <10-line hello world** | Every exemplar does this; users decide in 30 seconds | SMALL | Example: build trivial graph, run it, print result. Must be copy-pasteable, must actually run. |
| **Getting Started tutorial (single linear path)** | Typer/FastAPI pattern; Getting Started is NOT a menu | MEDIUM | 3 sections: (1) install + 10-line hello, (2) first graph with one agent + one tool + LLM call, (3) run in service mode. ~30 min end-to-end. First working output in <5 minutes. |
| **Concept pages for every major subsystem** | LangChain "Conceptual Guides"; SQLAlchemy narrative | LARGE | ~20 subsystems: graph, orchestrator, agents, execution units, memory, contracts, runs, conditions, mappings, policy, approvals, audit, secrets, identity, guardrails, dispatch, economics, storage, service, threads. Each is an Explanation-quadrant page: what it is, why it exists, mental model, where it fits. ~500-1500 words each. |
| **Per-subsystem usage guide** | Pydantic per-concept pages | LARGE | For each subsystem: Overview → Core Concepts → Minimal Example → Common Patterns → Pitfalls → Reference cross-link. Mirrors LangChain's structure. |
| **Auto-generated Python API reference via mkdocstrings** | Django/Pydantic/SQLAlchemy all have one; hand-writing reference for 22K LOC is impractical | MEDIUM | Use `mkdocstrings[python]` + Griffe. Requires docstring coverage sweep across public surface — flag for PITFALLS. |
| **HTTP API reference (OpenAPI-sourced)** | Zeroth ships a FastAPI service; users need both embeddings AND service docs | SMALL | Already have OpenAPI spec. Render via `neoteroi-mkdocs` swagger/redoc plugin or serve `/docs` endpoint and link. |
| **Install guide with all optional extras documented** | Zeroth has many optional backends (pgvector, Chroma, Elasticsearch, ARQ, Docker sandbox) | SMALL | Matrix: "I want to use X → install `zeroth-core[x]`". |
| **Configuration reference** | pydantic-settings drives everything; users will hit it immediately | SMALL | Every env var, every default, every secret. Auto-generatable from pydantic-settings schema. |
| **Deployment guide** | Docker-compose and Nginx TLS ship in the repo; users need to know how to stand it up | MEDIUM | Cover: local dev, docker-compose, standalone service, embedded in host app, with/without Regulus companion, with/without Postgres. |
| **Migration guide from monolith → `zeroth-core`** | Existing users (even internal) have `from zeroth.graph import ...` imports | MEDIUM | Single page covering (a) the rename pattern, (b) the LibCST codemod (or sed-level equivalent), (c) econ SDK path swap, (d) env var changes if any, (e) Docker image retag. |
| **Changelog / release notes** | PyPI releases need them; [keepachangelog.com](https://keepachangelog.com/) format | SMALL | `CHANGELOG.md` at repo root + rendered on docs site. |
| **`zeroth-studio` in its own public repo with independent CI** | Decision already logged | MEDIUM | Repo: `rrrozhd/zeroth-studio`. Own `package.json`, own CI, own release tags. Consumes `zeroth-core` via HTTP only. |
| **Studio README pointing at `zeroth-core`** | Discoverability: devs finding Studio first must know where the backend lives | SMALL | Cross-link both READMEs. |
| **Archive of monolithic repo (tarball + bare mirror + GitHub `rrrozhd/zeroth-archive`)** | Decision already logged; mostly done ad-hoc | SMALL | Finalize the ad-hoc archive, document where each layer lives, add a "this repo is archived" notice. |
| **License + CONTRIBUTING files** | PyPI convention; users check before depending | SMALL | Reuse existing. |
| **Docs site deployed on every commit to main** | Industry baseline (mkdocs gh-deploy) | SMALL | GitHub Actions workflow, publish to `gh-pages` or Cloudflare Pages. |

### Differentiators (Competitive Advantage)

Features that would make Zeroth's docs noticeably better than a median Python library's docs, aligned with the "governance-first" core value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Diátaxis-explicit IA** | LangChain adopted this and wrote a blog post about the improvement. Users find what they need faster. Reviewers/contributors know where new content goes. | SMALL (IA, not content) | Label every page as Tutorial / How-to / Concept / Reference in frontmatter. Four top-level TOC entries, not 15. |
| **Governance walkthrough tutorial** | This is Zeroth's actual differentiator vs LangGraph/CrewAI/Semantic Kernel. A tutorial that shows an approval gate stopping execution, an auditor reviewing the trail, and a policy blocking a tool call is unique. | MEDIUM | ~2000 words + runnable example. Tutorial-quadrant content. |
| **Recipes cookbook** (cross-subsystem how-tos) | Answers "how do I do X" where X spans 3-4 subsystems | MEDIUM | Examples: "add a human approval step to an agent tool call", "attach a pgvector memory to a graph node", "cap a run's budget with Regulus", "sandbox a Python execution unit", "retry a failed webhook from the DLQ". 10-15 recipes at launch. |
| **Runnable examples directory** (`examples/` in repo) | FastAPI does this; examples stay in sync because CI runs them | MEDIUM | Each example is a complete, minimal, runnable Python file with a README. CI smoke-tests them. Link inline from docs pages. |
| **LibCST codemod for the rename** | Automates migration; dramatically lowers upgrade friction | MEDIUM | `python -m zeroth.core.codemods.rename_from_monolith <path>`. Wraps `libcst.codemod.commands.rename.RenameCommand`. Worth it because the rename pattern is mechanical. |
| **"Choose your path" landing page** | Library-vs-service is genuinely confusing for Zeroth (it is both) | SMALL | Two prominent cards on home: "Embed as a library" vs "Run as a governed API service". Each links to a different Getting Started path. |
| **Per-subsystem Pitfalls callouts** | Zeroth has 20 subsystems with real production footguns (sandbox timeouts, approval SLAs, memory retention, economic budget overshoot) | MEDIUM | Inline `!!! warning` callouts on each subsystem page. Drawn directly from our PITFALLS.md research. |
| **HTTP API + Python API side-by-side** | Every subsystem has both a Python interface and an HTTP interface; showing both on one page is unusual and valuable | MEDIUM | Tabbed code blocks: `Python` / `HTTP` / `curl`. Forces us to verify parity and flags gaps. |
| **"What's indexed vs not" docstring coverage badge** | Honest signal to users about which parts are well-documented | SMALL | `interrogate` or `docstr-coverage` in CI, badge in README. |
| **Documented extension points** | Zeroth has several (memory connectors, LLM providers, execution units, judges) — documenting them unlocks the ecosystem | MEDIUM | "Writing a custom memory connector" / "Writing a custom execution unit" / etc. How-to quadrant. |
| **Cross-repo version compatibility matrix** (zeroth-core × zeroth-studio) | Prevents "I upgraded Studio and it broke" issues | SMALL | Single table: "zeroth-studio 0.2.x works with zeroth-core 0.1.x-0.3.x". Maintained in both repos. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that sound obviously good but waste effort or create maintenance burden for a 0.1.0 release.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Multi-version docs site (v0.1 / v0.2 / latest dropdown)** | "Mature libraries have it" | Pre-1.0, API is still shifting. `mike` plugin adds CI complexity. Stale old versions confuse more than help. Pydantic does this but they are v2.12 with real users on v1. | Ship only `latest`. Add version dropdown at 1.0. Until then, link to git tags for historical docs. |
| **Documentation in multiple languages** | "FastAPI does it" | FastAPI has 12 language maintainers. We have one team. Translations rot faster than code. | English only. Accept PRs for translations later if a maintainer emerges. |
| **Hand-written API reference for every class** | "Higher quality than auto-generated" | 22K LOC. Would take months. Goes out of sync on day one. Every exemplar except tiny libraries uses auto-generation. | mkdocstrings + Griffe. Hand-write only the curated narrative pages. |
| **Interactive Jupyter notebook examples** | "Jupyter is popular for AI" | Notebooks are hard to version-control, hard to diff, hard to CI-test, and get stale. LangChain has notebook cookbooks and [their own blog post](https://blog.langchain.com/langchain-documentation-refresh/) admits this is a pain point. | Plain `.py` files in `examples/` directory with README. CI-runnable. Copy-pasteable into notebooks if users want. |
| **Video tutorials / screencasts** | "Accessibility, different learning styles" | Massively expensive to produce, impossible to update in-place when API changes, requires separate hosting, SEO/search unfriendly. | Written tutorials only. Community members can produce videos later. |
| **Public Discord/Slack support channel at launch** | "Community building" | Pre-1.0 libraries drown their own maintainers in live support. Sets expectation of responsiveness we can't meet. | GitHub Issues + GitHub Discussions only. Discord/Slack post-1.0. |
| **"Awesome Zeroth" curated extension list** | "Ecosystem growth" | There is no ecosystem yet. Premature. Will sit empty and look abandoned. | Revisit at 1.0 when third-party extensions exist. |
| **Auto-generated CLI reference from `--help` output** | "Nice consistency" | Zeroth is primarily a library + HTTP service, not a CLI tool. Small CLI surface. Manual page is simpler and stays current. | Hand-written CLI page covering the ~5 commands. |
| **Docs-as-code in RST + Sphinx** | "Sphinx is the Python standard" | Author is more productive in Markdown. mkdocs-material rendering is better out-of-box. **Caveat:** mkdocs-material is moving to "minimal maintenance" in late 2026 per [squidfunk](https://squidfunk.github.io/mkdocs-material/alternatives/); Zensical is the successor (still alpha). | Use mkdocs-material now, re-evaluate migration path in 2026 Q4. Sphinx is the safer LONG-term bet if we want to avoid a future migration. **Flag for STACK.md decision.** |
| **Live documentation search via Algolia DocSearch** | "FastAPI has it" | Requires Algolia approval and setup. Local Lunr search is free and sufficient for a 0.1.0 site. | Built-in mkdocs search. Algolia later if traffic warrants. |
| **Monorepo with `zeroth-core` + `zeroth-studio` together** | "Simpler CI", "atomic commits" | Directly contradicts the existing design decision to split. Would re-couple release cadences we already decided to uncouple. | Honor the split decision. Two repos. Cross-repo contracts via HTTP API + OpenAPI schema. |
| **Real-time docs preview with editing** | "GitBook is nicer" | GitBook costs money, locks content in their platform, harder to git-review. | mkdocs serve + PR preview deploys (Cloudflare Pages or Netlify) gives 90% of the benefit. |
| **Porting every subsystem guide to both Python and HTTP simultaneously at v0.1** | "Parity is nice" | Doubles initial effort. Can ship Python narrative first and add HTTP tabs incrementally. | Python narrative first for all 20 subsystems. HTTP tabs added in a follow-up phase. |

---

## Feature Dependencies

```
PyPI package published (`zeroth-core` + `econ-instrumentation-sdk`)
    └──required by──> Install guide can be tested end-to-end
    └──required by──> Getting Started tutorial can work
    └──required by──> docs site "pip install" copy-paste is honest

Rename complete (`zeroth.*` → `zeroth.core.*`)
    └──required by──> Auto-generated API reference (mkdocstrings paths)
    └──required by──> Migration guide (can reference real new paths)
    └──required by──> LibCST codemod (targets the real paths)

Docstring coverage sweep on public surface
    └──required by──> Auto-generated API reference is useful
    └──required by──> Per-subsystem usage guide can cross-link to reference

Concept pages (~20 subsystems)
    └──required by──> Per-subsystem usage guides (usage guides reference concepts)
    └──required by──> Recipes cookbook (recipes cite concepts)

Getting Started tutorial
    └──required by──> Landing page (landing links to Getting Started)
    └──required by──> Governance walkthrough (reuses Getting Started's setup)

Runnable examples in `examples/`
    └──enhances──> Concept pages (inline snippets link to full examples)
    └──enhances──> Recipes cookbook (recipes ARE examples + prose)
    └──required by──> CI smoke test for examples

HTTP API reference
    └──requires──> OpenAPI spec is current (already true)
    └──enhances──> Per-subsystem usage guide (HTTP tab)

Studio repo split
    └──required by──> Cross-repo compatibility matrix
    └──required by──> Studio README cross-links
    └──blocks──> Studio phases 24-26 development (they move to new repo)

Archive of monolith
    └──required by──> Migration guide's "old repo is archived" notice
    └──mostly done ad-hoc──> Needs formalization only
```

### Dependency Notes

- **API reference requires docstring coverage:** Auto-generated reference is only as good as the docstrings. A coverage sweep (target: 95% on public surface) is a hard prerequisite. Use `interrogate` or `docstr-coverage` in CI.
- **Rename must land before docs writing:** Writing concept pages against `zeroth.graph` and then globally rewriting to `zeroth.core.graph` wastes editor time. Sequence: rename → docstring sweep → narrative docs.
- **PyPI publish must land before Getting Started can be tested:** The `pip install zeroth-core` copy-paste in Getting Started is a lie until the package exists on PyPI. TestPyPI is acceptable for pre-release validation.
- **`econ-instrumentation-sdk` must publish before `zeroth-core`:** It's a hard dependency (currently a local file path). Sequence: econ SDK to PyPI → zeroth-core to PyPI.
- **Studio split blocks v2.0 phases 24-26:** Those phases can only resume once `zeroth-studio` has its own repo, CI, and working dev loop against a `zeroth-core` API.

---

## MVP Definition

### Launch With (v0.1.0 of `zeroth-core` + `zeroth-studio` split + v0.1 docs)

Minimum viable product — what's needed for the milestone to be "done" and the library to be usable by a first external user.

**Library packaging:**
- [ ] `econ-instrumentation-sdk` published to PyPI (blocks everything else)
- [ ] `zeroth-core` published to PyPI with `zeroth.core.*` namespace
- [ ] `pyproject.toml` with optional-dependency extras documented
- [ ] `CHANGELOG.md` + license + contributing files at repo root

**Documentation (Diátaxis-organized):**
- [ ] Landing page with 10-line hello world + install + "choose your path" cards
- [ ] Getting Started: 3-section linear tutorial (install → hello graph → run in service mode), <30 min end-to-end, first output in <5 min
- [ ] Concept pages for all ~20 subsystems (~500-1500 words each)
- [ ] Usage guides for all ~20 subsystems (Overview → Example → Patterns → Pitfalls → Reference link)
- [ ] Governance walkthrough tutorial (Zeroth's differentiator)
- [ ] Auto-generated Python API reference via mkdocstrings
- [ ] HTTP API reference (rendered from OpenAPI)
- [ ] Configuration reference (auto-generated from pydantic-settings)
- [ ] Deployment guide (local / docker-compose / service mode / embedded)
- [ ] Migration guide (monolith → `zeroth.core.*`)
- [ ] 10-15 recipes in a Cookbook section
- [ ] `examples/` directory with CI-tested runnable examples
- [ ] Docs site deployed on every main commit

**Studio split:**
- [ ] `rrrozhd/zeroth-studio` public repo created
- [ ] Vue 3 frontend moved with history preserved
- [ ] Independent CI passing
- [ ] README cross-links both ways
- [ ] Cross-repo compatibility matrix documented

**Archive:**
- [ ] Ad-hoc monolith archive formalized (tarball, bare mirror, `rrrozhd/zeroth-archive`)
- [ ] Archive notice on old repo

### Add After Validation (v0.2.x)

Features to add once first external users have validated the core library.

- [ ] LibCST codemod for the rename (only worth building if users report mechanical migration pain)
- [ ] HTTP API tabs added inline to every subsystem usage guide (currently Python-only)
- [ ] Extension point guides (custom memory connector, custom LLM provider, custom execution unit)
- [ ] Algolia DocSearch (only if traffic warrants)
- [ ] Docstring coverage badge in README
- [ ] Governance case studies (real-world workflows with audit trails)

### Future Consideration (v1.0+)

- [ ] Multi-version docs site (`mike` plugin) — only once there are enough releases to matter
- [ ] Translations — only if maintainers emerge
- [ ] Video tutorials / screencasts
- [ ] Discord/Slack community channel
- [ ] "Awesome Zeroth" curated list
- [ ] Monorepo reconsolidation evaluation (unlikely; split is intentional)
- [ ] Migration off mkdocs-material to Zensical or Sphinx (forced by 2026 Q4 mkdocs-material maintenance change)

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| `zeroth-core` on PyPI | HIGH | MEDIUM | P1 |
| `econ-instrumentation-sdk` on PyPI | HIGH (blocks above) | LOW | P1 |
| Landing page + hello world | HIGH | LOW | P1 |
| Getting Started (3 sections) | HIGH | MEDIUM | P1 |
| Concept pages × 20 subsystems | HIGH | HIGH | P1 |
| Usage guides × 20 subsystems | HIGH | HIGH | P1 |
| Auto-generated Python API reference | HIGH | MEDIUM | P1 |
| HTTP API reference | MEDIUM | LOW | P1 |
| Migration guide | HIGH | MEDIUM | P1 |
| Configuration reference | HIGH | LOW | P1 |
| Deployment guide | HIGH | MEDIUM | P1 |
| Recipes cookbook (10-15) | MEDIUM | MEDIUM | P1 |
| `examples/` dir with CI tests | MEDIUM | MEDIUM | P1 |
| `zeroth-studio` repo split | HIGH | MEDIUM | P1 |
| Archive formalization | LOW | LOW | P1 (cleanup) |
| Governance walkthrough tutorial | HIGH (differentiator) | MEDIUM | P1 |
| Docstring coverage sweep | MEDIUM (enables reference quality) | MEDIUM | P1 |
| Cross-repo compatibility matrix | MEDIUM | LOW | P1 |
| LibCST codemod | MEDIUM | MEDIUM | P2 |
| HTTP/Python side-by-side tabs | MEDIUM | MEDIUM | P2 |
| Extension point guides | MEDIUM | MEDIUM | P2 |
| Docstring coverage badge | LOW | LOW | P2 |
| Multi-version docs (`mike`) | LOW (pre-1.0) | MEDIUM | P3 |
| Translations | LOW | HIGH | P3 |
| Video tutorials | LOW | HIGH | P3 |
| Discord/Slack community | LOW (pre-1.0) | HIGH (support burden) | P3 |

**Priority key:**
- **P1:** Must have for v3.0 milestone completion
- **P2:** Should have, add in v0.2.x iterations
- **P3:** Defer until post-1.0

---

## Exemplar Feature Analysis

| Feature | FastAPI | Pydantic | LangChain | SQLAlchemy | Our Approach |
|---------|---------|----------|-----------|------------|--------------|
| **IA framework** | Progressive narrative tutorial | Concepts + Reference | Explicit Diátaxis | Unified Tutorial + Reference | Explicit Diátaxis (LangChain pattern) |
| **Getting Started length** | Long progressive tutorial | Short + "Why Pydantic" | Short tutorial + "Quickstart" | Long narrative | **Short linear 3-section** (Typer pattern) |
| **API reference style** | Hand-written references | Auto-generated | Auto-generated | Mostly auto + curated | **Auto-generated via mkdocstrings** |
| **Examples location** | Inline + separate tutorial repo | Inline | `cookbook/` notebooks + inline | Inline | **`examples/` `.py` files + inline** (FastAPI pattern, minus tutorial repo) |
| **Multi-version docs** | Yes | Yes (v1 vs v2 dropdown) | Yes (v0.1 / v0.2 / latest) | Yes (1.4 / 2.0 / 2.1) | **No** (pre-1.0 — defer) |
| **Migration guide** | Not applicable | First-class page (v1→v2) | First-class page (v0.1→v0.2) | Migration section per version | **First-class page** (monolith → core) |
| **Doc engine** | MkDocs Material | MkDocs Material | Docusaurus (TS-based) | Sphinx | **MkDocs Material** (with Zensical migration flag) |
| **Cookbook/recipes** | "Advanced User Guide" | N/A | Large cookbook | Distributed in reference | **Dedicated Cookbook section** (LangChain pattern) |
| **Translations** | 12 languages | English only | English only | English only | **English only** |
| **Frontend UI separation** | N/A | N/A | LangSmith is separate closed-source product | N/A | `zeroth-studio` separate public repo |

---

## Information Architecture Recommendation (for ARCHITECTURE.md / REQUIREMENTS.md)

```
zeroth-core docs site
├── Home                                  (Explanation + Tutorial entry)
│   ├── 10-line hello world
│   ├── Install
│   └── "Choose your path" (library vs service)
├── Getting Started                       (Tutorial quadrant)
│   ├── 1. Install & first graph
│   ├── 2. Add an agent with an LLM and a tool
│   └── 3. Run in service mode with an approval gate
├── Concepts                              (Explanation quadrant)
│   ├── Why Zeroth / governance model
│   ├── Graph authoring
│   ├── Orchestrator runtime
│   ├── Agents & LLM providers
│   ├── Execution units & sandbox
│   ├── Memory connectors
│   ├── Contracts registry
│   ├── Runs, threads, conditions, mappings
│   ├── Policy, approvals, audit
│   ├── Secrets & identity
│   ├── Guardrails
│   ├── Dispatch & workers
│   ├── Economics (Regulus)
│   ├── Storage backends
│   └── Service mode
├── Guides                                (How-to quadrant)
│   ├── Per-subsystem usage guides (~20)
│   ├── Deployment (local / docker-compose / service / embedded)
│   ├── Extension points (custom memory / provider / execution unit) [v0.2]
│   └── Governance walkthrough
├── Cookbook / Recipes                    (How-to quadrant)
│   └── 10-15 cross-subsystem recipes
├── Reference                             (Reference quadrant)
│   ├── Python API (auto-generated, mkdocstrings)
│   ├── HTTP API (OpenAPI-rendered)
│   ├── Configuration (pydantic-settings-sourced)
│   └── CLI
├── Migration                             (Reference quadrant)
│   └── Monolith → `zeroth.core.*`
├── Changelog
└── Contributing
```

Total initial pages: roughly **80-100** counting all per-subsystem concept + guide pairs, recipes, and reference stubs.

---

## Open Questions for REQUIREMENTS.md

1. **Doc engine decision:** mkdocs-material now vs Sphinx for longevity? Flag the mkdocs-material "minimal maintenance in late 2026" risk in STACK.md.
2. **Docstring style:** Google-style, NumPy-style, or Sphinx-style? Must be consistent for mkdocstrings. (Recommendation: Google-style — most readable in source, well-supported by Griffe.)
3. **Docstring coverage target:** 95% on public surface? 100%? `interrogate` threshold?
4. **Recipe count for v0.1:** 10? 15? 20? (Recommendation: 10 for launch, grow organically.)
5. **Examples `.py` vs notebooks:** Confirm `.py` files (recommended) — notebooks are a documented anti-feature above.
6. **Migration guide scope:** Does it also cover deployments (Docker image retag, env var names) or only Python imports?
7. **Cross-repo compat matrix ownership:** Lives in `zeroth-core` docs, `zeroth-studio` docs, or both?
8. **Codemod priority:** Build LibCST codemod in v0.1 or defer to v0.2 once we hear user migration pain?

---

## Sources

- [Diátaxis framework (diataxis.fr)](https://diataxis.fr/) — the dominant documentation IA pattern
- [LangChain documentation refresh blog](https://blog.langchain.com/langchain-documentation-refresh/) — explicit Diátaxis adoption retrospective
- [LangChain Documentation Style Guide](https://python.langchain.com/v0.2/docs/contributing/documentation/style_guide/)
- [FastAPI official docs](https://fastapi.tiangolo.com/) — tutorial-first narrative pattern
- [Pydantic docs](https://docs.pydantic.dev/latest/) — concepts + auto-generated reference pattern
- [SQLAlchemy Unified Tutorial](https://docs.sqlalchemy.org/en/20/) — narrative + reference split
- [mkdocstrings overview](https://mkdocstrings.github.io/) — auto-generated API reference tooling
- [mkdocstrings-python usage](https://mkdocstrings.github.io/python/usage/)
- [Real Python: Build Python docs with MkDocs](https://realpython.com/python-project-documentation-with-mkdocs/)
- [Switching From Sphinx to MkDocs (Towards Data Science)](https://towardsdatascience.com/switching-from-sphinx-to-mkdocs-documentation-what-did-i-gain-and-lose-04080338ad38/)
- [Material for MkDocs — Alternatives page (Zensical transition notice)](https://squidfunk.github.io/mkdocs-material/alternatives/)
- [LibCST codemods tutorial](https://libcst.readthedocs.io/en/latest/codemods_tutorial.html)
- [LibCST RenameCommand source](https://github.com/Instagram/LibCST/blob/main/libcst/codemod/commands/rename.py)
- [Keep a Changelog](https://keepachangelog.com/)
- [Scientific Python Development Guide — Writing documentation](https://learn.scientific-python.org/development/guides/docs/)

---
*Feature research for: v3.0 Core Library Extraction, Studio Split & Documentation*
*Researched: 2026-04-10*
*Confidence: HIGH — exemplar patterns are stable, well-known industry practice; only open uncertainty is the 2026 Q4 mkdocs-material maintenance transition.*
