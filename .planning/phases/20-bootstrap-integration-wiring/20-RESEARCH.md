# Phase 20: Bootstrap Integration Wiring - Research

**Researched:** 2026-04-09
**Domain:** Service bootstrap wiring, dependency injection for AgentRunner
**Confidence:** HIGH

## Summary

Phase 20 closes two high-severity integration gaps identified in the v1.1 milestone audit: INT-01 (MemoryConnectorResolver not wired) and INT-02 (BudgetEnforcer not injected into AgentRunner). Both components exist and are fully implemented -- they simply are not connected at runtime. The bootstrap creates them and stores them on ServiceBootstrap, but never passes them through to the point where AgentRunner instances actually use them.

The core challenge is that `agent_runners` are passed INTO `bootstrap_service()` as a pre-built `Mapping[str, AgentRunner]` parameter. Bootstrap does not construct AgentRunner instances -- it receives them. This means injection must happen either (a) at dispatch time in the orchestrator, following the existing pattern used for InstrumentedProviderAdapter wrapping (lines 252-268 of runtime.py), or (b) by storing the resolver/enforcer on the orchestrator and having it set them on each runner before calling `run()`.

**Primary recommendation:** Follow the existing orchestrator dispatch-time injection pattern (same as ECON-01 cost instrumentation). Store `memory_resolver` and `budget_enforcer` on `RuntimeOrchestrator`, inject them onto the AgentRunner at dispatch time in `_dispatch_node`, and restore originals after execution.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| MEM-01 | Redis-backed key-value memory connector | Already implemented; needs MemoryConnectorResolver wired to make reads/writes resolve to real connectors |
| MEM-02 | Redis-backed conversation/thread memory connector | Already implemented; same resolver wiring needed |
| MEM-03 | pgvector-backed semantic memory connector | Already implemented; same resolver wiring needed |
| MEM-04 | ChromaDB memory connector | Already implemented; same resolver wiring needed |
| MEM-05 | Elasticsearch memory connector | Already implemented; same resolver wiring needed |
| MEM-06 | GovernAI bridged memory connectors | Already implemented; resolver wiring connects Scoped/Auditing wrappers |
| ECON-03 | Per-tenant budget caps enforced via policy guard before execution | BudgetEnforcer exists; needs injection into AgentRunner so check_budget fires pre-execution |
</phase_requirements>

## Architecture Patterns

### Current State: The Integration Gap

```
bootstrap_service()
  |
  +-- Creates memory_registry (InMemoryConnectorRegistry)      <-- EXISTS
  +-- Calls register_memory_connectors(memory_registry, ...)   <-- EXISTS
  +-- Creates budget_enforcer (BudgetEnforcer)                  <-- EXISTS
  |
  +-- Stores on ServiceBootstrap.memory_registry                <-- EXISTS
  +-- Stores on ServiceBootstrap.budget_enforcer                <-- EXISTS
  |
  +-- NEVER creates MemoryConnectorResolver                     <-- GAP (INT-01)
  +-- NEVER injects budget_enforcer into AgentRunner            <-- GAP (INT-02)
  +-- NEVER injects memory_resolver into AgentRunner            <-- GAP (INT-01)
```

### Target State After Phase 20

```
bootstrap_service()
  |
  +-- Creates memory_registry, registers connectors             <-- no change
  +-- Creates MemoryConnectorResolver(registry=memory_registry) <-- NEW
  +-- Creates budget_enforcer                                   <-- no change
  +-- Stores memory_resolver on orchestrator                    <-- NEW
  +-- Stores budget_enforcer on orchestrator                    <-- NEW
  |
  RuntimeOrchestrator._dispatch_node()
  |
  +-- Injects runner.memory_resolver = self.memory_resolver     <-- NEW
  +-- Injects runner.budget_enforcer = self.budget_enforcer     <-- NEW
  +-- Calls runner.run(...)
  +-- Restores originals (same pattern as provider wrapping)    <-- NEW
```

### Pattern 1: Dispatch-Time Injection (Existing Pattern to Follow)

