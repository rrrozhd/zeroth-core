# Roadmap: Zeroth

## Milestones

- v1.0 Runtime Foundation — Phases 1-9 (shipped 2026-03-27)
- v1.1 Production Readiness — Phases 11-21 (shipped 2026-04-09)
- v2.0 Zeroth Studio — Phases 22-26 (partially shipped: 22-23 done; 24-26 moved to `zeroth-studio` repo under v3.0)
- v3.0 Core Library Extraction, Studio Split & Documentation — Phases 27-32 (shipped 2026-04-11)
- v4.0 Platform Extensions for Production Agentic Workflows — Phases 33-40 (in progress)

## Phases

<details>
<summary>v1.0 Runtime Foundation (Phases 1-9) — SHIPPED 2026-03-27</summary>

- [x] Phase 1: Core Foundation (2/2 plans) — completed 2026-03-19
- [x] Phase 2: Execution Core (2/2 plans) — completed 2026-03-19
- [x] Phase 3: Platform Control (2/2 plans) — completed 2026-03-19
- [x] Phase 4: Deployment Surface (1/1 plan) — completed 2026-03-20
- [x] Phase 5: Integration & Polish (1/1 plan) — completed 2026-03-26
- [x] Phase 6: Identity & Tenant Governance (1/1 plan) — completed 2026-03-27
- [x] Phase 7: Transparent Governance & Provenance (1/1 plan) — completed 2026-03-27
- [x] Phase 8: Runtime Security Hardening (1/1 plan) — completed 2026-03-27
- [x] Phase 9: Durable Control Plane & Production Operations (1/1 plan) — completed 2026-03-27

</details>

<details>
<summary>v1.1 Production Readiness (Phases 11-21) — SHIPPED 2026-04-09</summary>

- [x] Phase 11: Config & Postgres Storage (3/3 plans) — completed 2026-04-06
- [x] Phase 12: Real LLM Providers & Retry (3/3 plans) — completed 2026-04-06
- [x] Phase 13: Regulus Economics Integration (3/3 plans) — completed 2026-04-07
- [x] Phase 14: Memory Connectors & Container Sandbox (5/5 plans) — completed 2026-04-07
- [x] Phase 15: Webhooks & Approval SLA (3/3 plans) — completed 2026-04-07
- [x] Phase 16: Distributed Dispatch & Horizontal Scaling (3/3 plans) — completed 2026-04-07
- [x] Phase 17: Deployment Packaging & Operations (3/3 plans) — completed 2026-04-07
- [x] Phase 18: Cross-Phase Integration Wiring (2/2 plans) — completed 2026-04-08
- [x] Phase 19: Agent Node LLM API Parity (3/3 plans) — completed 2026-04-08
- [x] Phase 20: Bootstrap Integration Wiring (1/1 plan) — completed 2026-04-09
- [x] Phase 21: Health Probe Fix & Tech Debt (1/1 plan) — completed 2026-04-09

</details>

<details>
<summary>v2.0 Zeroth Studio (Phases 22-26) — PARTIALLY SHIPPED; 24-26 moved to zeroth-studio repo</summary>

- [x] Phase 22: Canvas Foundation & Dev Infrastructure (6/6 plans) — completed 2026-04-09
- [x] Phase 23: Canvas Editing UX (4/4 plans) — completed 2026-04-09
- [->] Phase 24: Execution & AI Authoring — **moved to `zeroth-studio` repo** (part of v3.0 split)
- [->] Phase 25: Governance Visualization — **moved to `zeroth-studio` repo** (part of v3.0 split)
- [->] Phase 26: Versioning & Collaboration — **moved to `zeroth-studio` repo** (part of v3.0 split)

</details>

<details>
<summary>v3.0 Core Library Extraction, Studio Split & Documentation (Phases 27-32) — SHIPPED 2026-04-11</summary>

- [x] Phase 27: Monolith Archive & Namespace Rename (4/4 plans) — completed 2026-04-10
- [x] Phase 28: PyPI Publishing (econ-sdk + zeroth-core) (3/3 plans) — completed 2026-04-11
- [x] Phase 29: Studio Repo Split (4/4 plans) — completed 2026-04-11
- [x] Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough (5/5 plans) — completed 2026-04-11
- [x] Phase 31: Subsystem Concepts, Usage Guides, Cookbook & Examples (5/5 plans) — completed 2026-04-11
- [x] Phase 32: Reference Docs, Deployment & Migration Guide (6/6 plans) — completed 2026-04-11

