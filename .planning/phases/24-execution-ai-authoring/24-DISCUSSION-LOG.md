# Phase 24: Execution & AI Authoring - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-10
**Phase:** 24-execution-ai-authoring
**Areas discussed:** WebSocket + execution trigger, Model selector & provider catalog, Prompt editor (CodeMirror), Tool attachment + port tooltips, Post-execution display, MCP + tool connector registry, Memory / messages connector binding, Env settings page & secret UX, Run error handling & control

---

## Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| WebSocket + execution trigger | WS shape, Run button, status badges, results UI | ✓ |
| Model selector & provider catalog | Model source, workflow default, selector UI | ✓ |
| Prompt editor (CodeMirror) | Templating, variable binding, versioning | ✓ |
| Tool attachment + port tooltips | Edge type, exec units, hover tooltips, edge labels | ✓ |

---

## WebSocket + Execution Trigger

### Q1: How should the WebSocket connection be scoped?

| Option | Description | Selected |
|--------|-------------|----------|
| Per-workflow | `/workflows/{id}/events`, simpler routing, clean teardown | ✓ |
| Per-run | `/runs/{run_id}/events`, opens only after triggering | |
| Global session | One multiplexed WS, better for collab but complex | |

**User's choice:** Per-workflow (recommended)

### Q2: Where does the Run button live and how are inputs collected?

| Option | Description | Selected |
|--------|-------------|----------|
| Canvas toolbar + inputs modal | Run in CanvasControls, modal form for Start inputs | ✓ |
| Studio header + inline input panel | Top header button, bottom collapsible panel | |
| Right-click Start node | Trigger via Start node context menu | |

**User's choice:** Canvas toolbar + inputs modal (recommended)

### Q3: Where do per-node status badges and full results appear?

| Option | Description | Selected |
|--------|-------------|----------|
| Badges on node + details in inspector tab | Status overlay on nodes, Results tab in inspector | ✓ |
| Badges on node + bottom results drawer | Fourth panel region for run results | |
| Overlay chips + modal drill-down | Floating chips, modal for details | |

**User's choice:** Badges on node + details in inspector tab (recommended)

### Q4: How should the frontend reconnect if WebSocket drops mid-run?

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-reconnect + replay from last event | Seq-based replay, requires buffer | ✓ |
| Auto-reconnect + full state refresh | Refetch via REST, brief flicker | |
| Manual reconnect prompt | Banner + button | |

**User's choice:** Auto-reconnect + replay from last event (recommended)

---

## Model Selector & Provider Catalog

### Q1: Where does the model list come from?

| Option | Description | Selected |
|--------|-------------|----------|
| Backend /models endpoint | From LiteLLM config + env keys | ✓ |
| Static curated list in frontend | Hardcoded in nodes.ts | |
| Hybrid — static defaults + backend override | Offline fallback + runtime accuracy | |

**User's choice:** Backend /models endpoint (recommended)
**Notes:** Reframed in follow-up — endpoint serves **user-registered** model configurations, not server env. Users configure models themselves.

### Q2: Can the workflow have a default model used by all agents unless overridden?

| Option | Description | Selected |
|--------|-------------|----------|
| Per-node only | Each agent picks its own | (partial) |
| Workflow default + per-node override | Workflow metadata default_model | |

**User's choice:** Free-text — "the user has to set it make it, the parameters for it and the api key required"
**Notes:** Triggered major scope expansion. Confirmed in follow-up: users register model configs (name, provider, base model, api key from env settings, temp, max_tokens, top_p, custom URL, label, structured schema or text, tool/MCP connectors) via agent config card.

### Q3: What does the model selector show per model?

| Option | Description | Selected |
|--------|-------------|----------|
| Grouped by provider with context window + cost hints | Rich dropdown | ✓ |
| Flat list, name only | Minimal | |
| Searchable combobox with descriptions | For many models | |

**User's choice:** Grouped by provider with context window + cost hints (recommended)

### Q4: Where does the model selector live within the inspector?

| Option | Description | Selected |
|--------|-------------|----------|
| Inspector field for the agent node | Dynamic selector widget replacing hardcoded select | ✓ |
| Dedicated model section at top of inspector | Prominent separate section | |

