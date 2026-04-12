# Requirements: Zeroth

**Defined:** 2026-04-09 (v2.0) · **Updated:** 2026-04-12 (v4.0 milestone)
**Core Value:** Teams can author and operate governed multi-agent workflows without sacrificing production controls, auditability, or deployment rigor.

## v4.0 Requirements — Platform Extensions for Production Agentic Workflows

v4.0 closes 7 architectural gaps identified during a production adoption audit comparing zeroth-core against real-world LangGraph migration requirements. All extensions preserve existing test coverage, integrate with the governance stack, and maintain backward compatibility.

### Parallel Execution (PARA)

- [ ] **PARA-01**: A node can spawn N parallel execution branches from its output, where each branch receives a slice of the output (e.g., one item from a list)
- [ ] **PARA-02**: A synchronization barrier waits for all spawned branches to complete before proceeding, collecting branch outputs into an aggregated payload with deterministic ordering (by branch index)
- [ ] **PARA-03**: Each parallel branch has its own isolated execution context (visit counts, audit trail, failure tracking) — a failure in one branch does not automatically fail others (configurable: fail-fast vs best-effort)
- [ ] **PARA-04**: Policy enforcement, audit recording, and contract validation apply independently per branch, each producing its own audit records
- [ ] **PARA-05**: Cost attribution tracks per-branch spend; BudgetEnforcer is consulted before spawning branches with a pre-reservation of total estimated cost across N branches
- [ ] **PARA-06**: ExecutionSettings guardrails (max_total_steps, max_visits_per_node) account for parallel branches as sum across branches, not per-branch

### Subgraph Composition (SUBG)

- [ ] **SUBG-01**: A graph can reference another published graph as a subgraph node; the subgraph's entry contract must be compatible with the referencing edge's mapping output
- [ ] **SUBG-02**: When the orchestrator reaches a subgraph node, it enters the referenced graph as a nested scope; the subgraph's final output maps back to the parent graph's expected input for the next node
- [ ] **SUBG-03**: Subgraph execution shares the parent's thread_id; agents inside subgraphs can participate in the same thread memory (configurable per-node via thread_participation)
- [ ] **SUBG-04**: The parent graph's policies apply as a baseline; the subgraph can further restrict but not relax capabilities; audit records link to the parent run via parent_run_id
- [ ] **SUBG-05**: If a HumanApprovalNode inside a subgraph pauses execution, the parent run transitions to WAITING_APPROVAL; resolution resumes the subgraph and eventually the parent
- [ ] **SUBG-06**: The same subgraph definition can be referenced by multiple parent graphs and at multiple points within a single parent graph
- [ ] **SUBG-07**: Subgraph references can pin to a specific deployment version or float to the latest active deployment
- [ ] **SUBG-08**: Nested subgraphs (subgraph within a subgraph) are supported with a configurable depth limit

### Large Payload Externalization (ARTF)

- [ ] **ARTF-01**: A pluggable artifact store interface exists for large payloads, separate from run state, with implementations for Redis and local filesystem
- [ ] **ARTF-02**: Nodes can emit an ArtifactReference (store, key, content_type, size) as part of their output; the reference is stored in run history while the payload lives in the artifact store
- [ ] **ARTF-03**: Artifacts support configurable TTL; artifacts tied to a run are cleanable when the run is archived
- [ ] **ARTF-04**: Audit records log artifact references (not full payloads); the audit evidence export can optionally resolve references to full payloads
- [ ] **ARTF-05**: Contracts support an ArtifactReference type that validates the reference structure without requiring the full payload at validation time

### Context Window Management (CTXW)