</details>

### v4.0 Platform Extensions for Production Agentic Workflows (In Progress)

**Milestone Goal:** Close 7 architectural gaps identified during production adoption audit, enabling zeroth-core to support parallel execution, composable subgraphs, large payloads, context window management, resilient HTTP, prompt templates, and computed data mappings.

- [ ] **Phase 33: Computed Data Mappings** - Transform mapping operation using the existing expression engine for side-effect-free data transformation
- [ ] **Phase 34: Artifact Store** - Pluggable large payload externalization with Redis and filesystem backends, TTL, and audit compatibility
- [ ] **Phase 35: Resilient HTTP Client** - Managed async HTTP with retry/backoff, circuit breaking, connection pooling, capability-gated and audited
- [ ] **Phase 36: Prompt Template Management** - Versioned template registry with Jinja2 sandboxed rendering, agent node integration, and audit redaction
- [ ] **Phase 37: Context Window Management** - Token tracking with configurable compaction strategies to prevent agent context overflow
- [ ] **Phase 38: Parallel Fan-Out / Fan-In** - Spawn N parallel branches with per-branch isolation, governance, budget awareness, and deterministic fan-in
- [ ] **Phase 39: Subgraph Composition** - Reference published graphs as nested nodes with governance inheritance, thread continuity, and approval propagation
- [ ] **Phase 40: Integration & Service Wiring** - Wire all v4.0 features into service bootstrap, update OpenAPI spec, cross-feature interaction testing

## Phase Details

<details>
<summary>v1.0-v3.0 Phase Details (shipped)</summary>

### Phase 22: Canvas Foundation & Dev Infrastructure
**Goal**: Users can create workflow graphs by placing nodes and drawing edges on an interactive canvas, save and load them via the API, and work within a responsive three-panel Studio layout served from Docker
**Depends on**: Phase 21 (v1.1 shipped)
**Requirements**: CANV-01, CANV-02, CANV-06, CANV-09, CANV-10, API-01, API-03, INFRA-01, INFRA-02
**Success Criteria** (what must be TRUE):
  1. User can drag nodes onto a pannable, zoomable canvas and draw edges between typed node ports
  2. User can save a workflow graph and reload it in a new browser session with all nodes and edges intact
  3. User can navigate the canvas with pan, zoom, fit-to-view, and minimap
  4. User sees a responsive three-panel layout (workflow rail, canvas, inspector) with collapsible panels
  5. Studio frontend is served via Nginx alongside FastAPI in Docker, and frontend types are generated from the backend OpenAPI spec
**Plans:** 6/6 plans complete
**UI hint**: yes

### Phase 23: Canvas Editing UX
**Goal**: Users have a complete editing experience with categorized node palette, property inspector, auto-layout, undo/redo, keyboard shortcuts, and validation feedback
**Depends on**: Phase 22
**Requirements**: CANV-03, CANV-04, CANV-05, CANV-07, CANV-08, AUTH-06
**Success Criteria** (what must be TRUE):
  1. User can browse and search node types in a categorized sidebar palette
  2. User can select a node and view/edit its properties in the inspector panel
  3. User can auto-layout the graph into a readable DAG arrangement
  4. User can undo and redo canvas operations, and use keyboard shortcuts for common actions (delete, select-all, copy/paste, duplicate)
  5. User can see validation indicators on nodes with missing fields, invalid connections, or type mismatches
**Plans:** 4/4 plans complete
**UI hint**: yes

