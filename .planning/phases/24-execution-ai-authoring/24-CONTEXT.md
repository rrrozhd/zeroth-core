# Phase 24: Execution & AI Authoring - Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 24 delivers workflow execution from the Studio canvas with real-time per-node updates via WebSocket, plus the full AI authoring surface: agent node configuration (model/prompt/tools/memory), a prompt store, a user-managed model registry, a connectors registry (MCP / built-in tools / execution units), a Studio settings area with environment variables backed by the Zeroth secrets service, structured execution results per node, and run control (start/stop, fail-fast).

This phase **expands the roadmap's original scope** — discussion revealed that "select model" and "attach tools" cannot be implemented without upstream registries for models, prompts, connectors, and env vars. CONTEXT.md captures the full surface so planner can scope plans (may split into multiple plans within the phase).

This phase does NOT deliver: governance visualization (Phase 25 — approval gates, RBAC, audit trail, budgets, environment switching), version diffing (Phase 26), collaboration / presence (Phase 26), per-node retry (future backlog), LLM token streaming on canvas (explicitly out of scope per PROJECT.md).

</domain>

<decisions>
## Implementation Decisions

### WebSocket Transport (API-02)
- **D-01:** Per-workflow WebSocket at `/api/studio/v1/workflows/{workflow_id}/events`. One connection per open workflow; closes on workflow close. Single transport used for run events (now), plus future presence/validation (Phase 25/26).
- **D-02:** Message envelope: `{ seq: int, type: "run_started" | "node_status" | "node_result" | "run_completed" | "run_failed" | "heartbeat", run_id, node_id?, payload }`. Monotonic `seq` per run enables replay.
- **D-03:** Reconnect behavior: exponential backoff with jitter. On reconnect, client sends `{ last_seq }`; backend replays buffered events from that point. Requires per-run ring buffer of recent events (size: researcher to confirm — target ~500 events).
- **D-04:** Heartbeat: server sends `heartbeat` every 15s while WS open. Client treats 30s silence as dead connection and reconnects.
- **D-05:** Auth: WebSocket inherits session auth from the REST API (cookie/token). Researcher to confirm FastAPI WS auth pattern.

### Execution Trigger & Run Control (API-04, AUTH-05)
- **D-06:** "Run" button lives in `CanvasControls.vue` alongside auto-layout and undo/redo. Clicking opens an **Inputs modal** that auto-generates a form from the Start node's input schema.
- **D-07:** While a run is in progress, the Run button becomes a red **Stop** button that cancels the run via `POST /runs/{run_id}/cancel` (new endpoint) — backend uses existing cancellation path.
- **D-08:** Trigger dispatches via existing `POST /runs` with `{ graph_id, graph_spec, inputs }`. Response returns `run_id`; frontend immediately opens (or reuses) the workflow WS and filters events for this `run_id`.
- **D-09:** Fail-fast: first node failure halts the run. Downstream nodes marked `skipped`. Matches Zeroth's governance posture. No per-node retry in Phase 24 — full re-run required.
- **D-10:** Results persistence: on workflow load, frontend calls `GET /runs?workflow_id={id}&latest=true` to hydrate the most recent run's per-node results. Badges and results restored automatically.

### Per-Node Execution Display (AUTH-05)
- **D-11:** Status badges overlay each node: `running` (spinner, blue), `complete` (green check), `failed` (red X), `skipped` (grey), `pending` (default). Rendered on `BaseNode.vue` as an absolutely-positioned chip.
- **D-12:** Persistent node chips after run completion: **status badge, token count + cost chip, model-used chip, latency chip**. All four shown on agent nodes; non-agent nodes show only status + latency.
- **D-13:** Full results shown in a new **"Results" tab in the inspector panel** (alongside the existing Properties tab). Activated when a node is selected and has last-run data.
- **D-14:** Results tab content for agent nodes: (a) rendered prompt with variables substituted, (b) raw output (JSON-formatted if structured schema used), (c) step-by-step timeline of LLM calls + tool calls, (d) token breakdown (input/output/total), (e) cost, (f) latency, (g) model used.
- **D-15:** Multi-turn / tool-calling runs display as an **ordered step timeline** in the Results tab. Each step is expandable (LLM call → tool call → LLM call → final output). Matches async step-completion model. Not a chat transcript.
- **D-16:** On failure, Results tab shows a **structured error**: type (e.g., ProviderError), message, collapsible stack trace, and suggested fix when determinable (e.g., "Check ANTHROPIC_API_KEY env var"). Other sections (prompt, tokens) still rendered if data available.

