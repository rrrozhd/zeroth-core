# Coding Conventions

**Analysis Date:** 2026-04-05

## Naming Patterns

**Files:**
- Use `snake_case.py` for all Python modules: `run_api.py`, `thread_store.py`, `dead_letter.py`
- Error modules are named `errors.py` within their package: `src/zeroth/graph/errors.py`, `src/zeroth/contracts/errors.py`
- Model modules are named `models.py` within their package: `src/zeroth/graph/models.py`, `src/zeroth/runs/models.py`
- API route modules end with `_api.py`: `src/zeroth/service/run_api.py`, `src/zeroth/service/audit_api.py`, `src/zeroth/service/contracts_api.py`
- Repository modules are named `repository.py`: `src/zeroth/graph/repository.py`, `src/zeroth/runs/repository.py`

**Functions:**
- Use `snake_case` for all functions and methods
- Private methods prefixed with single underscore: `_validate_references`, `_utc_now`, `_serialize_run`
- Module-level private helpers: `_utc_now()`, `_new_id()`, `_label_str()`, `_semantic_graph_dump()`
- Test functions use `test_` prefix with descriptive snake_case: `test_run_creation_accepts_input_and_supplied_thread_id`

**Variables:**
- Use `snake_case` for all variables
- Constants are `UPPER_SNAKE_CASE`: `GRAPH_SCHEMA_VERSION`, `GRAPH_SCHEMA_SCOPE`, `TEST_API_KEYS`

**Classes:**
- Use `PascalCase`: `GraphRepository`, `AgentNode`, `ExecutableUnitRunner`
- Error classes end with `Error`: `GraphLifecycleError`, `AgentTimeoutError`, `ManifestValidationError`
- Pydantic response/request models end accordingly: `RunStatusResponse`, `RunInvocationRequest`, `HealthResponse`
- Enums use `PascalCase` and inherit from `StrEnum`: `GraphStatus`, `RunStatus`, `Capability`, `PolicyDecision`
- Protocol classes end with `Like`: `ServiceBootstrapLike`, `RunApiBootstrapLike`

**Enum values:**
- Use `UPPER_SNAKE_CASE` for enum member names
- Use `lower_snake_case` strings for values: `DRAFT = "draft"`, `NETWORK_READ = "network_read"`

## Code Style

**Formatting:**
- Formatter: `ruff format`
- Line length: 100 characters
- Target version: Python 3.12
- Config: `pyproject.toml` `[tool.ruff]`

**Linting:**
- Linter: `ruff check`
- Rule sets: `E` (pycodestyle), `F` (pyflakes), `I` (isort), `N` (pep8-naming), `UP` (pyupgrade), `B` (flake8-bugbear), `SIM` (flake8-simplify)
- Config: `pyproject.toml` `[tool.ruff.lint]`

**Run commands:**
```bash
uv run ruff check src/     # lint
uv run ruff format src/    # format
```

## Import Organization

**Order (enforced by ruff `I` rule):**
1. `from __future__ import annotations` -- always the first line in every module
2. Standard library imports: `pathlib`, `sqlite3`, `datetime`, `enum`, `uuid`, `contextlib`, `asyncio`, `threading`
3. Third-party imports: `fastapi`, `pydantic`, `pytest`, `governai`, `redis`, `httpx`
4. Local package imports: `zeroth.*`

**Style:**
- Prefer `from X import Y` over bare `import X`
- Group imports from the same package on separate lines
- Use full dotted paths: `from zeroth.graph.models import Graph` (no relative imports)
- Package `__init__.py` files re-export key symbols via `__all__`

**Example from** `src/zeroth/orchestrator/runtime.py`:
```python
from __future__ import annotations

import inspect
from collections.abc import Mapping
from dataclasses import dataclass
from time import perf_counter
from typing import Any

from zeroth.agent_runtime import AgentRunner, RepositoryThreadResolver
from zeroth.approvals import ApprovalDecision, ApprovalRecord, ApprovalService
from zeroth.audit import AuditRepository, NodeAuditRecord
```

**Path Aliases:**
- None. All imports use full dotted paths.

## Error Handling

**Hierarchy pattern:**
- Each domain package defines errors at `src/zeroth/<package>/errors.py`
- A base error per domain; specific errors inherit from it
- Base errors inherit from built-ins: `Exception` for domain errors, `ValueError` for validation errors, `RuntimeError` for operational errors

