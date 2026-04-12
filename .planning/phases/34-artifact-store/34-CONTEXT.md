# Phase 34: Artifact Store - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a pluggable artifact store subsystem that lets nodes externalize large payloads into an external store (Redis or filesystem) instead of embedding them in run state. The artifact store provides TTL-based lifecycle management, audit-compatible references, and contract validation support for ArtifactReference types.

</domain>

<decisions>
## Implementation Decisions

### Store Interface Design
- **D-01:** ArtifactStore uses a Protocol (not ABC) — consistent with `ThreadStateStore`, `ProviderAdapter`, and `SecretProvider` patterns in the codebase. Methods: `async store(key, data, content_type, ttl) -> ArtifactReference`, `async retrieve(key) -> bytes`, `async delete(key) -> bool`, `async refresh_ttl(key, ttl) -> bool`, `async exists(key) -> bool`.
- **D-02:** Two implementations: `RedisArtifactStore` (uses SETEX for TTL) and `FilesystemArtifactStore` (stores files in a configurable directory with a metadata sidecar for TTL tracking).
- **D-03:** Key generation uses `{run_id}/{node_id}/{uuid}` pattern — namespaced by run for bulk cleanup on archive.
- **D-04:** Store selection is via `ArtifactStoreSettings` added to the unified `ZerothSettings` — follows the existing `RedisSettings`, `DatabaseSettings`, `MemorySettings` pattern.

### ArtifactReference Model
- **D-05:** `ArtifactReference` is a Pydantic model with fields: `store` (str — backend identifier), `key` (str — storage key), `content_type` (str — MIME type), `size` (int — bytes), `created_at` (datetime), `ttl_seconds` (int | None). It is JSON-serializable and can be embedded in node output payloads.
- **D-06:** ArtifactReference is placed in a new `zeroth.core.artifacts` package — follows the pattern of `zeroth.core.mappings`, `zeroth.core.conditions`, etc.

### TTL Management
- **D-07:** Default TTL is configurable via settings (default: 3600 seconds / 1 hour). Per-artifact TTL override is supported at store time.
- **D-08:** TTL refresh on checkpoint/approval: when a run transitions to WAITING_APPROVAL or is checkpointed, the orchestrator calls `refresh_ttl()` on all artifact references in the current run state. This prevents dangling references during long approval waits.
- **D-09:** Bulk cleanup: `async cleanup_run(run_id) -> int` deletes all artifacts with keys matching the run_id prefix. Called when a run is archived.

### Audit Integration
- **D-10:** Audit records (NodeAuditRecord) log ArtifactReferences inline — the reference model (store, key, content_type, size) is small enough to embed. Full payload is NOT logged.
- **D-11:** Audit evidence export adds an optional `resolve_artifacts: bool` parameter. When true, the export resolves each ArtifactReference to retrieve the full payload from the store. When false (default), only references are included.

### Contract Support
- **D-12:** Contracts support an `ArtifactReference` type marker. Contract validation checks the reference structure (all required fields present, valid types) without retrieving the actual payload. This is a lightweight schema check, not a content check.

### Claude's Discretion
- Whether to add a `metadata: dict[str, Any]` field to ArtifactReference for extensibility
- Filesystem store implementation details (directory structure, sidecar format, cleanup strategy)
- Error handling patterns (ArtifactStoreError hierarchy)
- Test fixture strategy for Redis tests (mock vs skip when unavailable)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing Patterns (Protocol-based subsystems)
- `src/zeroth/core/agent_runtime/models.py` — `ThreadStateStore(Protocol)` — reference for Protocol-based store interface
- `src/zeroth/core/secrets/provider.py` — `SecretProvider(Protocol)` — reference for pluggable provider pattern
- `src/zeroth/core/storage/redis.py` — `GovernAIRedisRuntimeStores` — reference for Redis integration patterns

### Settings
- `src/zeroth/core/config/settings.py` — Unified `ZerothSettings` with `RedisSettings`, `DatabaseSettings` sub-models

### Audit System
- `src/zeroth/core/audit/models.py` — `NodeAuditRecord` — where artifact references will be logged
- `src/zeroth/core/audit/sanitizer.py` — Audit sanitization (artifact refs should pass through unsanitized)

### Contract System
- `src/zeroth/core/contracts/registry.py` — `ContractRegistry`, `ContractReference` — reference for type registration

### Orchestrator Integration
- `src/zeroth/core/orchestrator/runtime.py` — Run lifecycle, checkpoint handling, approval transitions
- `src/zeroth/core/runs/repository.py` — Run state persistence, archival

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `RedisSettings` in `config/settings.py` — Redis connection config already exists, ArtifactStore can reuse it
- `GovernAIRedisRuntimeStores` in `storage/redis.py` — Shows how other subsystems use Redis with the existing config
- `ThreadStateStore(Protocol)` — Exact pattern to follow for the ArtifactStore interface
- `SecretProvider(Protocol)` — Another Protocol-based pluggable backend example

### Established Patterns
- **Protocol for interfaces** — All pluggable backends use `Protocol`, not ABC
- **Pydantic ConfigDict(extra="forbid")** — All models use strict validation
- **Settings sub-models** — New settings group added as a nested BaseModel in ZerothSettings
- **Package structure** — Each subsystem is a package with models.py, errors.py, __init__.py

### Integration Points
- `orchestrator/runtime.py` — TTL refresh on checkpoint/approval pause
- `runs/repository.py` — Bulk cleanup on run archival
- `audit/models.py` — Artifact references in NodeAuditRecord
- `contracts/registry.py` — ArtifactReference type support
- `config/settings.py` — ArtifactStoreSettings sub-model
- `service/bootstrap.py` — Initialize artifact store at service startup

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. The implementation is well-constrained by ARTF-01 through ARTF-05 and the existing codebase patterns.

</specifics>

<deferred>
## Deferred Ideas

- S3/GCS artifact store backend — explicitly out of scope per FUTURE-06 in REQUIREMENTS.md
- Artifact content-addressable storage (deduplication) — future optimization
- Artifact streaming for very large payloads — async iteration could be added later

</deferred>

---

*Phase: 34-artifact-store*
*Context gathered: 2026-04-13*
