# Phase 34: Artifact Store - Research

**Researched:** 2026-04-12
**Domain:** Pluggable artifact store for large payload externalization in async Python
**Confidence:** HIGH

## Summary

Phase 34 adds a subsystem that lets workflow nodes externalize large payloads (binary data, generated files, intermediate results) into an external store rather than embedding them in run state. The artifact store provides TTL-based lifecycle management, audit-compatible references, and contract validation support for ArtifactReference types.

The implementation is well-constrained by the CONTEXT.md decisions and maps cleanly onto existing codebase patterns. The `ArtifactStore` Protocol follows the exact same pattern as `ThreadStateStore` and `SecretProvider`. The two concrete implementations (Redis SETEX and filesystem with sidecar metadata) use libraries already in the dependency tree. Redis is available via the `dispatch` extras (`redis>=5.0.0`, installed as 5.3.1), and filesystem operations use stdlib `pathlib` + `asyncio` (no new dependency needed). All integration points (audit, contracts, orchestrator, settings) have clear precedents in the codebase.

**Primary recommendation:** Build the `zeroth.core.artifacts` package following the exact structure of `zeroth.core.mappings` (models.py, errors.py, store.py, __init__.py). Use `redis.asyncio` for the Redis backend with `setex` for TTL. Use synchronous filesystem I/O wrapped in `asyncio.to_thread` for the filesystem backend. Integrate into the orchestrator's existing `write_checkpoint` call sites for TTL refresh.

<user_constraints>

## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** ArtifactStore uses a Protocol (not ABC) -- consistent with ThreadStateStore, SecretProvider, and ProviderAdapter patterns in the codebase. Methods: `async store(key, data, content_type, ttl) -> ArtifactReference`, `async retrieve(key) -> bytes`, `async delete(key) -> bool`, `async refresh_ttl(key, ttl) -> bool`, `async exists(key) -> bool`.
- **D-02:** Two implementations: `RedisArtifactStore` (uses SETEX for TTL) and `FilesystemArtifactStore` (stores files in a configurable directory with a metadata sidecar for TTL tracking).
- **D-03:** Key generation uses `{run_id}/{node_id}/{uuid}` pattern -- namespaced by run for bulk cleanup on archive.
- **D-04:** Store selection is via `ArtifactStoreSettings` added to the unified `ZerothSettings` -- follows the existing RedisSettings, DatabaseSettings, MemorySettings pattern.
- **D-05:** `ArtifactReference` is a Pydantic model with fields: store (str), key (str), content_type (str), size (int), created_at (datetime), ttl_seconds (int | None). JSON-serializable, embeddable in node output payloads.
- **D-06:** ArtifactReference is placed in a new `zeroth.core.artifacts` package -- follows the pattern of zeroth.core.mappings, zeroth.core.conditions, etc.
- **D-07:** Default TTL is configurable via settings (default: 3600 seconds / 1 hour). Per-artifact TTL override is supported at store time.
- **D-08:** TTL refresh on checkpoint/approval: when a run transitions to WAITING_APPROVAL or is checkpointed, the orchestrator calls refresh_ttl() on all artifact references in the current run state.
- **D-09:** Bulk cleanup: `async cleanup_run(run_id) -> int` deletes all artifacts with keys matching the run_id prefix. Called when a run is archived.
- **D-10:** Audit records (NodeAuditRecord) log ArtifactReferences inline -- the reference model is small enough to embed. Full payload is NOT logged.
- **D-11:** Audit evidence export adds an optional `resolve_artifacts: bool` parameter. When true, resolves ArtifactReferences to full payloads. When false (default), only references are included.
- **D-12:** Contracts support an ArtifactReference type marker. Contract validation checks reference structure (all required fields present, valid types) without retrieving the actual payload.

### Claude's Discretion
- Whether to add a `metadata: dict[str, Any]` field to ArtifactReference for extensibility
- Filesystem store implementation details (directory structure, sidecar format, cleanup strategy)
- Error handling patterns (ArtifactStoreError hierarchy)
- Test fixture strategy for Redis tests (mock vs skip when unavailable)