**User's choice:** Inspector field for the agent node (recommended)

---

## Scope Clarification Round (after Q2 free-text answer)

Multi-part free-text response:

1. **Agent config card form** — card-like form on node creation with: required parameters, memory/messages connectors, input spec, system/user prompts from prompt store, API keys from user-defined env settings.
2. **Parameter list** — temp, max tokens, top_p, custom URL, label, structured schema or text, tool/MCP connectors.
3. **Secrets** — "whatever is safest for secret management, maybe some dedicated service" → interpreted as: reuse Zeroth's existing `src/zeroth/secrets/` service.
4. **Scope expansion** — "all good" to expanding Phase 24 to include prompt store, env settings, model registry, connector registry, agent config card.

**Also queued:** "Later we will talk about what agent nodes can display after executions" → addressed as follow-up after Area 4.

---

## Prompt Editor (CodeMirror) — reframed as Prompt Store

### Q1: Where and how are prompts authored in the prompt store?

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated Prompts page in Studio | New left-nav entry with editor | |
| Inline modal from agent config card | No dedicated page | |
| Both — dedicated page + inline create shortcut | Primary page + inline shortcut | ✓ |

**User's choice:** Both — dedicated page + inline create shortcut

### Q2: What templating language does the CodeMirror prompt editor support?

| Option | Description | Selected |
|--------|-------------|----------|
| Jinja-style {{variable}} with upstream node refs | Matches LangChain conventions | ✓ |
| Plain text with $variable syntax | Simpler | |
| Plain text, no templating | Variables as separate fields | |

**User's choice:** Jinja-style {{variable}} (recommended)

### Q3: How are prompt variables validated and bound to upstream data?

| Option | Description | Selected |
|--------|-------------|----------|
| Auto-detect placeholders + bind via agent config | Explicit variable bindings section | ✓ |
| Free-form — runtime resolves at execution | No UI validation | |
| Implicit — all upstream node outputs in scope | No explicit binding | |

**User's choice:** Auto-detect placeholders + bind via agent config (recommended)

### Q4: Do prompts support versioning / history in the store?

| Option | Description | Selected |
|--------|-------------|----------|
| Simple versioning — each save creates a new version | Linear history | ✓ |
| No versioning — prompts are mutable | Simplest | |
| Full git-style versioning | Overkill for v2.0 | |

**User's choice:** Simple versioning (recommended)

---

## Tool Attachment + Port Tooltips

### Q1: How do tools / execution units attach to agent nodes on the canvas?

| Option | Description | Selected |
|--------|-------------|----------|
| Distinct 'tool' edge type with dedicated agent port | New port type, dashed edges | ✓ |
| Config-only via agent config card | No canvas edge | |
| Hybrid — config for registry + canvas edge visualization | Config-driven read-only edges | |

**User's choice:** Distinct 'tool' edge type with dedicated agent port (recommended)

### Q2: Are execution units treated the same as tools for attachment?

| Option | Description | Selected |
|--------|-------------|----------|
| Same mechanism, distinct category | Same port/edge, different category in picker | ✓ |
| Separate port types 'tool' and 'exec_unit' | Doubles validation rules | |
| Execution units only attach via data edges | Keep current model | |

**User's choice:** Same mechanism, distinct category (recommended)

### Q3: What info appears on port hover tooltips (AUTH-04)?

| Option | Description | Selected |
|--------|-------------|----------|
| Type + sample + last-run value | Label, type, schema, observed value | ✓ |
| Type + schema only | No run data | |
| Type only | Label + type | |

**User's choice:** Type + sample + last-run value (recommended)

### Q4: How do typed edge labels show data type between nodes?

| Option | Description | Selected |
|--------|-------------|----------|
| Small inline label mid-edge, visible on hover/select | Subtle at rest | ✓ |
| Always-visible chip on edges | Clearer, busier | |
| No inline labels — only in hover tooltips | Cleanest | |

**User's choice:** Small inline label mid-edge (recommended)

---

## Post-Execution Display (follow-up to Area 1)

### Q1: What persistent indicators should agent nodes show on the canvas after a run? (multi-select)