### Agent Config Card (AUTH-01, AUTH-02, AUTH-03)
- **D-17:** Creating or editing an agent node opens an **Agent Config Card** — a rich configuration surface (this IS the inspector Properties tab for agent nodes, not a separate modal). All agent configuration fields live here.
- **D-18:** Agent config fields: **label**, **model** (from registry), **system prompt** (from prompt store), **user prompt** (from prompt store), **input spec**, **temperature**, **max_tokens**, **top_p**, **custom base URL** (optional override), **structured output schema** (JSON schema or plain text mode), **tools** (multi-select from connector registry), **execution units** (multi-select), **memory connector** (reference to canvas Memory Resource node), **messages connector** (reference to canvas/registry), **thread key** (templatized, e.g., `{{inputs.user_id}}`), **API key reference** (picked from env vars by name).
- **D-19:** The config card form uses the Phase 23 auto-generated property form pattern — `InspectorField.vue` gains new field types: `model-select`, `prompt-ref`, `tool-multi-select`, `memory-ref`, `env-var-ref`, `json-schema`.

### Model Registry (AUTH-01)
- **D-20:** Backend endpoint `GET /api/studio/v1/models` returns all user-registered model configurations for the tenant. Response includes: id, label, provider, base_model (LiteLLM string), context_window, cost_tier, default_params. API keys never returned.
- **D-21:** Model CRUD at `/api/studio/v1/models` (POST/PUT/DELETE). Create payload: label, provider, base_model, env_var_ref (for API key), base_url, default params (temp/max_tokens/top_p).
- **D-22:** Model registry UI lives in **Studio Settings → Models** tab. List + create/edit form.
- **D-23:** Model selector in agent config card: grouped-by-provider dropdown showing label, context window (e.g., "200k"), and cost tier ($ / $$ / $$$).
- **D-24:** **Per-node only** — no workflow-level default model. Each agent explicitly picks its model.

### Prompt Store (AUTH-02)
- **D-25:** Dedicated **Prompts page** in Studio (new top-level entry or within Settings — planner to decide placement, recommend top-level for authoring-first UX).
- **D-26:** Prompt fields: name, description, role (`system` | `user`), body (CodeMirror), variables (auto-detected), version list. Each save creates a new version; agent nodes reference either a specific version or `latest`.
- **D-27:** CodeMirror editor with **Jinja-style `{{variable}}` templating**. Supported refs: `{{inputs.X}}` (workflow inputs), `{{nodes.nodeName.output}}` (upstream node outputs), `{{env.VAR}}` (env vars — resolved at runtime). Placeholders highlighted in editor.
- **D-28:** Inline shortcut: "+ New prompt" button in agent config card opens an inline modal with CodeMirror to author and save without leaving the canvas. Saved prompts land in the same store.
- **D-29:** Variable binding on agent node: when a prompt is selected, the agent config card parses placeholders and shows a **Variable Bindings** section mapping each `{{var}}` to an upstream node output or static value. Unbound variables flagged with the Phase 23 validation indicator (red border).
- **D-30:** Simple versioning — no branching, no diff view in Phase 24. History list only.
- **D-31:** Backend: new `/api/studio/v1/prompts` CRUD endpoints. Storage in Postgres (same DB as graphs).