**What:** The orchestrator already injects InstrumentedProviderAdapter at dispatch time in `_dispatch_node` (runtime.py lines 252-268). It saves the original, replaces it, runs the agent, then restores.

**When to use:** When the dependency is created at bootstrap scope but consumed at agent-runner scope, and runners are externally provided.

**Example (existing code to mirror):**
```python
# Phase 18 pattern in _dispatch_node (runtime.py:252-268)
original_provider = runner.provider
if self.regulus_client is not None and self.cost_estimator is not None:
    runner.provider = InstrumentedProviderAdapter(...)
try:
    result = await self._run_agent_with_optional_enforcement(...)
finally:
    runner.provider = original_provider
```

**New code should follow identical structure:**
```python
# Save originals
original_memory_resolver = runner.memory_resolver
original_budget_enforcer = runner.budget_enforcer

# Inject bootstrap-scoped dependencies
if self.memory_resolver is not None:
    runner.memory_resolver = self.memory_resolver
if self.budget_enforcer is not None:
    runner.budget_enforcer = self.budget_enforcer

try:
    result = await self._run_agent_with_optional_enforcement(...)
finally:
    runner.memory_resolver = original_memory_resolver
    runner.budget_enforcer = original_budget_enforcer
    runner.provider = original_provider
```

### Pattern 2: MemoryConnectorResolver Construction

**What:** Create a MemoryConnectorResolver from the populated registry at bootstrap time.

**Example:**
```python
# In bootstrap_service(), after register_memory_connectors():
memory_resolver = MemoryConnectorResolver(
    registry=memory_registry,
    thread_repository=thread_repository,
)
orchestrator.memory_resolver = memory_resolver
```

### Anti-Patterns to Avoid

- **Modifying AgentRunner.__init__ signature:** The runner already accepts `memory_resolver` and `budget_enforcer` as optional kwargs. No changes needed to AgentRunner itself.
- **Creating resolver per-dispatch:** The resolver is stateless except for its registry reference. Create once at bootstrap, reuse across all dispatches.
- **Forgetting to restore originals:** The try/finally pattern is critical. Without it, a runner retains injected state from a previous dispatch.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Memory connector resolution | Custom lookup in orchestrator | MemoryConnectorResolver (already exists) | Handles Scoped/Auditing wrapper chain per GovernAI protocol |
| Budget checking | Inline HTTP calls in orchestrator | BudgetEnforcer (already exists) | Has TTL cache, fail-open semantics, proper error handling |

**Key insight:** All components already exist and are fully tested. This phase is purely wiring -- connecting existing parts with no new business logic.

## Common Pitfalls

### Pitfall 1: Forgetting Thread Repository on Resolver
**What goes wrong:** MemoryConnectorResolver tracks thread-memory bindings via ThreadRepository. If not provided, `_record_thread_binding` silently skips.
**Why it happens:** It's an optional parameter in the resolver constructor.
**How to avoid:** Pass `thread_repository=thread_repository` when constructing the resolver in bootstrap.
**Warning signs:** Memory reads/writes work but thread binding records are missing from the database.

### Pitfall 2: Budget Enforcer Tenant ID Resolution
**What goes wrong:** AgentRunner extracts tenant_id from enforcement_context. The orchestrator builds enforcement_context from run.metadata, which may not contain tenant_id.
**Why it happens:** tenant_id comes from run.metadata or defaults to "default" in the runner (line 133-136 of runner.py).
**How to avoid:** Ensure the enforcement_context passed to the runner includes the run's tenant_id. Check that `_enforcement_context_for` includes it, or that the run metadata has it.
**Warning signs:** All budget checks go against the "default" tenant instead of the actual tenant.

### Pitfall 3: Not Handling Missing agent_runners Entry
**What goes wrong:** If no AgentRunner is registered for a node_id, `self.agent_runners.get(node.node_id)` returns None and raises NodeDispatcherError before any injection happens.
**Why it happens:** Normal flow -- but tests need to account for this.
**How to avoid:** Injection code must come after the None check (after line 248).