### Phase 27: Monolith Archive & Namespace Rename
**Goal**: The monolithic Zeroth repo is preserved in a multi-layer archive, and all Python source is relocated into the `zeroth.core.*` PEP 420 namespace package with the full existing test suite passing on the renamed layout
**Depends on**: Phase 23 (v2.0 partial ship)
**Requirements**: ARCHIVE-01, ARCHIVE-02, ARCHIVE-03, RENAME-01, RENAME-02, RENAME-03, RENAME-04, RENAME-05
**Success Criteria** (what must be TRUE):
  1. The monolith is recoverable from three independent archive layers (local tarball, local bare mirror, `rrrozhd/zeroth-archive` on GitHub) and the archived repo carries a visible "archived — see zeroth-core/zeroth-studio" notice
  2. All 36 pre-existing worktree branches, both stashes, and the detached-HEAD worktree are preserved in the archive and can be checked out from the bare mirror
  3. All Python source lives under `zeroth.core.*` with no top-level `zeroth/__init__.py` (PEP 420 namespace package), no deletions, no functional changes
  4. The full existing test suite (280+ tests) passes against the renamed package with zero skips and zero regressions
  5. Docstring coverage on the `zeroth.core.*` public surface reaches >=90% (measured by `interrogate`) using a single consistent style (Google-style)
**Plans**: 4/4 plans complete

### Phase 28: PyPI Publishing (`econ-instrumentation-sdk` + `zeroth-core`)
**Goal**: Both `econ-instrumentation-sdk` and `zeroth-core` are published to PyPI via GitHub Actions trusted publisher, a clean-venv install of `zeroth-core[all]` succeeds, and every declared optional extra is verified installable
**Depends on**: Phase 27
**Requirements**: PKG-01, PKG-02, PKG-03, PKG-04, PKG-05, PKG-06
**Success Criteria** (what must be TRUE):
  1. `econ-instrumentation-sdk` is live on PyPI at a stable version, and `zeroth-core` depends on it via a PyPI version constraint, not a local file path
  2. `pyproject.toml` declares the full set of optional-dependency extras (`[memory-pg]`, `[memory-chroma]`, `[memory-es]`, `[dispatch]`, `[sandbox]`, `[all]`) and each extra resolves and installs cleanly
  3. Repository root contains `CHANGELOG.md` (keepachangelog format), `LICENSE`, and `CONTRIBUTING.md` suitable for public PyPI consumption
  4. PyPI releases are published by GitHub Actions using trusted publisher (OIDC), with no long-lived API tokens in the repo or CI
  5. In a fresh virtualenv, `pip install zeroth-core` followed by running the Getting Started hello example produces working output end-to-end
**Plans:** 3/3 plans complete

### Phase 29: Studio Repo Split
**Goal**: `rrrozhd/zeroth-studio` exists as a public repo with preserved git history, passing independent CI, HTTP-only consumption of `zeroth-core`, and a documented cross-repo compatibility matrix
**Depends on**: Phase 28
**Requirements**: STUDIO-01, STUDIO-02, STUDIO-03, STUDIO-04, STUDIO-05
**Success Criteria** (what must be TRUE):
  1. `rrrozhd/zeroth-studio` is a public GitHub repository containing the Vue 3 + Vue Flow frontend with full git history preserved (via subtree or git-filter-repo)
  2. `zeroth-studio` has its own CI pipeline (lint, typecheck, build, test) that passes on its default branch without touching or importing from `zeroth-core`
  3. `zeroth-studio`'s only contract with `zeroth-core` is HTTP/OpenAPI — its frontend types are generated from the `zeroth-core` OpenAPI spec via `openapi-typescript`
  4. Both repos' READMEs cross-link, and a `zeroth-studio x zeroth-core` compatibility matrix is documented and maintained
  5. A developer can clone `zeroth-studio`, run `npm install && npm run dev`, and develop against a running `zeroth-core` service without any cross-repo source dependencies
**Plans:** 4/4 plans complete

### Phase 30: Docs Site Foundation, Getting Started & Governance Walkthrough
**Goal**: The `zeroth-core` documentation site is live on a public URL, built with mkdocs-material using explicit Diataxis IA, and contains the complete "first working path" — landing page, 3-section Getting Started tutorial, and a Governance Walkthrough showcasing Zeroth's differentiator
**Depends on**: Phase 28
**Requirements**: SITE-01, SITE-02, SITE-03, SITE-04, DOCS-01, DOCS-02, DOCS-05
**Success Criteria** (what must be TRUE):
  1. The docs site is built by mkdocs-material with four top-level Diataxis sections (Tutorials / How-to Guides / Concepts / Reference), has built-in search, and auto-generates a site map
  2. A GitHub Actions workflow builds and deploys the docs to a public URL on every commit to `main`, and pull requests get preview deploys
  3. The landing page shows a 10-line hello-world, install snippet, and a "Choose your path" split between embedding as a library and running as a service
  4. Getting Started is a single linear 3-section tutorial (install -> first graph with one agent/tool/LLM -> run in service mode with an approval gate) that produces first working output in under 5 minutes and completes in under 30
  5. A Governance Walkthrough tutorial runs end-to-end with an approval gate stopping execution, an auditor reviewing the trail, and a policy blocking a tool call