### Tools & Connectors Registry (AUTH-03)
- **D-32:** Dedicated **Connectors page** in Studio (new top-level nav entry). Tabs: **MCP Servers**, **Built-in Tools**, **Execution Units**, **Memory Connectors**.
- **D-33:** MCP server registration follows full MCP spec: name, description, transport (stdio | http | sse), command OR URL, env vars ref, auth method, auto-discovered tool list (fetched on successful connect), connection status badge.
- **D-34:** Agent config card "Tools" field: multi-select dropdown grouped by registered connector. Each tool is individually checkable (e.g., MCP server "github" exposes `search_repos`, `create_issue`, `read_file` — user checks the ones they want).
- **D-35:** Selected tools appear as chips in the config card AND render as **visual tool-attachment edges** on the canvas (driven by config, read-only on canvas).
- **D-36:** Execution units registered in the **Execution Units tab** of Connectors page: name, sandbox mode, image/runtime, resource limits, entry point. Picked via the same multi-select mechanism as tools.
- **D-37:** Tool attachment on canvas uses a **distinct 'tool' port type + edge type**. Agent nodes gain a dedicated 'tools' input port (left side, separate from data/control ports). Tool edges rendered dashed, distinct color (TBD — planner chooses consistent with existing port palette).
- **D-38:** Execution units use the **same** 'tool' port/edge mechanism but render with their existing sandbox badge so they're visually distinct from MCP tools.
- **D-39:** Backend: new `/api/studio/v1/connectors/mcp`, `/api/studio/v1/connectors/execution-units`, `/api/studio/v1/connectors/memory` CRUD endpoints. MCP discovery via existing MCP client in Zeroth (researcher to locate).

### Memory & Messages Connectors (AUTH-03)
- **D-40:** Memory connectors (semantic retrieval) and messages connectors (conversation history) are **separate concepts** with separate fields on the agent config card.
- **D-41:** Memory: existing **canvas Memory Resource node** (Phase 22) stays as visualization. Agent config "Memory" field references a connected Memory Resource node by ID. Canvas draws a dashed teal memory edge from agent to memory node. The Memory Resource node itself gets a config card adding: connector reference (from Connectors → Memory tab), TTL, namespace.
- **D-42:** Messages connectors work the same way but can also come from the registry directly (no canvas node required for simple cases) — agent config has a "Messages" field picking from registered messages connectors.
- **D-43:** **Thread key** field on agent config card: templatized string like `{{inputs.user_id}}` or static literal. Drives per-thread message history scoping. Enables multi-user workflows. Matches Zeroth's thread model (`src/zeroth/runs/models.py` Thread class).

### Environment Variables & Secrets (AUTH-01 prerequisite)
- **D-44:** New **Studio Settings** area with tabs: **Environment Variables**, **Models**, (Connectors and Prompts may be top-level or tabbed — planner decides).
- **D-45:** Env var fields: name (e.g., `ANTHROPIC_API_KEY`), value (write-only, masked `••••` after save), description, scope (`tenant` | `workflow` — workflow scope is future; tenant is Phase 24 default), created_at, updated_at.
- **D-46:** **Secrets storage: reuse Zeroth's existing secrets service** (`src/zeroth/secrets/`). Env var CRUD writes secrets to this backend, stores only the reference/ID in the studio DB. Values never returned to frontend after creation. Researcher to validate the existing secrets service supports the required operations (create, update, reference by name, tenant isolation).
- **D-47:** Env var references in config fields use **by-name autocomplete**: when editing an env-var-ref field (API keys, base URLs with secrets), user sees a dropdown of registered env var names. Backend resolves name → secret value at runtime.
- **D-48:** Validation: "Test connection" button on each env-var-consuming config (e.g., model registry entry, MCP server entry). Test action calls backend which uses the referenced secret to attempt a real connection. Env page itself has no standalone test (context-specific testing is more useful).
- **D-49:** Backend: new `/api/studio/v1/env-vars` CRUD endpoints. Writes pass through to `src/zeroth/secrets/` service.