### Deferred Ideas (OUT OF SCOPE)
- S3/GCS artifact store backend -- explicitly out of scope per FUTURE-06 in REQUIREMENTS.md
- Artifact content-addressable storage (deduplication) -- future optimization
- Artifact streaming for very large payloads -- async iteration could be added later

</user_constraints>

<phase_requirements>

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ARTF-01 | Pluggable artifact store interface with Redis and filesystem implementations, configurable via settings | Protocol pattern from ThreadStateStore; RedisSettings pattern; redis.asyncio SETEX verified available; ArtifactStoreSettings sub-model in ZerothSettings |
| ARTF-02 | Nodes can emit ArtifactReference as part of output; reference persisted in run history, payload in artifact store | ArtifactReference Pydantic model; RunHistoryEntry output_snapshot stores dict[str, Any] which can contain serialized ArtifactReference |
| ARTF-03 | Configurable TTL; artifacts cleanable on run archive; TTL refresh on checkpoint/approval | Redis SETEX and expire() for TTL; orchestrator write_checkpoint call sites identified for TTL refresh hooks; scan_iter for prefix-based bulk cleanup |
| ARTF-04 | Audit records log references not payloads; evidence export can optionally resolve references | NodeAuditRecord.output_snapshot already stores dict; evidence.py build_summary is the integration point for export |
| ARTF-05 | Contracts support ArtifactReference type with structural validation | Pydantic model_validate on ArtifactReference for structural checks; contract registry can register ArtifactReference as a contract type |

</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Build/test commands:** `uv sync`, `uv run pytest -v`, `uv run ruff check src/`, `uv run ruff format src/`
- **Project layout:** `src/zeroth/` main package, `tests/` pytest tests
- **Progress logging:** Every implementation session MUST use the `progress-logger` skill
- **Pydantic validation:** All models use `ConfigDict(extra="forbid")` [VERIFIED: codebase grep]
- **Protocol over ABC:** All pluggable backends use `Protocol`, not ABC [VERIFIED: codebase grep]
- **Package structure:** Each subsystem is a package with models.py, errors.py, __init__.py [VERIFIED: codebase grep]

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| redis (redis.asyncio) | >=5.0.0 (installed: 5.3.1) | Redis artifact store backend | Already in dependency tree via `dispatch` extras; provides async SETEX, EXPIRE, SCAN_ITER [VERIFIED: pyproject.toml + runtime check] |
| pydantic | >=2.10 (installed: 2.12.5) | ArtifactReference model, ArtifactStoreSettings | Already core dependency; model_validate for contract validation [VERIFIED: pyproject.toml + runtime check] |
| pydantic-settings | >=2.13 | ArtifactStoreSettings in ZerothSettings | Already core dependency; same pattern as RedisSettings, DatabaseSettings [VERIFIED: pyproject.toml] |
| Python stdlib (pathlib, asyncio, json) | 3.12+ | Filesystem store implementation | No new dependency; asyncio.to_thread wraps blocking I/O [VERIFIED: pyproject.toml requires-python] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| uuid (stdlib) | 3.12 | Key generation (uuid4 hex) | Every artifact store operation for key uniqueness [VERIFIED: existing pattern in runs/models.py] |
| datetime (stdlib) | 3.12 | created_at timestamps | ArtifactReference timestamps, sidecar metadata [VERIFIED: existing pattern in audit/models.py] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| asyncio.to_thread for FS | aiofiles | Would add a new dependency; stdlib asyncio.to_thread is sufficient for file I/O and avoids dependency bloat [ASSUMED] |
| Redis SETEX | Redis SET with EX param | Both work; SETEX is more explicit and matches the CONTEXT.md language. `set(name, value, ex=ttl)` is equivalent but SETEX is the canonical TTL-set command [VERIFIED: redis.asyncio signatures] |
| JSON sidecar for FS metadata | SQLite per-directory | JSON sidecar is simpler, atomic per-file, no shared-state contention; SQLite would add complexity for local dev [ASSUMED] |