**Plans:** 5/5 plans complete

### Phase 31: Subsystem Concepts, Usage Guides, Cookbook & Examples
**Goal**: Every major `zeroth.core.*` subsystem has a paired Concept page and Usage Guide on the docs site, the Cookbook contains at least 10 cross-subsystem recipes, and an `examples/` directory with CI-tested runnable `.py` files covers the main subsystems
**Depends on**: Phase 30
**Requirements**: DOCS-03, DOCS-04, DOCS-06, DOCS-12
**Success Criteria** (what must be TRUE):
  1. Every major subsystem (graph, orchestrator, agents, execution units, memory, contracts, runs, conditions, mappings, policy, approvals, audit, secrets, identity, guardrails, dispatch, economics, storage, service, threads) has a Concept page explaining what it is, why it exists, and where it fits
  2. Every major subsystem has a Usage Guide (Overview -> Minimal example -> Common patterns -> Pitfalls -> Reference cross-link) paired with its Concept page
  3. The Cookbook section contains at least 10 cross-subsystem recipes covering the most common Zeroth tasks (approval steps, memory attachment, budget capping, sandboxing, webhook retry, etc.)
  4. The repo root contains an `examples/` directory with runnable `.py` files (no notebooks) exercising the main subsystems
  5. A CI job smoke-tests every file in `examples/` on every commit to `main`, and the job is green
**Plans:** 5/5 plans complete

### Phase 32: Reference Docs, Deployment & Migration Guide
**Goal**: The docs site has a complete Reference quadrant (Python API auto-generated from docstrings, HTTP API rendered from OpenAPI, Configuration reference auto-generated from pydantic-settings), plus a Deployment Guide covering every supported mode and a Migration Guide from the monolith layout
**Depends on**: Phase 30 (and Phase 27 for the renamed import paths referenced by the API reference)
**Requirements**: DOCS-07, DOCS-08, DOCS-09, DOCS-10, DOCS-11
**Success Criteria** (what must be TRUE):
  1. Python API Reference is auto-generated from docstrings via mkdocstrings + Griffe for the full `zeroth.core.*` public surface, cross-linked from narrative pages
  2. HTTP API Reference is rendered from the `zeroth-core` FastAPI OpenAPI spec and published alongside the Python reference on the docs site
  3. Configuration Reference is auto-generated from the pydantic-settings schemas and documents every env var, its default, and whether it is a secret
  4. Deployment Guide covers local development, docker-compose, standalone service mode, embedded-in-host-app mode, and deployments with and without the Regulus companion service
  5. Migration Guide walks an existing monolith user through the switch to `zeroth.core.*` — import rename pattern, econ SDK path swap, env var changes, and Docker image retag
**Plans:** 6/6 plans complete

</details>

### Phase 33: Computed Data Mappings
**Goal**: Edge mappings can compute derived values from source payloads using the existing expression engine, enabling side-effect-free data transformation between nodes
**Depends on**: Phase 32 (v3.0 shipped)
**Requirements**: XFRM-01, XFRM-02, XFRM-03, XFRM-04
**Success Criteria** (what must be TRUE):
  1. A graph author can define a transform mapping on an edge that evaluates an expression (e.g., `payload.items | length`, `payload.score * 100`) and writes the result to a target path on the next node's input
  2. Transform expressions can reference `payload.*`, `state.*`, and `variables.*` using the same syntax as condition expressions, and the evaluated result is validated against the target node's input contract
  3. Transform expressions are guaranteed side-effect-free: no network access, no filesystem access, no imports, no dunder attribute traversal -- enforced by the hardened expression evaluator with namespace restrictions
  4. Existing passthrough, rename, constant, and default mapping operations continue to work unchanged (backward compatibility)