### Typed Ports & Data Flow Visualization (AUTH-04)
- **D-50:** Port hover tooltips show: port label, port type (data | control | tool | memory | messages | any), expected schema (if defined by node type), **and the last-run observed value from that port** when run data is available. Makes data flow concrete and aids debugging.
- **D-51:** Typed edges carry a **small inline label** near the midpoint showing the data type (e.g., `string`, `User`, `ToolResult`). Visible subtly at rest, fully visible on hover or when the edge is selected. Doesn't clutter the canvas.
- **D-52:** Port types extended from Phase 22's {`data`, `control`, `memory`, `any`} to add **`tool`** and **`messages`**. Update `apps/studio/src/types/nodes.ts` `PortType` union.

### Claude's Discretion
- Exact WebSocket event buffer size (target ~500; planner/researcher refines based on expected event volume)
- Exact colors for status badges, tool edges, memory edges, messages edges (consistent with existing Tailwind palette)
- Animation for status badge transitions (running → complete)
- Icon choices for new tabs (Models, Prompts, Connectors, Env Vars)
- Exact layout of the Variable Bindings section in agent config card
- JSON schema editor widget — simple textarea with validation vs full structured editor (recommend simple for v2.0)
- Step timeline visual style in Results tab (vertical list with expand chevrons recommended)
- How "tool" ports position on agent node (left, right, bottom — recommend a distinct side to separate from data ports)
- Debounce/throttle for WebSocket event handling on the frontend
- CodeMirror theme (should match existing Tailwind dark/light modes)
- Whether the Inputs modal supports JSON paste as a fallback to the form
- Exact error-to-suggested-fix mapping logic (start with 3-5 common errors, extend as needed)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Product & UX Direction
- `docs/superpowers/specs/2026-03-29-zeroth-studio-design.md` — Validated Studio UX decisions (canvas-first, n8n-style editor posture)
- `.planning/PROJECT.md` — Principles, non-negotiables, out-of-scope list (no LLM streaming on canvas, not a chatbot builder)

### Planning & Scope
- `.planning/REQUIREMENTS.md` — Requirement IDs covered: API-02, API-04, AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
- `.planning/ROADMAP.md` §Phase 24 — Phase 24 goal, success criteria, dependencies

### Prior Phase Context (must understand before extending)
- `.planning/phases/22-canvas-foundation-dev-infrastructure/22-CONTEXT.md` — Vue Flow, 8 node types, Pinia stores, three-panel layout, REST-only posture
- `.planning/phases/23-canvas-editing-ux/23-CONTEXT.md` — Command pattern undo/redo, inspector auto-form pattern, validation indicator pattern

### Frontend Integration Points (Phase 22/23 code that Phase 24 extends)
- `apps/studio/src/api/client.ts` — REST fetch wrapper; Phase 24 adds a sibling `websocket.ts` client
- `apps/studio/src/stores/canvas.ts` — Canvas Pinia store; extend with run status per node
- `apps/studio/src/stores/workflow.ts` — Workflow store; extend to track current run, last run hydration
- `apps/studio/src/types/nodes.ts` — `NODE_TYPE_REGISTRY`, `PortType` union (add `tool`, `messages`), `PropertyFieldType` (add `model-select`, `prompt-ref`, `tool-multi-select`, `memory-ref`, `env-var-ref`, `json-schema`)
- `apps/studio/src/composables/usePortValidation.ts` — Extend for new port types and tool-edge validation
- `apps/studio/src/composables/useNodeValidation.ts` — Extend for agent config card variable-binding validation
- `apps/studio/src/components/canvas/StudioCanvas.vue` — Main canvas component; wire run status, tool edges, edge labels
- `apps/studio/src/components/canvas/CanvasControls.vue` — Add Run/Stop button and Inputs modal trigger
- `apps/studio/src/components/nodes/BaseNode.vue` — Add status badge, token/cost/model/latency chips
- `apps/studio/src/components/nodes/AgentNode.vue` — Add 'tools' port, connect to agent config card
- `apps/studio/src/components/inspector/NodeInspector.vue` — Add Results tab; extend Properties tab with new field types
- `apps/studio/src/components/inspector/InspectorField.vue` — Add new field type renderers