**Example from** `src/zeroth/agent_runtime/errors.py`:
```python
class AgentRuntimeError(Exception):
    """Base error for all agent runtime failures."""

class AgentInputValidationError(AgentRuntimeError): ...
class AgentProviderError(AgentRuntimeError): ...
class AgentTimeoutError(AgentProviderError): ...   # sub-hierarchy
class AgentOutputValidationError(AgentRuntimeError): ...
class AgentRetryExhaustedError(AgentRuntimeError):
    def __init__(self, *, attempts: int, last_error: Exception): ...
```

**Example from** `src/zeroth/contracts/errors.py`:
```python
class ContractRegistryError(Exception):
class ContractNotFoundError(ContractRegistryError): ...
class ContractVersionExistsError(ContractRegistryError): ...
class ContractTypeResolutionError(ContractRegistryError): ...
```

**Error message pattern -- mandatory:**
- Build the message on a separate line, then raise. Never inline the f-string in `raise`:
```python
msg = f"graph version {graph.graph_id}@{graph.version} is immutable"
raise GraphLifecycleError(msg)
```

**HTTP error handling (FastAPI API boundary):**
- Use `raise HTTPException(status_code=..., detail=...)` in API route handlers
- Chain with `from exc`: `raise HTTPException(...) from exc`
- Map domain errors to HTTP status codes at the API layer only -- domain code never imports `HTTPException`

## Model/Schema Patterns (Pydantic)

**Base configuration -- apply to every new model:**
```python
from pydantic import BaseModel, ConfigDict, Field

class MyModel(BaseModel):
    model_config = ConfigDict(extra="forbid")
```
- Always set `extra="forbid"` to reject unknown fields
- Add `validate_assignment=True` on mutable state models like `Run`, `Thread`

**Field defaults:**
- `Field(default_factory=list)` for mutable list defaults (never bare `[]`)
- `Field(default_factory=dict)` for dict defaults
- `Field(default_factory=_utc_now)` for timestamp defaults
- `Field(default_factory=_new_id)` for generated IDs (`uuid4().hex`)
- `Field(ge=1)` for positive integer constraints

**Discriminated unions:**
```python
Node = Annotated[
    AgentNode | ExecutableUnitNode | HumanApprovalNode,
    Field(discriminator="node_type"),
]
```
Each variant declares its discriminator as a `Literal` field:
```python
class AgentNode(NodeBase):
    node_type: Literal["agent"] = "agent"
```

**Model validators:**
- Use `@model_validator(mode="after")` for cross-field validation
- Name with `_` prefix: `_validate_references`, `_fill_governai_defaults`, `_require_key_source`

**Immutable updates:**
- Use `model.model_copy(update={...})` to create modified copies
- Never mutate model fields directly unless `validate_assignment=True` is set

**Serialization:**
- `model.model_dump(mode="json")` for JSON-safe output
- `model.model_dump(exclude={...})` to omit fields for semantic comparison

## Repository/Storage Patterns

**Repository pattern:**
- Each domain has a repository at `src/zeroth/<domain>/repository.py`
- Constructor takes `SQLiteDatabase` and calls `apply_migrations` immediately
- Standard CRUD methods: `save()`, `get()`, `list()`, `create()`
- Internal `_require()` helper: fetch or raise `KeyError`

**Transaction pattern:**
```python
with self._database.transaction() as connection:
    row = connection.execute("SELECT ...", (params,)).fetchone()
```
- Always use `self._database.transaction()` context manager
- Connections use `Row` factory: access columns by name `row["payload"]`

**Migration pattern:**
- Define `Migration(version=N, name="...", sql="...")` in a `storage.py` module per domain
- Versions are contiguous starting at 1
- Schema scopes are string constants: `"graphs"`, `"runs"`, `"contracts"`
- Migrations applied in `__init__` of each repository

**Encryption:**
- `EncryptedField` in `src/zeroth/storage/sqlite.py` provides Fernet-based field encryption for sensitive columns

## API Endpoint Patterns

**Route registration:**
- Dedicated `register_*_routes(app: FastAPI)` functions in `src/zeroth/service/*_api.py`
- Called from `src/zeroth/service/app.py` `create_app()` function
- Routes are closures defined inside the registration function