**Installation:**
```bash
# No new packages needed. redis is already available via dispatch extras.
uv sync
```

**Version verification:**
- redis: 5.3.1 installed, >=5.0.0 required [VERIFIED: `uv run python -c "import redis; print(redis.__version__)"` => 5.3.1]
- pydantic: 2.12.5 installed [VERIFIED: runtime check]
- Python: >=3.12 required [VERIFIED: pyproject.toml]

## Architecture Patterns

### Recommended Project Structure
```
src/zeroth/core/artifacts/
    __init__.py          # Public API exports
    models.py            # ArtifactReference, ArtifactStoreSettings
    errors.py            # ArtifactStoreError hierarchy
    store.py             # ArtifactStore Protocol + implementations
    cleanup.py           # Bulk cleanup and TTL refresh helpers
```

### Pattern 1: Protocol-Based Store Interface
**What:** Define `ArtifactStore` as a `typing.Protocol` with five async methods.
**When to use:** Always -- this is the locked decision from CONTEXT.md.
**Example:**
```python
# Source: ThreadStateStore pattern from src/zeroth/core/agent_runtime/models.py
from typing import Protocol

class ArtifactStore(Protocol):
    async def store(
        self, key: str, data: bytes, content_type: str, ttl: int | None = None,
    ) -> ArtifactReference: ...

    async def retrieve(self, key: str) -> bytes: ...

    async def delete(self, key: str) -> bool: ...

    async def refresh_ttl(self, key: str, ttl: int) -> bool: ...

    async def exists(self, key: str) -> bool: ...
```

### Pattern 2: Redis SETEX for TTL-Managed Storage
**What:** Use `redis.asyncio.Redis.setex(key, ttl_seconds, data)` for atomic set-with-TTL.
**When to use:** RedisArtifactStore implementation.
**Example:**
```python
# Source: redis.asyncio.Redis.setex signature verified via runtime inspection
import redis.asyncio as aioredis

class RedisArtifactStore:
    def __init__(self, redis_url: str, prefix: str = "zeroth:artifact") -> None:
        self._client = aioredis.Redis.from_url(redis_url)
        self._prefix = prefix

    async def store(self, key: str, data: bytes, content_type: str, ttl: int | None = None) -> ArtifactReference:
        full_key = f"{self._prefix}:{key}"
        if ttl is not None:
            await self._client.setex(full_key, ttl, data)
        else:
            await self._client.set(full_key, data)
        # Store metadata in a companion key
        meta_key = f"{full_key}:meta"
        meta = {"content_type": content_type, "size": len(data), ...}
        if ttl is not None:
            await self._client.setex(meta_key, ttl, json.dumps(meta))
        else:
            await self._client.set(meta_key, json.dumps(meta))
        return ArtifactReference(store="redis", key=key, ...)

    async def refresh_ttl(self, key: str, ttl: int) -> bool:
        full_key = f"{self._prefix}:{key}"
        result = await self._client.expire(full_key, ttl)
        await self._client.expire(f"{full_key}:meta", ttl)
        return bool(result)
```

### Pattern 3: Filesystem Store with JSON Sidecar
**What:** Store artifact files in a directory tree with `.meta.json` sidecar files for TTL and content_type tracking.
**When to use:** FilesystemArtifactStore for local development without Redis.
**Example:**
```python
# Source: stdlib pathlib + asyncio.to_thread pattern
from pathlib import Path
import asyncio
import json

class FilesystemArtifactStore:
    def __init__(self, base_dir: str | Path) -> None:
        self._base = Path(base_dir)

    async def store(self, key: str, data: bytes, content_type: str, ttl: int | None = None) -> ArtifactReference:
        file_path = self._base / key
        meta_path = self._base / f"{key}.meta.json"
        await asyncio.to_thread(self._write_sync, file_path, data, meta_path, content_type, ttl)
        return ArtifactReference(store="filesystem", key=key, ...)

    def _write_sync(self, file_path: Path, data: bytes, meta_path: Path, content_type: str, ttl: int | None) -> None:
        file_path.parent.mkdir(parents=True, exist_ok=True)
        file_path.write_bytes(data)
        meta = {
            "content_type": content_type,
            "size": len(data),
            "created_at": datetime.now(UTC).isoformat(),
            "ttl_seconds": ttl,
            "expires_at": (datetime.now(UTC) + timedelta(seconds=ttl)).isoformat() if ttl else None,
        }
        meta_path.write_text(json.dumps(meta))
```