| Option | Description | Selected |
|--------|-------------|----------|
| Status badge (last run outcome) | Success/failed/skipped | ✓ |
| Token count + cost chip | Tokens and cost | ✓ |
| Model used chip | Actual model string | ✓ |
| Latency indicator | Time taken | ✓ |

**User's choice:** ALL four selected.

### Q2: What should the inspector 'Results' tab show for an agent node after execution?

| Option | Description | Selected |
|--------|-------------|----------|
| Full breakdown: rendered prompt, raw output, tool calls, tokens, cost, latency | Debugging-first | ✓ |
| Output + key stats only | Cleaner | |
| Collapsible sections — user picks | Flexible | |

**User's choice:** Full breakdown (recommended)

### Q3: How should multi-turn / tool-calling agent runs be displayed?

| Option | Description | Selected |
|--------|-------------|----------|
| Step-by-step timeline in Results tab | Ordered expandable steps | ✓ |
| Final output only, steps in separate audit view | Cleaner main view | |
| Flat chat-log style | Conflicts with non-chatbot stance | |

**User's choice:** Step-by-step timeline (recommended)

### Q4: Should results persist across page reloads?

| Option | Description | Selected |
|--------|-------------|----------|
| Fetch latest run from backend on canvas open | Hydrate from GET /runs | ✓ |
| Session-only — results clear on reload | Simpler | |
| User picks a run from a dropdown | More powerful | |

**User's choice:** Fetch latest run from backend on canvas open (recommended)

---

## MCP + Tool Connector Registry

### Q1: Where are MCP servers / tools registered?

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated 'Connectors' settings page | Tabs for MCP / tools / exec units | ✓ |
| Inline in agent config card only | No central registry | |
| Per-workflow tool list in workflow settings | Workflow-scoped | |

**User's choice:** Dedicated 'Connectors' settings page (recommended)

### Q2: What fields does a registered MCP server have?

| Option | Description | Selected |
|--------|-------------|----------|
| Full MCP spec compliance | Transport, URL/command, auth, auto-discovery | ✓ |
| Minimal — name + URL + API key ref | Essentials only | |
| MCP + built-in tool shims in same form | Unified form | |

**User's choice:** Full MCP spec compliance (recommended)

### Q3: How does the agent config card pick tools from the registry?

| Option | Description | Selected |
|--------|-------------|----------|
| Multi-select dropdown grouped by connector | Checkable tools | ✓ |
| Tree view with hierarchical expand | Heavier UI | |
| Search-first picker modal | For many tools | |

**User's choice:** Multi-select dropdown grouped by connector (recommended)

### Q4: How do execution units fit into this registry?

| Option | Description | Selected |
|--------|-------------|----------|
| Register in the same 'Connectors' page, separate tab | Unified surface | ✓ |
| Execution units stay canvas-only | No registry | |
| Hybrid — registered + canvas both | Flexible | |

**User's choice:** Register in the same 'Connectors' page, separate tab (recommended)

---

## Memory / Messages Connector Binding

### Q1: How are memory connectors referenced by agent nodes?

| Option | Description | Selected |
|--------|-------------|----------|
| Canvas Memory Resource node + agent config reference | Keeps Phase 22 visualization | ✓ |
| Config-only from Connectors page | Deprecate canvas node | |
| Both — registry definitions + canvas instances | Most complex | |

**User's choice:** Canvas Memory Resource node + agent config reference (recommended)

### Q2: What is a 'messages connector' and how does it differ from memory?

| Option | Description | Selected |
|--------|-------------|----------|
| Conversation history store — separate from semantic memory | Two connector types | ✓ |
| Same thing — one 'memory' field covers both | Unified | |
| Messages is built-in, only memory is pluggable | Hybrid | |

**User's choice:** Conversation history store — separate from semantic memory (recommended)

### Q3: How is the message history / memory scope configured per agent?

| Option | Description | Selected |
|--------|-------------|----------|
| Thread key from upstream data | Templatized thread ID | ✓ |
| Static key per workflow | Simpler | |
| Auto-generated per run | Stateless | |

**User's choice:** Thread key from upstream data (recommended)

### Q4: How does the Memory Resource canvas node interact with Phase 24 changes?

| Option | Description | Selected |
|--------|-------------|----------|
| Keep as-is, add config fields | Extend existing | ✓ |
| Replace with registry entries only | Remove canvas node | |
| Split — registry for types, canvas for instances | Hybrid | |