**Bootstrap access:**
- API handlers access dependencies via `request.app.state.bootstrap`
- Type the bootstrap with `Protocol` classes: `RunApiBootstrapLike`, `ServiceBootstrapLike`
- Helper: `_bootstrap(request: Request) -> BootstrapLike`

**Request/Response models:**
- Request models end with `Request`: `RunInvocationRequest`
- Response models end with `Response`: `RunStatusResponse`, `HealthResponse`
- All use `ConfigDict(extra="forbid")`

**Authentication middleware:**
- Global HTTP middleware in `src/zeroth/service/app.py` authenticates every request
- Sets `request.state.principal` as `AuthenticatedPrincipal`
- Returns 401 JSON on authentication failure
- Propagates/generates `X-Correlation-ID` header

**Authorization:**
- Call `require_permission(request, Permission.RUN_CREATE)` in route handlers
- Call `require_deployment_scope(request, deployment)` for tenant isolation
- Module: `src/zeroth/service/authorization.py`

## Logging and Observability

**Logging framework:** No dedicated logging library (no `logging` or `structlog` usage detected).

**Correlation ID propagation** via `contextvars` at `src/zeroth/observability/correlation.py`:
```python
from zeroth.observability.correlation import get_correlation_id, new_correlation_id, set_correlation_id
```
- Generated per HTTP request in middleware
- Propagated via `X-Correlation-ID` header

**Metrics:** In-process Prometheus-format collector at `src/zeroth/observability/metrics.py`:
- `MetricsCollector` dataclass with thread-safe counters, gauges, histograms
- Rendered as Prometheus text format

**Audit trail:** Structured audit records stored in DB via `src/zeroth/audit/repository.py`.

## Comments and Docstrings

**Module docstrings:**
- Every module has a docstring (1-3 sentences)
- Conversational, second-person style: "You won't create this directly", "Use this as a context manager"

**Class and method docstrings:**
- Every public class and method has a docstring
- Written in plain English; concise one-liners or 2-3 sentence paragraphs
- Private methods also have docstrings in most cases

**Inline comments:**
- Used sparingly; explain "why" not "what"
- Example: `# Keep the last error so callers can inspect what ultimately went wrong`

## Type Annotation Usage

**Completeness:**
- All function signatures fully annotated (parameters and return types)
- `from __future__ import annotations` in every module enables PEP 604 union syntax

**Patterns to follow:**
- `str | None` (not `Optional[str]`)
- `list[str]` (not `List[str]`)
- `dict[str, Any]` (not `Dict[str, Any]`)
- `Literal["agent", "executable_unit"]` for discriminators
- `Annotated[..., Field(...)]` for discriminated unions
- `collections.abc.Iterator`, `collections.abc.Mapping`, `collections.abc.Sequence` for abstract container types
- `TYPE_CHECKING` guard for import-only types to avoid circular imports

**Protocol classes:**
- Use `typing.Protocol` for structural subtyping at API boundaries
- Examples: `ServiceBootstrapLike` in `src/zeroth/service/app.py`, `RunApiBootstrapLike` in `src/zeroth/service/run_api.py`

## Module Design

**Exports:**
- Each package `__init__.py` re-exports key public symbols with `__all__`
- Example: `src/zeroth/storage/__init__.py` re-exports `SQLiteDatabase`, `Migration`, `RedisConfig`, etc.

**Package structure per domain:**
- `models.py` -- Pydantic data models
- `errors.py` -- Domain exception hierarchy
- `repository.py` -- Persistence layer (SQLite)
- Service/logic modules: `evaluator.py`, `executor.py`, `guard.py`, etc.
- API routes live separately in `src/zeroth/service/*_api.py`

**Dataclasses vs Pydantic:**
- Use Pydantic `BaseModel` for serialized/persisted data and API schemas
- Use `@dataclass` for internal containers and helpers: `Migration`, `MetricsCollector`, `ExecutableUnitBinding`
- Use `@dataclass(frozen=True, slots=True)` for immutable value objects: `Migration`
- Use `@dataclass(slots=True)` for mutable test doubles: `BlockingAgentRunner`, `FailingAgentRunner`

---

*Convention analysis: 2026-04-05*