**Plans**: 2 plans
Plans:
- [x] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [x] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 34: Artifact Store
**Goal**: Nodes can externalize large payloads into a pluggable artifact store instead of embedding them in run state, preventing payload bloat while preserving audit traceability and contract compatibility
**Depends on**: Phase 33
**Requirements**: ARTF-01, ARTF-02, ARTF-03, ARTF-04, ARTF-05
**Success Criteria** (what must be TRUE):
  1. A pluggable ArtifactStore interface exists with working implementations for Redis (with SETEX TTL) and local filesystem, configurable via settings
  2. A node can emit an ArtifactReference (store, key, content_type, size) as part of its output; the reference is persisted in run history while the actual payload lives in the artifact store
  3. Artifacts support configurable TTL; artifacts tied to a run are cleanable when the run is archived; TTLs are refreshed when a run is checkpointed or paused for approval (preventing dangling references)
  4. Audit records log artifact references (not full payloads); audit evidence export can optionally resolve references to retrieve full payloads
  5. Contracts support an ArtifactReference type that validates the reference structure without requiring the full payload at validation time
**Plans**: 2 plans
Plans:
- [x] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [x] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 35: Resilient HTTP Client
**Goal**: Agent tools and executable units have access to a platform-provided async HTTP client with managed retry, circuit breaking, connection pooling, governance gating, and audit logging
**Depends on**: Phase 32 (v3.0 shipped)
**Requirements**: HTTP-01, HTTP-02, HTTP-03, HTTP-04, HTTP-05, HTTP-06
**Success Criteria** (what must be TRUE):
  1. A platform-provided async HTTP client (wrapping httpx) is available to agent tools and executable units, configurable per-node or per-tool with sensible defaults
  2. The client retries failed requests with exponential backoff and jitter for configurable status codes (default: 408, 429, 5xx), and a per-endpoint circuit breaker opens after configurable failure thresholds and resets after a timeout
  3. Connection pools are shared or per-tenant with configurable limits, and the client resolves auth headers/tokens from the SecretResolver automatically based on endpoint configuration
  4. Every external HTTP call is gated by NETWORK_READ / NETWORK_WRITE / EXTERNAL_API_CALL capabilities, logged in audit records (URL, method, status code, latency), and subject to rate limiting
**Plans**: 2 plans
Plans:
- [x] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 36: Prompt Template Management
**Goal**: Graph authors can define versioned prompt templates and reference them from agent nodes, with Jinja2 sandboxed rendering at runtime and automatic audit redaction of secret variables
**Depends on**: Phase 32 (v3.0 shipped)
**Requirements**: TMPL-01, TMPL-02, TMPL-03, TMPL-04
**Success Criteria** (what must be TRUE):
  1. A template registry stores and versions prompt templates by name, analogous to the contract registry, and templates can be created, retrieved, and listed via the registry API
  2. Templates support variable interpolation from node input, run state, or memory using Jinja2 SandboxedEnvironment, preventing template injection attacks
  3. An agent node can reference a template by name and version instead of providing a raw instruction string; the template is resolved and rendered at runtime before the LLM invocation
  4. The rendered prompt (post-interpolation) is available in audit records; template variables containing secrets are automatically redacted in audit output
**Plans**: 2 plans
Plans:
- [ ] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 37: Context Window Management
**Goal**: Agent threads track accumulated token usage and automatically apply configurable compaction strategies before context overflow, preserving conversation continuity across runs
**Depends on**: Phase 32 (v3.0 shipped)
**Requirements**: CTXW-01, CTXW-02, CTXW-03, CTXW-04, CTXW-05
**Success Criteria** (what must be TRUE):
  1. Approximate token count of accumulated agent messages per thread is tracked using the LLM provider's tokenizer (via litellm.token_counter), updated after each LLM invocation
  2. When token count exceeds a configurable threshold, a compaction strategy is applied before the next LLM invocation (default: observation masking of older messages)
  3. Compaction strategy is pluggable per agent node with three built-in strategies: truncation (drop oldest), observation masking (replace tool outputs with placeholders), and LLM-based summarization (condense older messages)
  4. Compaction results are stored in thread memory so they persist across runs; original messages can optionally be archived for audit retrieval
  5. Per-agent-node settings are configurable: max_context_tokens, summary_trigger_ratio, compaction_strategy, and preserve_recent_messages_count