**User's choice:** Keep as-is, add config fields (recommended)

---

## Env Settings Page & Secret UX

### Q1: Where does the env settings page live?

| Option | Description | Selected |
|--------|-------------|----------|
| Studio Settings → Environment Variables tab | Unified settings area | ✓ |
| Top-level 'Env' left-nav entry | Flat nav | |
| Per-workflow env vars | Workflow-scoped | |

**User's choice:** Studio Settings → Environment Variables tab (recommended)

### Q2: What fields does each env variable entry have?

| Option | Description | Selected |
|--------|-------------|----------|
| Name + value (masked) + description + scope | Rich metadata | ✓ |
| Minimal — just name + value | Simplest | |
| Full audit — last-used-by tracking | Tracks usage | |

**User's choice:** Name + value (masked) + description + scope (recommended)

### Q3: How are env vars referenced from agent configs?

| Option | Description | Selected |
|--------|-------------|----------|
| By name string with autocomplete | Dropdown of registered names | ✓ |
| By UUID (hidden from user) | Safer from rename breakage | |
| Template-style {{env.VAR}} | Inline templating | |

**User's choice:** By name string with autocomplete (recommended)

### Q4: Can env vars be validated / tested before saving?

| Option | Description | Selected |
|--------|-------------|----------|
| 'Test connection' button per env-consuming config | Context-specific testing | ✓ |
| Test button on the env var itself | Self-contained | |
| No testing — runtime failures only | Simplest | |

**User's choice:** 'Test connection' button per env-consuming config (recommended)

---

## Run Error Handling & Control

### Q1: What happens to downstream nodes when a node fails mid-run?

| Option | Description | Selected |
|--------|-------------|----------|
| Stop the entire run | Fail-fast, downstream skipped | ✓ |
| Continue with partial results | Null upstream data | |
| Pause the run and wait for user action | WAITING_INTERRUPT state | |

**User's choice:** Stop the entire run (recommended)

### Q2: How does the Run button behave during an active run?

| Option | Description | Selected |
|--------|-------------|----------|
| Becomes 'Stop' | Toggle to red Stop | ✓ |
| Disabled + secondary 'Stop' button | Two buttons | |
| Always 'Run' — parallel runs | No stop | |

**User's choice:** Becomes 'Stop' (recommended)

### Q3: Can the user retry a single failed node?

| Option | Description | Selected |
|--------|-------------|----------|
| Not in Phase 24 — full re-run only | Defer to backlog | ✓ |
| Yes — 'Retry this node' button | Requires backend resume | |
| Yes — 'Retry from here' | Middle ground | |

**User's choice:** Not in Phase 24 — full re-run only (recommended)

### Q4: How are errors displayed in the Results tab?

| Option | Description | Selected |
|--------|-------------|----------|
| Structured error: type + message + stack + suggested fix | Rich error UI | ✓ |
| Raw error message + stack only | Simple | |
| Error summary + link to audit log | Thin layer | |

**User's choice:** Structured error (recommended)

---

## Claude's Discretion

Captured throughout as D-items and in the "Claude's Discretion" section of CONTEXT.md. Areas explicitly deferred to Claude/researcher/planner:

- WebSocket event buffer size
- Exact colors for status badges, tool edges, memory edges, messages edges
- Animation timing for status transitions
- Icon choices for new tabs
- Variable Bindings section layout
- JSON schema editor widget (simple vs full)
- Step timeline visual style
- Tool port position on agent node
- Debounce/throttle for WS events
- CodeMirror theme
- Inputs modal JSON paste fallback
- Error-to-suggested-fix mapping specifics

## Deferred Ideas

All captured in CONTEXT.md `<deferred>` section. Summary:

- Per-node retry — future backlog
- Workflow-level default model — future enhancement
- Git-style prompt versioning with diffs — overkill for v2.0
- WAITING_INTERRUPT pause-and-resume UI — future
- Parallel runs of same workflow — future
- Workflow-scoped env vars — future
- Env var test button on page itself — context-specific testing instead
- JSON paste fallback for Inputs modal — later
- Run history dropdown — Phase 26 / post-v2.0