### Pattern 4: Settings Sub-Model Integration
**What:** Add `ArtifactStoreSettings` as a nested BaseModel in `ZerothSettings`, following the exact pattern of `RedisSettings`, `MemorySettings`, etc.
**When to use:** Configuration of the artifact store backend.
**Example:**
```python
# Source: src/zeroth/core/config/settings.py pattern
class ArtifactStoreSettings(BaseModel):
    """Artifact store configuration."""
    backend: str = "filesystem"  # "filesystem" or "redis"
    default_ttl_seconds: int = 3600
    filesystem_base_dir: str = ".zeroth/artifacts"
    redis_key_prefix: str = "zeroth:artifact"

# In ZerothSettings:
artifact_store: ArtifactStoreSettings = Field(default_factory=ArtifactStoreSettings)
```

### Pattern 5: Orchestrator TTL Refresh Integration
**What:** At every `write_checkpoint` call site in the orchestrator, scan the run's execution history for ArtifactReferences and refresh their TTLs.
**When to use:** Preventing dangling references during long approval waits.
**Integration points identified in orchestrator/runtime.py:**
1. After `_drive()` completes a node step (line ~245)
2. When entering WAITING_APPROVAL status (line ~228)
3. When run completes (line ~160)

### Anti-Patterns to Avoid
- **Storing full payloads in audit records:** Audit records should only embed ArtifactReference (store, key, content_type, size), never the bytes. The evidence export's `resolve_artifacts` flag handles on-demand resolution.
- **Blocking I/O in async context:** Filesystem operations MUST use `asyncio.to_thread` to avoid blocking the event loop. Never call `Path.write_bytes()` directly in an async method.
- **Separate TTL for data and metadata keys in Redis:** Always refresh/set TTL on both the data key and its `:meta` companion key together. A mismatch causes orphaned metadata or missing content_type.
- **Using `KEYS` command for prefix scans:** Use `scan_iter(match=pattern)` instead of `keys(pattern)`. `KEYS` blocks Redis for large keyspaces. [VERIFIED: redis.asyncio has scan_iter]

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Redis connection management | Custom connection pool | `redis.asyncio.Redis.from_url()` | Handles connection pooling, reconnection, and URL parsing [VERIFIED: existing pattern in storage/redis.py] |
| JSON serialization for metadata | Custom serializer | `pydantic.BaseModel.model_dump(mode="json")` + `json.dumps` | Handles datetime serialization, type coercion [VERIFIED: existing pattern throughout codebase] |
| TTL expiration tracking for FS | Custom timer/scheduler | JSON sidecar with `expires_at` field + periodic cleanup sweep | Avoids in-process timers that don't survive restarts |
| UUID generation | Custom ID scheme | `uuid4().hex` | Established pattern in `runs/models.py` [VERIFIED: codebase pattern] |
| Async file I/O | Custom thread pool executor | `asyncio.to_thread()` | Built into Python 3.12+, simpler than managing executors [VERIFIED: requires-python >=3.12] |

**Key insight:** The artifact store is a thin wrapper over existing Redis/filesystem primitives. The complexity is in the lifecycle management (TTL refresh, bulk cleanup, audit integration) not in the storage itself.

## Common Pitfalls

### Pitfall 1: Redis Key Atomicity for Data + Metadata
**What goes wrong:** Data key and metadata key get out of sync -- one expires while the other persists, or one write succeeds while the other fails.
**Why it happens:** SETEX is not transactional across two keys. Network failure between two SETEX calls leaves inconsistent state.
**How to avoid:** Use Redis pipeline to batch the two SETEX operations atomically. For refresh_ttl, pipeline the two EXPIRE calls.
**Warning signs:** `retrieve()` returns data but metadata lookup fails, or metadata exists but data is gone.

