---
phase: 44-declarative-template-memory-sourcing
type: context
status: completed
completed_at: 2026-04-15
---

# Phase 44 Context: Declarative Template Memory Sourcing

## Problem Statement

Agent prompt templates (Phase 36) rendered with `"memory": {}` hardcoded. No mechanism existed for an agent node to declare which memory connector values should populate `{{ memory.* }}` in its instruction template. Graph authors had to use dedicated memory-read nodes before each agent, adding noise to graph topologies for a common pattern.

## Goal

Let `AgentNodeData` carry a `template_memory_bindings` list so the orchestrator can declaratively fetch memory values and inject them into the template `memory` namespace at dispatch time — no extra graph nodes required.

## Decision Register

| ID | Decision | Rationale |
|----|----------|-----------|
| D-01 | `TemplateMemoryBinding` as a standalone Pydantic model on `AgentNodeData` | Composable, serialisable, survives graph persistence unchanged. |
| D-02 | `connector_instance_id` must reference a name already in `memory_refs` | Reuses the existing connector resolution path; no parallel wiring needed. |
| D-03 | Scope (`run`/`thread`/`shared`) declared per-binding, not per-node | Different bindings may legitimately span scopes. |
| D-04 | `scan` mode implemented as `connector.search({}, scope)` + client-side prefix filter | No native prefix scan on the connector protocol; empty-query search returns all entries. |
| D-05 | `thread_id` resolution moved to before the template block in `_dispatch_node` | THREAD-scoped bindings need `thread_id` to resolve the ScopedMemoryConnector target before render_vars is built. |
| D-06 | `_substitute_tmb_key` uses regex for `{input.field}`, `{state.field}`, `{run.run_id}` placeholders | Safe, no eval — unknown placeholders left unchanged for debugging visibility. |
| D-07 | `MemoryBindingResolutionError` subclasses `OrchestratorError` (not `NodeDispatcherError`) | Not a node-type dispatch failure; it is a pre-dispatch resolution failure. `_drive` retry logic operates on `OrchestratorError` so the error is correctly terminal. |
| D-08 | SHARED scope used for test data pre-population | Avoids run_id dependency — `ScopedMemoryConnector` resolves SHARED to `"__shared__"` unconditionally, allowing pre-population before the run starts. |
| D-09 | `from governai.memory.models import MemoryScope` imported inside `_resolve_template_memory` | Avoids a hard module-level governai dependency in the orchestrator; lazy import matches the existing Phase 36 pattern for template imports. |
| D-10 | TMB audit records stored under `execution_metadata.template_memory_bindings` in the node audit record | Consistent with `rendered_prompt` and `context_window` audit enrichments from Phases 36 and 37. |

## Deferred

- `key_prefix` substitution using `{state.field}` placeholders — implemented for completeness via `_substitute_tmb_key` but no dedicated test for scan-mode prefix substitution.
- THREAD-scoped binding integration test — thread setup requires `RepositoryThreadResolver` wiring; deferred. Unit tests cover THREAD scope mapping through `_SCOPE_MAP`.
- `max_items` ordering guarantee — truncation is deterministic (dict insertion order from `connector.search`), but ordering is backend-defined. A future plan can add an `order_by` field.