- [ ] **CTXW-01**: Approximate token count of accumulated agent messages per thread is tracked using the LLM provider's tokenizer (via litellm.token_counter)
- [ ] **CTXW-02**: When token count exceeds a configurable threshold, a compaction strategy is applied before the next LLM invocation (default: observation masking of older messages)
- [ ] **CTXW-03**: Compaction strategy is pluggable per agent node (built-in: truncation, observation masking, LLM-based summarization)
- [ ] **CTXW-04**: Compaction results are stored in thread memory so they persist across runs; original messages can optionally be archived for audit
- [ ] **CTXW-05**: Per-agent-node settings are configurable: max_context_tokens, summary_trigger_ratio, compaction_strategy, preserve_recent_messages_count

### Resilient HTTP Client (HTTP)

- [ ] **HTTP-01**: A platform-provided async HTTP client is available to agent tools and executable units, configurable per-node or per-tool
- [ ] **HTTP-02**: Configurable retry with exponential backoff and jitter; retryable status codes configurable (default: 408, 429, 5xx)
- [ ] **HTTP-03**: Per-endpoint circuit breaker with configurable failure threshold and reset timeout
- [ ] **HTTP-04**: Shared or per-tenant connection pools with configurable limits
- [ ] **HTTP-05**: External HTTP calls are gated by NETWORK_READ / NETWORK_WRITE / EXTERNAL_API_CALL capabilities, logged in audit records (URL, method, status, latency), and subject to rate limiting
- [ ] **HTTP-06**: HTTP client resolves auth headers/tokens from the SecretResolver automatically based on configuration

### Prompt Template Management (TMPL)

- [ ] **TMPL-01**: A template registry stores and versions prompt templates by name, analogous to the contract registry
- [ ] **TMPL-02**: Templates support variable interpolation from node input, run state, or memory using Jinja2 SandboxedEnvironment
- [ ] **TMPL-03**: Agent nodes can reference a template by name+version instead of providing a raw instruction string; template is rendered at runtime
- [ ] **TMPL-04**: The rendered prompt (post-interpolation) is available in audit records; template variables containing secrets are redacted

### Computed Data Mappings (XFRM)

- [ ] **XFRM-01**: A new transform mapping operation evaluates an expression against the source payload and writes the result to the target path
- [ ] **XFRM-02**: Transform expressions use the same expression evaluation engine as conditions (the existing _SafeEvaluator) and can access payload.*, state.*, variables.*
- [ ] **XFRM-03**: The output of a transform expression is validated against the target node's input contract
- [ ] **XFRM-04**: Transform expressions are guaranteed side-effect-free (no network, no filesystem, no imports); the expression evaluator enforces this with hardened namespace restrictions

---

## v3.0 Requirements — Core Library Extraction, Studio Split & Documentation (shipped)

v3.0 is a packaging and documentation milestone. No new runtime features. Deliverables: ship `zeroth-core` on PyPI under the `zeroth.core.*` namespace, move the Vue Studio to its own public repo, write in-depth documentation for every subsystem, and formalize the monolith archive.

### Packaging (PKG)

- [ ] **PKG-01**: `econ-instrumentation-sdk` (Regulus SDK) is published to PyPI with a stable version, replacing the current local file-path dependency
- [ ] **PKG-02**: `zeroth-core` is published to PyPI as a pip-installable library and `pip install zeroth-core` succeeds in a clean virtualenv
- [ ] **PKG-03**: `pyproject.toml` declares optional-dependency extras for every swappable backend (`[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]`), with each extra verified installable and documented
- [ ] **PKG-04**: Repository root contains `CHANGELOG.md` (keepachangelog format), `LICENSE`, and `CONTRIBUTING.md` suitable for a public PyPI release
- [ ] **PKG-05**: PyPI releases are published via GitHub Actions using trusted publisher (OIDC), not a long-lived API token
- [ ] **PKG-06**: Installing `zeroth-core` from a clean environment and running the Getting Started hello example works end-to-end (acceptance test for the whole packaging stack)

### Namespace Rename (RENAME)