### Pitfall 2: Filesystem TTL Cleanup Race Conditions
**What goes wrong:** A cleanup sweep deletes an artifact file while another coroutine is mid-read.
**Why it happens:** File deletion and file read are not coordinated.
**How to avoid:** Catch `FileNotFoundError` in `retrieve()` and treat it as "artifact not found" rather than letting it propagate. Use `try/except` consistently.
**Warning signs:** Spurious errors during concurrent cleanup + retrieval.

### Pitfall 3: Scan-Based Bulk Cleanup Performance
**What goes wrong:** `cleanup_run(run_id)` with `scan_iter` is slow for Redis instances with millions of keys.
**Why it happens:** SCAN iterates the entire keyspace, not just the matching prefix.
**How to avoid:** The `{run_id}/{node_id}/{uuid}` key pattern is designed for prefix matching. Use `scan_iter(match=f"{prefix}:{run_id}/*")` with a reasonable `count` parameter (e.g., 100). For production scale, consider Redis modules or dedicated namespacing.
**Warning signs:** cleanup_run taking seconds instead of milliseconds.

### Pitfall 4: Circular Import with Orchestrator Integration
**What goes wrong:** Importing ArtifactStore in orchestrator/runtime.py causes circular imports because the artifacts package imports from core modules that runtime.py also imports.
**Why it happens:** The codebase has TYPE_CHECKING guards (see orchestrator/runtime.py line 20-23) specifically to avoid this.
**How to avoid:** Use `TYPE_CHECKING` guards for type-only imports. Pass the artifact store as a constructor parameter to RuntimeOrchestrator (like approval_service, webhook_service, etc.).
**Warning signs:** ImportError at module load time.

### Pitfall 5: ArtifactReference Serialization in Run State
**What goes wrong:** ArtifactReference objects in run execution_history don't survive JSON round-tripping because they're serialized as plain dicts and not deserialized back.
**Why it happens:** RunHistoryEntry.output_snapshot is `dict[str, Any]`, not typed to ArtifactReference.
**How to avoid:** ArtifactReference is a Pydantic model with `model_dump(mode="json")` / `model_validate()`. The TTL refresh helper must be able to find and reconstruct ArtifactReferences from the serialized dicts in run history. Use a discriminator field or store a `_type: "artifact_reference"` marker.
**Warning signs:** TTL refresh silently skips artifacts because it can't find them in the serialized run state.

### Pitfall 6: Missing TTL Refresh on Side-Effect Approval
**What goes wrong:** Artifacts expire while a run is WAITING_APPROVAL for side-effect policy (not just HumanApprovalNode).
**Why it happens:** The orchestrator has two approval paths: (1) HumanApprovalNode gate and (2) side-effect policy approval via `_request_side_effect_approval`. Both call `write_checkpoint` but TTL refresh must hook into both.
**How to avoid:** Hook TTL refresh into every `write_checkpoint` call site, not just the HumanApprovalNode path. There are 7+ call sites in the _drive() loop.
**Warning signs:** Artifacts expire during side-effect approval waits but not during human approval waits.

## Code Examples

### Key Generation Pattern
```python
# Source: D-03 from CONTEXT.md, uuid4 pattern from runs/models.py
from uuid import uuid4

def generate_artifact_key(run_id: str, node_id: str) -> str:
    """Generate a namespaced artifact key for prefix-based bulk cleanup."""
    return f"{run_id}/{node_id}/{uuid4().hex}"
```

### ArtifactReference Model
```python
# Source: D-05 from CONTEXT.md, ConfigDict pattern from codebase
from datetime import UTC, datetime
from pydantic import BaseModel, ConfigDict, Field

class ArtifactReference(BaseModel):
    """Lightweight pointer to an externalized artifact."""
    model_config = ConfigDict(extra="forbid")

    store: str           # Backend identifier ("redis", "filesystem")
    key: str             # Storage key ({run_id}/{node_id}/{uuid})
    content_type: str    # MIME type
    size: int            # Payload size in bytes
    created_at: datetime = Field(default_factory=lambda: datetime.now(UTC))
    ttl_seconds: int | None = None
```