**Plans**: 2 plans
Plans:
- [ ] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 38: Parallel Fan-Out / Fan-In
**Goal**: A node can spawn N parallel execution branches that run concurrently with per-branch isolation, and a synchronization barrier collects all branch outputs into a deterministically ordered aggregated payload
**Depends on**: Phase 33 (computed mappings for fan-in aggregation), Phase 34 (artifact store for large parallel payloads)
**Requirements**: PARA-01, PARA-02, PARA-03, PARA-04, PARA-05, PARA-06
**Success Criteria** (what must be TRUE):
  1. A graph author can configure a node to spawn N parallel branches from its output (e.g., one branch per list item), and a synchronization barrier collects all branch outputs into an aggregated payload with deterministic ordering by branch index
  2. Each parallel branch has its own isolated execution context (visit counts, audit trail, failure tracking); a failure in one branch does not automatically fail others when configured for best-effort mode (fail-fast is also supported)
  3. Policy enforcement, audit recording, and contract validation apply independently per branch, each producing its own audit records linked to the parent run
  4. Cost attribution tracks per-branch spend; BudgetEnforcer is consulted before spawning with a pre-reservation of total estimated cost; ExecutionSettings guardrails (max_total_steps, max_visits_per_node) account for parallel branches as sum across all branches
**Plans**: 2 plans
Plans:
- [ ] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 39: Subgraph Composition
**Goal**: A graph can reference another published graph as a nested subgraph node, with the orchestrator entering the subgraph as a scoped execution that inherits governance, shares thread memory, and propagates approvals back to the parent
**Depends on**: Phase 38 (parallel execution -- co-designed, shares _drive() loop changes)
**Requirements**: SUBG-01, SUBG-02, SUBG-03, SUBG-04, SUBG-05, SUBG-06, SUBG-07, SUBG-08
**Success Criteria** (what must be TRUE):
  1. A graph author can add a subgraph node that references another published graph by name; the subgraph's entry contract must be compatible with the referencing edge's mapping output, and the subgraph's final output maps back to the parent graph's expected input
  2. The orchestrator enters the subgraph as a nested scope sharing the parent's thread_id (configurable); agents inside subgraphs participate in the same thread memory; the parent's policies apply as a baseline that the subgraph can further restrict but not relax
  3. If a HumanApprovalNode inside a subgraph pauses execution, the parent run transitions to WAITING_APPROVAL; resolution resumes the subgraph and eventually the parent run
  4. The same subgraph can be referenced by multiple parent graphs and at multiple points within a single parent; subgraph references can pin to a specific deployment version or float to the latest active deployment; nested subgraphs (subgraph within a subgraph) are supported with a configurable depth limit
  5. Audit records from subgraph execution link to the parent run via parent_run_id, and node IDs are namespaced to prevent collisions across nesting levels
**Plans**: 2 plans
Plans:
- [ ] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

### Phase 40: Integration & Service Wiring
**Goal**: All v4.0 features are wired into the service bootstrap, the OpenAPI spec reflects the new capabilities, cross-feature interactions are tested, and documentation is updated
**Depends on**: Phase 38, Phase 39 (all feature phases complete)
**Requirements**: All v4.0 requirements (integration validation)
**Success Criteria** (what must be TRUE):
  1. Service bootstrap initializes all new subsystems (artifact store, HTTP client, template registry, context window manager) and makes them available to the orchestrator and agent runtime without manual configuration beyond settings
  2. The OpenAPI spec includes endpoints for new v4.0 capabilities (artifact retrieval, template CRUD, parallel run status) and the docs site reflects the updated API surface
  3. Cross-feature interactions work correctly: parallel branches can use artifact store for large outputs, agent nodes inside parallel branches respect context window limits, subgraph nodes inside parallel branches execute with proper governance isolation
  4. All existing tests continue to pass (backward compatibility), and new integration tests cover the cross-feature scenarios listed above
**Plans**: 2 plans
Plans:
- [ ] 33-01-PLAN.md — Core transform operation: model, error, validator, executor
- [ ] 33-02-PLAN.md — Safe builtins, orchestrator wiring, integration tests