### Backend Integration Points
- `src/zeroth/service/studio_api.py` §lines 197-299 — Existing studio CRUD; add WebSocket endpoint, env var CRUD, model CRUD, prompt CRUD, connector CRUD
- `src/zeroth/service/run_api.py` §lines 97-141 — `POST /runs` (existing), `GET /runs/{run_id}` — Phase 24 adds `POST /runs/{run_id}/cancel` and `GET /runs?workflow_id=X&latest=true`
- `src/zeroth/service/app.py` — Main FastAPI app; register new routers and WebSocket routes
- `src/zeroth/runs/models.py` — `Run`, `RunStatus` (PENDING, RUNNING, WAITING_APPROVAL, COMPLETED, FAILED, WAITING_INTERRUPT), `Thread` — Phase 24 taps into these for event emission
- `src/zeroth/runs/repository.py` — Run repository; add query for latest run by workflow
- `src/zeroth/orchestrator/runtime.py` — Orchestrator; emit per-node events during execution
- `src/zeroth/dispatch/worker.py`, `src/zeroth/dispatch/arq_wakeup.py` — Worker loop that executes runs; hook for publishing events to WebSocket channel
- `src/zeroth/agent_runtime/provider.py` §lines 139-227 — `LiteLLMProviderAdapter` (provider format `{vendor}/{model}`); Phase 24 model registry wraps this
- `src/zeroth/agent_runtime/runner.py` — Agent execution; emit step events (LLM call, tool call, final output)
- `src/zeroth/secrets/` — **Existing secrets service** — env var CRUD must write here. Researcher confirms available operations.
- `src/zeroth/memory/` — Memory connector backends
- (MCP client location — researcher to locate; Zeroth may use an existing Python MCP library)

### External Standards
- Model Context Protocol spec — MCP server registration fields, transport types, tool discovery
- LiteLLM provider string format — `{provider}/{model}` (e.g., `anthropic/claude-sonnet-4-5-20250514`)
- Jinja2 variable syntax reference — for prompt templating

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `apps/studio/src/api/client.ts` — REST wrapper; mirror for WebSocket client
- `apps/studio/src/components/inspector/NodeInspector.vue` + `InspectorField.vue` — Auto-form pattern from Phase 23; extend rather than rebuild
- `apps/studio/src/composables/useNodeValidation.ts` + `usePortValidation.ts` — Validation indicator infrastructure ready to reuse for variable-binding validation
- `apps/studio/src/stores/canvas.ts` — Command pattern store from Phase 23; execution state additions need to coexist with undo/redo (run state is NOT undoable)
- `src/zeroth/service/run_api.py` — `POST /runs`, `GET /runs/{run_id}` already exist; extend with cancel + query
- `src/zeroth/runs/models.py` — `RunStatus` enum already covers the states needed (RUNNING, COMPLETED, FAILED, etc.)
- `src/zeroth/agent_runtime/provider.py` — LiteLLM adapter already exists; model registry is a thin layer over this
- `src/zeroth/secrets/` — Existing secrets service; reuse rather than rebuild
- dagre, `@vue-flow/core`, Pinia, Tailwind — all already installed from Phase 22

### Established Patterns
- Pinia stores with `defineStore` composable pattern
- Vue 3 `<script setup>` composition API
- Auto-generated inspector forms from node type schema (Phase 23)
- Command pattern with undo/redo for canvas mutations (Phase 23) — run-state mutations are NOT commands (not undoable)
- Tailwind CSS with existing color palette
- Component-per-node-type with BaseNode base
- REST paths under `/api/studio/v1/*`
- LiteLLM provider strings as the canonical model identifier
- Zeroth governance: secrets referenced by ID, never materialized in API responses