- [ ] **RENAME-01**: All Python source is relocated from `zeroth.*` to `zeroth.core.*` with zero deletions and zero functional changes (pure rename)
- [ ] **RENAME-02**: The package is a PEP 420 namespace package — no top-level `zeroth/__init__.py` — so future sibling packages (`zeroth.studio`, `zeroth.ext.*`) can coexist under `zeroth.*`
- [ ] **RENAME-03**: All internal imports, string references, entry points, and console scripts point at the new `zeroth.core.*` paths
- [ ] **RENAME-04**: The existing test suite (280+ tests) passes against the renamed package with no skips or regressions
- [ ] **RENAME-05**: Docstring coverage on the public surface of `zeroth.core.*` reaches ≥90% (measured by `interrogate`) with a consistent style (Google-style)

### Documentation Content (DOCS)

- [x] **DOCS-01**: Landing page presents a 10-line hello-world, install snippet, and a "Choose your path" split between embedding as a library and running as a governed service
- [x] **DOCS-02**: Getting Started is a single linear 3-section tutorial (install → first graph with one agent/tool/LLM → run in service mode with an approval gate), reaching first working output in under 5 minutes and completing in under 30
- [x] **DOCS-03**: Every major `zeroth.core.*` subsystem has a Concept page (what it is, why it exists, mental model, where it fits) — covering graph, orchestrator, agents, execution units, memory, contracts, runs, conditions, mappings, policy, approvals, audit, secrets, identity, guardrails, dispatch, economics, storage, service, threads
- [x] **DOCS-04**: Every major subsystem has a Usage Guide (Overview → Minimal example → Common patterns → Pitfalls → Reference cross-link) paired with its Concept page
- [x] **DOCS-05**: A Governance Walkthrough tutorial shows an end-to-end run where an approval gate stops execution, an auditor reviews the trail, and a policy blocks a tool call (Zeroth's differentiator vs. LangGraph/CrewAI)
- [x] **DOCS-06**: A Cookbook section contains at least 10 cross-subsystem recipes at launch (examples: add a human approval step, attach pgvector memory to a node, cap a run's budget with Regulus, sandbox a Python execution unit, retry a failed webhook from the DLQ)
- [x] **DOCS-07**: Python API Reference is auto-generated from docstrings via mkdocstrings + Griffe for the full public surface
- [x] **DOCS-08**: HTTP API Reference is rendered from the FastAPI OpenAPI spec and published alongside the Python reference
- [x] **DOCS-09**: Configuration Reference is auto-generated from pydantic-settings schemas and documents every env var, default, and secret
- [x] **DOCS-10**: Deployment Guide covers local dev, docker-compose, standalone service mode, embedded-in-host-app mode, and deployments with/without the Regulus companion service
- [x] **DOCS-11**: Migration Guide explains how to move from the monolithic `zeroth.*` layout to `zeroth.core.*` (import rename pattern, econ SDK path swap, env var changes, Docker image retag)
- [x] **DOCS-12**: An `examples/` directory at the repo root contains runnable `.py` files (not notebooks) exercising the main subsystems, and CI smoke-tests each on every main commit

### Documentation Site Infrastructure (SITE)

- [x] **SITE-01**: Documentation is built with mkdocs-material using explicit Diátaxis IA (Tutorials / How-to Guides / Concepts / Reference as four top-level sections)
- [x] **SITE-02**: A GitHub Actions workflow builds and deploys the docs site to a public URL on every commit to `main`
- [ ] **SITE-03**: Pull request previews deploy a rendered version of the changed docs so reviewers see the output before merge
- [x] **SITE-04**: The docs site includes built-in search and an automatically generated site map

### Studio Repo Split (STUDIO)

- [x] **STUDIO-01**: A new public repository `rrrozhd/zeroth-studio` contains the Vue 3 + Vue Flow frontend with its full git history preserved (subtree or filter-repo)
- [x] **STUDIO-02**: `zeroth-studio` has its own independent CI pipeline (lint, typecheck, build, test) passing on its default branch
- [x] **STUDIO-03**: `zeroth-studio` consumes `zeroth-core` only via HTTP/OpenAPI — no Python imports, no shared source tree
- [x] **STUDIO-04**: Both repositories' READMEs cross-link to each other, and a cross-repo compatibility matrix (`zeroth-studio` version ↔ `zeroth-core` version) is documented and maintained in at least one of them
- [x] **STUDIO-05**: The frontend types used by `zeroth-studio` are generated from the `zeroth-core` OpenAPI spec via `openapi-typescript`, keeping the two in sync without manual edits

### Monolith Archive (ARCHIVE)

- [ ] **ARCHIVE-01**: A multi-layer archive of the pre-split monolithic repo exists and is documented: local tarball, local bare mirror, and GitHub repository `rrrozhd/zeroth-archive`
- [ ] **ARCHIVE-02**: All 36 worktree branches, both stashes, and the detached-HEAD worktree from the ad-hoc split work are preserved in the archive and recoverable
- [ ] **ARCHIVE-03**: The archive repository carries a visible "this repo is archived — see rrrozhd/zeroth-core and rrrozhd/zeroth-studio" notice in its README and repo description

## Future Requirements

Tracked but out of scope for v4.0. Will be opened as new phases once current gaps are closed.

- **FUTURE-01**: LibCST codemod for automatic `zeroth.*` → `zeroth.core.*` import rewriting
- **FUTURE-02**: HTTP/curl tabs in subsystem usage guides
- **FUTURE-03**: Extension-point guides (custom memory connectors, LLM providers, execution units, judges)
- **FUTURE-04**: Distributed parallel execution across workers (currently in-process asyncio only)
- **FUTURE-05**: HTTP response caching in resilient client (separate concern from resilience)
- **FUTURE-06**: S3/GCS artifact store backend (Redis + filesystem sufficient for v4.0)
- **FUTURE-07**: Model fallback chains / streaming responses

## Deferred to `zeroth-studio` repo (v2.0 phases 24-26)

These requirements are **not cancelled** — they continue in the new `zeroth-studio` repo after the split and will be roadmapped there separately.

- [ ] **API-02** (from v2.0): Studio receives real-time updates via WebSocket (execution status, validation, presence)
- [ ] **API-04** (from v2.0): Studio can trigger workflow execution and receive per-node status updates
- [ ] **AUTH-01** through **AUTH-05** (from v2.0): Canvas execution & AI authoring (model selector, prompt editor, tool attachment, data-flow tooltips, per-node results)
- [ ] **GOV-01** through **GOV-07** (from v2.0): Governance visualization (approval gates, audit trails, sandbox badges, RBAC-aware canvas, cost/budget display, env switching)
- [ ] **COLLAB-01**, **COLLAB-02** (from v2.0): Graph version diff view, collaborative presence indicators

## Out of Scope

Explicitly excluded for v4.0. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Distributed parallel execution across workers | In-process asyncio is sufficient; scale by running more workers |
| Full expression language / DSL for mappings | Existing `_SafeEvaluator` is sufficient; a DSL adds complexity without proportional value |
| Automatic context summarization as default | Research shows observation masking outperforms summarization; summarization is opt-in only |
| HTTP response caching | Separate concern from resilience; defer to future milestone |
| Subgraph runtime flattening | Airflow tried this and deprecated it; use recursive invocation with isolation |
| General-purpose object store | Artifact store handles only workflow intermediate data with TTL cleanup |
| Real-time streaming responses | Async invocation model is sufficient for v4.0 |

## Traceability

Which phases cover which requirements. Updated after roadmap creation.

### v4.0 Phase Mappings

| Requirement | Phase | Status |
|-------------|-------|--------|
| XFRM-01 | Phase 33: Computed Data Mappings | Pending |
| XFRM-02 | Phase 33: Computed Data Mappings | Pending |
| XFRM-03 | Phase 33: Computed Data Mappings | Pending |
| XFRM-04 | Phase 33: Computed Data Mappings | Pending |
| ARTF-01 | Phase 34: Artifact Store | Pending |
| ARTF-02 | Phase 34: Artifact Store | Pending |
| ARTF-03 | Phase 34: Artifact Store | Pending |
| ARTF-04 | Phase 34: Artifact Store | Pending |
| ARTF-05 | Phase 34: Artifact Store | Pending |
| HTTP-01 | Phase 35: Resilient HTTP Client | Pending |
| HTTP-02 | Phase 35: Resilient HTTP Client | Pending |
| HTTP-03 | Phase 35: Resilient HTTP Client | Pending |
| HTTP-04 | Phase 35: Resilient HTTP Client | Pending |
| HTTP-05 | Phase 35: Resilient HTTP Client | Pending |
| HTTP-06 | Phase 35: Resilient HTTP Client | Pending |
| TMPL-01 | Phase 36: Prompt Template Management | Pending |
| TMPL-02 | Phase 36: Prompt Template Management | Pending |
| TMPL-03 | Phase 36: Prompt Template Management | Pending |
| TMPL-04 | Phase 36: Prompt Template Management | Pending |
| CTXW-01 | Phase 37: Context Window Management | Pending |
| CTXW-02 | Phase 37: Context Window Management | Pending |
| CTXW-03 | Phase 37: Context Window Management | Pending |
| CTXW-04 | Phase 37: Context Window Management | Pending |
| CTXW-05 | Phase 37: Context Window Management | Pending |
| PARA-01 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| PARA-02 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| PARA-03 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| PARA-04 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| PARA-05 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| PARA-06 | Phase 38: Parallel Fan-Out / Fan-In | Pending |
| SUBG-01 | Phase 39: Subgraph Composition | Pending |
| SUBG-02 | Phase 39: Subgraph Composition | Pending |
| SUBG-03 | Phase 39: Subgraph Composition | Pending |
| SUBG-04 | Phase 39: Subgraph Composition | Pending |
| SUBG-05 | Phase 39: Subgraph Composition | Pending |
| SUBG-06 | Phase 39: Subgraph Composition | Pending |
| SUBG-07 | Phase 39: Subgraph Composition | Pending |
| SUBG-08 | Phase 39: Subgraph Composition | Pending |

**Coverage:**
- v4.0 requirements: 38 total
- Mapped to phases: 38/38
- Unmapped: 0
- Integration validation (Phase 40) covers all 38 requirements collectively

---

## v2.0 Requirements (Zeroth Studio — partially shipped)

Phases 22–23 shipped in v2.0. Phases 24–26 deferred to the new `zeroth-studio` repo — see "Deferred to `zeroth-studio` repo" above. v2.0 REQ-IDs are preserved for history but not re-mapped in v3.0.

### Canvas Foundation

- [x] **CANV-01** · [x] **CANV-02** · [x] **CANV-03** · [x] **CANV-04** · [x] **CANV-05** · [x] **CANV-06** · [x] **CANV-07** · [x] **CANV-08** · [x] **CANV-09** · [x] **CANV-10**

### Graph Authoring API

- [x] **API-01** · [x] **API-03** · [ ] **API-02** (→ zeroth-studio) · [ ] **API-04** (→ zeroth-studio)

### Governance Visualization

- [ ] **GOV-01** … **GOV-07** (→ zeroth-studio)

### AI Authoring

- [ ] **AUTH-01** … **AUTH-05** (→ zeroth-studio) · [x] **AUTH-06**

### Versioning & Collaboration

- [ ] **COLLAB-01** · [ ] **COLLAB-02** (→ zeroth-studio)

### Deployment & Infrastructure

- [x] **INFRA-01** · [x] **INFRA-02**

---

*Requirements last updated: 2026-04-12 after v4.0 roadmap creation.*