## Progress

**Execution Order:**
Phases execute in numeric order. v4.0 runs 33 -> 34 -> 35/36/37 (parallelizable) -> 38 -> 39 -> 40.

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Core Foundation | v1.0 | 2/2 | Complete | 2026-03-19 |
| 2. Execution Core | v1.0 | 2/2 | Complete | 2026-03-19 |
| 3. Platform Control | v1.0 | 2/2 | Complete | 2026-03-19 |
| 4. Deployment Surface | v1.0 | 1/1 | Complete | 2026-03-20 |
| 5. Integration & Polish | v1.0 | 1/1 | Complete | 2026-03-26 |
| 6. Identity & Tenant Governance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 7. Transparent Governance & Provenance | v1.0 | 1/1 | Complete | 2026-03-27 |
| 8. Runtime Security Hardening | v1.0 | 1/1 | Complete | 2026-03-27 |
| 9. Durable Control Plane & Production Operations | v1.0 | 1/1 | Complete | 2026-03-27 |
| 11. Config & Postgres Storage | v1.1 | 3/3 | Complete | 2026-04-06 |
| 12. Real LLM Providers & Retry | v1.1 | 3/3 | Complete | 2026-04-06 |
| 13. Regulus Economics Integration | v1.1 | 3/3 | Complete | 2026-04-07 |
| 14. Memory Connectors & Container Sandbox | v1.1 | 5/5 | Complete | 2026-04-07 |
| 15. Webhooks & Approval SLA | v1.1 | 3/3 | Complete | 2026-04-07 |
| 16. Distributed Dispatch & Horizontal Scaling | v1.1 | 3/3 | Complete | 2026-04-07 |
| 17. Deployment Packaging & Operations | v1.1 | 3/3 | Complete | 2026-04-07 |
| 18. Cross-Phase Integration Wiring | v1.1 | 2/2 | Complete | 2026-04-08 |
| 19. Agent Node LLM API Parity | v1.1 | 3/3 | Complete | 2026-04-08 |
| 20. Bootstrap Integration Wiring | v1.1 | 1/1 | Complete | 2026-04-09 |
| 21. Health Probe Fix & Tech Debt | v1.1 | 1/1 | Complete | 2026-04-09 |
| 22. Canvas Foundation & Dev Infrastructure | v2.0 | 6/6 | Complete | 2026-04-09 |
| 23. Canvas Editing UX | v2.0 | 4/4 | Complete | 2026-04-09 |
| 24. Execution & AI Authoring | v2.0 | — | Moved to `zeroth-studio` | — |
| 25. Governance Visualization | v2.0 | — | Moved to `zeroth-studio` | — |
| 26. Versioning & Collaboration | v2.0 | — | Moved to `zeroth-studio` | — |
| 27. Monolith Archive & Namespace Rename | v3.0 | 4/4 | Complete | 2026-04-10 |
| 28. PyPI Publishing (econ-sdk + zeroth-core) | v3.0 | 3/3 | Complete | 2026-04-11 |
| 29. Studio Repo Split | v3.0 | 4/4 | Complete | 2026-04-11 |
| 30. Docs Site Foundation, Getting Started & Governance Walkthrough | v3.0 | 5/5 | Complete | 2026-04-11 |
| 31. Subsystem Concepts, Usage Guides, Cookbook & Examples | v3.0 | 5/5 | Complete | 2026-04-11 |
| 32. Reference Docs, Deployment & Migration Guide | v3.0 | 6/6 | Complete | 2026-04-11 |
| 33. Computed Data Mappings | v4.0 | 2/2 | Complete    | 2026-04-12 |
| 34. Artifact Store | v4.0 | 0/0 | Not started | - |
| 35. Resilient HTTP Client | v4.0 | 0/0 | Not started | - |
| 36. Prompt Template Management | v4.0 | 0/0 | Not started | - |
| 37. Context Window Management | v4.0 | 0/0 | Not started | - |
| 38. Parallel Fan-Out / Fan-In | v4.0 | 0/0 | Not started | - |
| 39. Subgraph Composition | v4.0 | 0/0 | Not started | - |
| 40. Integration & Service Wiring | v4.0 | 0/0 | Not started | - |