### Integration Points
- `BaseNode.vue` — Add status badge + post-run chips overlay. Must not break Phase 23 validation indicator styling (red border coexists with status badge).
- `StudioCanvas.vue` — Render new tool edges, edge labels, handle new port types. Coordinate with Vue Flow edge types.
- `CanvasControls.vue` — Add Run/Stop button. Layout adjustment for new button.
- `NodeInspector.vue` — Add Results tab. Tab switcher added; default tab remains Properties.
- `WorkflowRail.vue` — May need new entries for top-level nav (Prompts, Connectors, Settings) — coordinate with existing rail design.
- `src/zeroth/service/app.py` — Register new routers. Add WebSocket endpoint registration pattern (first WS in the project).
- `src/zeroth/orchestrator/runtime.py` + `src/zeroth/dispatch/worker.py` — Hook execution lifecycle to event publication (pub/sub pattern — Redis? in-memory? researcher to recommend).
- `src/zeroth/secrets/` — Add any missing CRUD operations needed by Phase 24 env var page.

</code_context>

<specifics>
## Specific Ideas

- **Agent config card is the primary authoring surface** — users described it as a "card-like form" when creating an agent node. Everything an agent needs (model, prompts, tools, memory, messages, thread key, env vars) lives here. This is the inspector Properties tab for agent nodes, enriched with rich field types.
- **Bring-your-own-credentials model** — users register their own models, API keys, and MCP servers. Zeroth does not ship pre-configured providers. This shifts Phase 24 from "use what's configured" to "user configures everything via Studio."
- **Prompt store is first-class** — prompts are reusable artifacts with versions, not inline strings. This mirrors how production prompt tooling works (LangSmith, Humanloop, Portkey).
- **Jinja-style templating with explicit binding** — variables auto-detected in the prompt, then explicitly bound to upstream outputs in the agent config. Prevents silent "variable not in scope" bugs.
- **Tool edges as distinct port/edge type** — users want visual distinction between data flow and capability attachment. New 'tool' port, dashed edges, driven by config not manual drawing.
- **Messages ≠ Memory** — users explicitly want conversation history separate from semantic memory. Two fields, two connector types.
- **Thread key templatization** — `{{inputs.user_id}}` for per-user scoping. Matches Zeroth's existing Thread model.
- **Fail-fast governance posture** — first failure halts run; downstream marked skipped. No partial-success runs in Phase 24.
- **Reuse Zeroth secrets service** — user explicitly wants "whatever is safest for secret management, maybe some dedicated service" — existing `src/zeroth/secrets/` fits.
- **Step-timeline for multi-turn / tool calls** — not a chat transcript (conflicts with "not a chatbot builder" principle), but an ordered expandable timeline.
- **Four persistent chips on agent nodes** — status, tokens+cost, model used, latency. Multi-chip density is acceptable because agents are information-dense nodes.

</specifics>

<deferred>
## Deferred Ideas

- **Per-node retry** — retry a single failed node without re-running the whole workflow. Requires backend resume logic. Defer to post-v2.0 backlog.
- **Workflow-level default model** — inherit model from workflow settings unless overridden per node. Defer; Phase 24 is per-node only.
- **Full git-style prompt versioning with diffs and branches** — simple linear versioning is enough for v2.0.
- **Prompt diff view** — side-by-side prompt version comparison. Defer.
- **WAITING_INTERRUPT pause-and-resume UI** — backend supports this state, but Phase 24 uses fail-fast only. Interactive pause-and-resume is a future enhancement.
- **Multiple parallel runs of the same workflow** — Phase 24 assumes one active run per workflow at a time.
- **Workflow-scoped env vars** — Phase 24 env vars are tenant-scoped only. Workflow scope is a future enhancement.
- **"Test" button on env var page itself** — testing happens at the consuming config (model, connector) instead.
- **JSON paste fallback for Inputs modal** — form-only in Phase 24; paste comes later if needed.
- **Run history dropdown** — Phase 24 auto-loads the latest run only. Multi-run history view is a future feature (ties into Phase 26 collaboration / audit).

</deferred>

---

*Phase: 24-execution-ai-authoring*
*Context gathered: 2026-04-10*