### Pitfall 4: Orchestrator Dataclass Field Addition
**What goes wrong:** RuntimeOrchestrator is a `@dataclass(slots=True)` -- adding new fields requires them to have defaults or they break all existing instantiations.
**Why it happens:** Slots dataclasses have strict field ordering rules.
**How to avoid:** Add `memory_resolver` and `budget_enforcer` as optional fields with `None` defaults, positioned after existing optional fields.
**Warning signs:** TypeError at import/instantiation time about missing arguments.

## Code Examples

### Bootstrap Wiring (bootstrap.py changes)

```python
# After register_memory_connectors (around line 296):
memory_resolver = MemoryConnectorResolver(
    registry=memory_registry,
    thread_repository=thread_repository,
)

# Wire into orchestrator (after line 266):
orchestrator.memory_resolver = memory_resolver
orchestrator.budget_enforcer = budget_enforcer
```

### Orchestrator Injection (runtime.py changes)

```python
# In _dispatch_node, after runner is retrieved (after line 248):
original_memory_resolver = runner.memory_resolver
original_budget_enforcer = runner.budget_enforcer

if self.memory_resolver is not None:
    runner.memory_resolver = self.memory_resolver
if self.budget_enforcer is not None:
    runner.budget_enforcer = self.budget_enforcer

# Existing provider wrapping stays the same...
original_provider = runner.provider
# ... (existing InstrumentedProviderAdapter wrapping) ...

try:
    # ... existing execution code ...
finally:
    runner.provider = original_provider
    runner.memory_resolver = original_memory_resolver
    runner.budget_enforcer = original_budget_enforcer
```

### RuntimeOrchestrator New Fields

```python
@dataclass(slots=True)
class RuntimeOrchestrator:
    # ... existing fields ...
    # Phase 20: Memory and budget injection for AgentRunner
    memory_resolver: object | None = None
    budget_enforcer: object | None = None
```

## Files to Modify

| File | Change | Why |
|------|--------|-----|
| `src/zeroth/orchestrator/runtime.py` | Add `memory_resolver` and `budget_enforcer` fields; inject in `_dispatch_node` | Dispatch-time injection following existing pattern |
| `src/zeroth/service/bootstrap.py` | Create MemoryConnectorResolver; wire resolver + enforcer into orchestrator | Bootstrap-time construction and wiring |
| `tests/` (new test file) | Integration tests proving end-to-end wiring | Verify memory resolves to real connectors; verify budget rejection fires |

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Agent runners externally constructed | Orchestrator injects bootstrap-scoped deps at dispatch time | Phase 18 (provider wrapping) | Established pattern for dependency injection without changing runner construction |

## Open Questions

1. **ServiceBootstrap.memory_resolver field**
   - What we know: ServiceBootstrap has `memory_registry` but no `memory_resolver` field
   - What's unclear: Whether to add a `memory_resolver` field to ServiceBootstrap for external access
   - Recommendation: Add it for consistency and testability, even though the primary consumer is the orchestrator

## Project Constraints (from CLAUDE.md)

- Build/test: `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- Project layout: `src/zeroth/` main package, `tests/` pytest tests
- Progress logging mandatory via progress-logger skill
- Context efficiency: read only what's needed per task

## Sources

### Primary (HIGH confidence)
- Direct codebase analysis of `src/zeroth/service/bootstrap.py` (full file)
- Direct codebase analysis of `src/zeroth/agent_runtime/runner.py` (full file)
- Direct codebase analysis of `src/zeroth/orchestrator/runtime.py` (full file)
- Direct codebase analysis of `src/zeroth/memory/registry.py` (full file)
- Direct codebase analysis of `src/zeroth/econ/budget.py` (full file)
- `.planning/v1.1-MILESTONE-AUDIT.md` (gap definitions INT-01, INT-02, FLOW-07, FLOW-08)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - no new dependencies, all components exist
- Architecture: HIGH - follows established dispatch-time injection pattern from Phase 18
- Pitfalls: HIGH - identified from direct code analysis of existing patterns

**Research date:** 2026-04-09
**Valid until:** 2026-05-09 (stable internal wiring, no external dependencies)