### Redis Pipeline for Atomic Store
```python
# Source: redis.asyncio pipeline pattern [VERIFIED: redis.asyncio availability]
async def store(self, key: str, data: bytes, content_type: str, ttl: int | None = None) -> ArtifactReference:
    full_key = f"{self._prefix}:{key}"
    meta_key = f"{full_key}:meta"
    meta = json.dumps({
        "content_type": content_type,
        "size": len(data),
        "created_at": datetime.now(UTC).isoformat(),
    })
    async with self._client.pipeline(transaction=True) as pipe:
        if ttl is not None:
            pipe.setex(full_key, ttl, data)
            pipe.setex(meta_key, ttl, meta)
        else:
            pipe.set(full_key, data)
            pipe.set(meta_key, meta)
        await pipe.execute()
    return ArtifactReference(
        store="redis", key=key, content_type=content_type,
        size=len(data), ttl_seconds=ttl,
    )
```

### Bulk Cleanup via SCAN
```python
# Source: redis.asyncio.scan_iter [VERIFIED: method signature]
async def cleanup_run(self, run_id: str) -> int:
    """Delete all artifacts for a run. Returns count of deleted keys."""
    pattern = f"{self._prefix}:{run_id}/*"
    deleted = 0
    async for key in self._client.scan_iter(match=pattern, count=100):
        await self._client.delete(key)
        deleted += 1
    return deleted
```

### ArtifactReference Extraction from Run State
```python
# Source: Pattern needed for TTL refresh (Pitfall 5)
def extract_artifact_refs(run: Run) -> list[ArtifactReference]:
    """Find all ArtifactReferences in a run's execution history."""
    refs: list[ArtifactReference] = []
    for entry in run.execution_history:
        _scan_dict_for_refs(entry.output_snapshot, refs)
    # Also check metadata and final_output
    if isinstance(run.final_output, dict):
        _scan_dict_for_refs(run.final_output, refs)
    return refs

def _scan_dict_for_refs(data: dict[str, Any], refs: list[ArtifactReference]) -> None:
    """Recursively scan a dict for ArtifactReference-shaped objects."""
    for value in data.values():
        if isinstance(value, dict) and _looks_like_artifact_ref(value):
            refs.append(ArtifactReference.model_validate(value))
        elif isinstance(value, dict):
            _scan_dict_for_refs(value, refs)
        elif isinstance(value, list):
            for item in value:
                if isinstance(item, dict):
                    _scan_dict_for_refs(item, refs)

def _looks_like_artifact_ref(d: dict[str, Any]) -> bool:
    """Check if a dict has the ArtifactReference shape."""
    return {"store", "key", "content_type", "size"}.issubset(d.keys())
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| redis-py sync API | redis.asyncio (unified in redis-py >=5.0) | redis-py 5.0 (2023) | No separate aioredis package needed; `import redis.asyncio` [VERIFIED: installed redis 5.3.1] |
| aioredis standalone package | Merged into redis-py | redis-py 4.2+ | aioredis is deprecated; use `redis.asyncio` [VERIFIED: aioredis not in pyproject.toml] |
| Separate thread pool for file I/O | asyncio.to_thread() | Python 3.9+ | Simpler than managing ThreadPoolExecutor; built-in [VERIFIED: requires-python >=3.12] |

**Deprecated/outdated:**
- **aioredis:** Merged into redis-py; do not use as separate package
- **redis-py sync in async context:** Always use `redis.asyncio` in async code

## Discretion Recommendations

### Metadata Field on ArtifactReference
**Recommendation:** Add `metadata: dict[str, Any] = Field(default_factory=dict)`. This follows the same pattern as `AgentConfig.metadata`, `ContractVersion.metadata`, `ToolContractBinding.metadata` throughout the codebase. Cost is minimal (one dict field), and it enables extensibility for downstream consumers without schema changes. [ASSUMED -- decision is Claude's discretion per CONTEXT.md]

### Filesystem Store Implementation Details
**Recommendation:**
- Directory structure: `{base_dir}/{run_id}/{node_id}/{uuid}` matching the key pattern. Each file has a companion `.meta.json` sidecar.
- Sidecar format: JSON with `content_type`, `size`, `created_at`, `ttl_seconds`, `expires_at` fields.
- Cleanup strategy: Filesystem cleanup_run deletes the entire `{base_dir}/{run_id}/` directory tree. TTL cleanup is a periodic sweep that reads `.meta.json` files and deletes expired entries. The sweep can be triggered lazily on `retrieve()` or by an explicit `cleanup_expired()` method.
[ASSUMED -- no strong precedent in codebase for filesystem cleanup patterns]

### Error Handling Hierarchy
**Recommendation:**
```
ArtifactStoreError (base)
    ArtifactNotFoundError   -- retrieve/delete on missing key
    ArtifactStorageError    -- backend write failure
    ArtifactTTLError        -- TTL refresh on expired/missing artifact
```
This follows the `ContractRegistryError` -> `ContractNotFoundError` / `ContractVersionExistsError` / `ContractTypeResolutionError` pattern. [VERIFIED: contracts/errors.py pattern]

### Test Fixture Strategy for Redis
**Recommendation:** Mock Redis for unit tests; skip when unavailable for integration. The codebase already uses `AsyncMock` for Redis in `tests/test_health_probes.py` (mocking `redis_from_url`). Follow the same pattern: mock `redis.asyncio.Redis` for unit tests of `RedisArtifactStore`. For integration tests, use `pytest.mark.skipif` when Redis is not available (like the existing `requires_docker` fixture in conftest.py). [VERIFIED: test_health_probes.py mock pattern]

## Assumptions Log

> List all claims tagged [ASSUMED] in this research.

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | asyncio.to_thread is sufficient for filesystem I/O (no aiofiles needed) | Alternatives Considered | Low -- aiofiles could be added later if performance is insufficient; asyncio.to_thread works for the expected payload sizes |
| A2 | JSON sidecar is simpler than SQLite for filesystem metadata | Alternatives Considered | Low -- JSON sidecar has no shared-state contention; SQLite would only be better for high-throughput FS stores which is not the expected use case |
| A3 | Metadata dict field on ArtifactReference is worthwhile | Discretion Recommendations | Very low -- one optional field, follows codebase conventions |
| A4 | Filesystem cleanup via directory tree deletion is appropriate | Discretion Recommendations | Low -- alternative is file-by-file deletion which is slower and more error-prone |

## Open Questions

1. **How to detect ArtifactReferences in serialized run state for TTL refresh?**
   - What we know: RunHistoryEntry.output_snapshot is `dict[str, Any]` -- ArtifactReferences are serialized as plain dicts when stored in the database.
   - What's unclear: Should we use a discriminator field (e.g., `_type: "artifact_reference"`) or rely on duck-typing (check for required fields)?
   - Recommendation: Use duck-typing with `_looks_like_artifact_ref()` that checks for the four required fields (`store`, `key`, `content_type`, `size`). This avoids adding a discriminator field to the model and is robust enough given the unique shape of ArtifactReference.

2. **Should ArtifactStore be added to RuntimeOrchestrator constructor or resolved from settings at runtime?**
   - What we know: RuntimeOrchestrator takes `approval_service`, `webhook_service`, etc. as constructor parameters (dataclass fields). Bootstrap wires them at startup.
   - What's unclear: Whether the artifact store should be optional (like `approval_service: ApprovalService | None = None`) or required.
   - Recommendation: Make it optional (`artifact_store: ArtifactStore | None = None`) since not all deployments need artifact externalization. When None, TTL refresh is a no-op.

3. **Filesystem TTL cleanup: lazy vs. periodic?**
   - What we know: Redis handles TTL natively. Filesystem needs manual cleanup.
   - What's unclear: Whether to check expiration on every `retrieve()` call or run a periodic background task.
   - Recommendation: Lazy cleanup on `retrieve()` (return "not found" for expired artifacts) plus a `cleanup_expired()` method that can be called from a background task or CLI command. Do NOT implement the background task in this phase -- it's unnecessary complexity for a local dev tool.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| redis (Python package) | RedisArtifactStore | Yes (via dispatch extras) | 5.3.1 | FilesystemArtifactStore (default backend) |
| Redis server | RedisArtifactStore at runtime | Not checked (runtime dep) | -- | FilesystemArtifactStore (default backend) |
| Python >=3.12 | asyncio.to_thread | Yes | 3.12+ | -- |

**Missing dependencies with no fallback:**
- None -- filesystem is the default backend, Redis is optional.

**Missing dependencies with fallback:**
- Redis server at runtime: falls back to filesystem backend automatically via settings.

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | N/A -- artifact store is internal subsystem |
| V3 Session Management | No | N/A |
| V4 Access Control | Partially | Artifact keys are namespaced by run_id; access control is inherited from run-level authorization |
| V5 Input Validation | Yes | Pydantic model validation on ArtifactReference (ConfigDict extra="forbid"); content_type is string-validated; key format is enforced by generation function |
| V6 Cryptography | No | Artifact data is not encrypted at rest in Redis or filesystem (same as run state -- encryption is handled at the database layer per DatabaseSettings.encryption_key) |

### Known Threat Patterns for Artifact Store

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Path traversal in filesystem store key | Tampering | Validate that resolved file path is within base_dir; reject keys containing `..` |
| Oversized payload denial-of-service | Denial of Service | Add max_artifact_size_bytes setting; reject payloads exceeding limit before writing |
| Artifact reference spoofing | Spoofing | ArtifactReference is generated server-side; never accept client-supplied artifact keys |
| TTL bypass by manipulating sidecar | Tampering | Sidecar is server-managed; if filesystem is writable by attacker, security boundary is already breached |

## Sources

### Primary (HIGH confidence)
- `src/zeroth/core/agent_runtime/models.py` -- ThreadStateStore Protocol pattern, verified by reading source
- `src/zeroth/core/secrets/provider.py` -- SecretProvider Protocol pattern, verified by reading source
- `src/zeroth/core/storage/redis.py` -- RedisConfig, GovernAIRedisRuntimeStores pattern, verified by reading source
- `src/zeroth/core/config/settings.py` -- ZerothSettings, RedisSettings, nested BaseModel pattern, verified by reading source
- `src/zeroth/core/audit/models.py` -- NodeAuditRecord structure, verified by reading source
- `src/zeroth/core/audit/evidence.py` -- Evidence export functions, verified by reading source
- `src/zeroth/core/contracts/registry.py` -- ContractRegistry, ContractReference, verified by reading source
- `src/zeroth/core/orchestrator/runtime.py` -- write_checkpoint call sites, approval flow, verified by grep
- `src/zeroth/core/runs/models.py` -- Run, RunHistoryEntry models, verified by reading source
- `src/zeroth/core/runs/repository.py` -- RunRepository.write_checkpoint, verified by grep
- `src/zeroth/core/mappings/` -- Package structure pattern (models.py, errors.py, etc.), verified by reading source
- `pyproject.toml` -- Dependencies and extras, verified by reading source
- Redis method signatures -- verified via runtime inspection of redis.asyncio.Redis

### Secondary (MEDIUM confidence)
- redis-py documentation for SETEX, SCAN patterns -- based on verified method signatures

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries verified installed, no new dependencies
- Architecture: HIGH -- all patterns verified against existing codebase, locked decisions are specific
- Pitfalls: HIGH -- identified from direct code reading of orchestrator integration points
- Security: MEDIUM -- threat patterns are standard for blob stores; no specific vulnerability research done

**Research date:** 2026-04-12
**Valid until:** 2026-05-12 (stable domain, no fast-moving dependencies)
