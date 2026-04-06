# Testing Patterns

**Analysis Date:** 2026-04-05

## Test Framework

**Runner:**
- pytest >= 8.0
- pytest-asyncio >= 0.25
- Config: `pyproject.toml` `[tool.pytest.ini_options]`

**Async mode:**
- `asyncio_mode = "auto"` -- all `async def test_*` functions are automatically treated as async tests (no need for `@pytest.mark.asyncio` but it is sometimes used explicitly)

**Assertion Library:**
- Built-in `assert` statements (no third-party assertion library)

**Run Commands:**
```bash
uv sync                    # install/update deps
uv run pytest -v           # run all tests (280 tests collected)
uv run pytest tests/graph/ # run a specific package
uv run ruff check src/     # lint (run alongside tests)
```

## Test Count and Structure

**Total tests collected:** 280

**Test directories mirror source packages:**
```
tests/
├── __init__.py
├── conftest.py                    # Root conftest: sqlite_db fixture
├── test_smoke.py                  # Import smoke test
├── agent_runtime/                 # 5 test files
│   ├── test_agent_runtime.py
│   ├── test_tools.py
│   ├── test_thread_store.py
│   ├── test_runner_tools.py
│   └── test_runner_integration.py
├── approvals/
│   └── test_service.py
├── audit/
│   └── test_audit_repository.py
├── conditions/
│   ├── test_recorder.py
│   ├── test_evaluator.py
│   └── test_branch.py
├── contracts/
│   └── test_registry.py
├── deployments/
│   └── test_service.py
├── dispatch/
│   ├── test_lease.py
│   ├── test_recovery.py
│   └── test_worker.py
├── execution_units/
│   ├── test_adapters.py
│   ├── test_manifest.py
│   ├── test_io.py
│   ├── test_runner.py
│   ├── test_integrity.py
│   ├── test_sandbox.py
│   └── test_sandbox_hardening.py
├── graph/
│   ├── test_models.py
│   ├── test_validation.py
│   └── test_repository.py
├── guardrails/
│   ├── test_dead_letter.py
│   └── test_rate_limit.py
├── live_scenarios/
│   └── test_research_audit.py
├── mappings/
│   ├── test_validator.py
│   └── test_executor.py
├── memory/
│   └── test_connectors.py
├── observability/
│   ├── test_correlation.py
│   └── test_metrics.py
├── orchestrator/
│   └── test_runtime.py
├── policy/
│   ├── test_guard.py
│   └── test_runtime_enforcement.py
├── runs/
│   ├── conftest.py                # Domain-specific conftest: runs_db fixture
│   ├── test_models.py
│   └── test_repository.py
├── secrets/
│   ├── test_data_protection.py
│   └── test_provider.py
├── service/
│   ├── helpers.py                 # Shared test helpers (not a test file)
│   ├── test_app.py
│   ├── test_auth_api.py
│   ├── test_bearer_auth.py
│   ├── test_rbac_api.py
│   ├── test_tenant_isolation.py
│   ├── test_run_api.py
│   ├── test_thread_api.py
│   ├── test_audit_api.py
│   ├── test_evidence_api.py
│   ├── test_approval_api.py
│   ├── test_contract_api.py
│   ├── test_admin_api.py
│   ├── test_guardrails_api.py
│   ├── test_durable_dispatch.py
│   ├── test_metrics_endpoint.py
│   ├── test_e2e_phase4.py
│   └── test_e2e_phase5.py
└── storage/
    ├── test_json.py
    ├── test_sqlite.py
    └── test_redis.py
```

## Test File Organization

**Location:** Mirror pattern -- `tests/<package>/test_<module>.py` mirrors `src/zeroth/<package>/<module>.py`

**Naming:** Files are prefixed `test_` and named after the module or concept they test.

**Non-test helpers:** `tests/service/helpers.py` contains shared test utilities (not collected by pytest).

## Fixture Patterns

**Root conftest** at `tests/conftest.py`:
```python
@pytest.fixture
def sqlite_db(tmp_path: Path) -> SQLiteDatabase:
    return SQLiteDatabase(tmp_path / "zeroth.db")
```
- Provides a fresh SQLite database per test via `tmp_path`
- Used by nearly all tests that need persistence

**Domain-specific conftest** at `tests/runs/conftest.py`:
```python
@pytest.fixture
def runs_db(tmp_path: Path) -> SQLiteDatabase:
    return SQLiteDatabase(tmp_path / "runs.db")
```
- Separate DB fixture for runs tests (different DB file name)

**Fixture usage:**
- Fixtures are injected by name: `def test_something(sqlite_db) -> None:`
- `tmp_path` (pytest built-in) is used directly for file-system tests
- `monkeypatch` is used for patching: `monkeypatch.setattr("zeroth.service.bootstrap.deserialize_graph", ...)`

## Test Structure

**Sync tests (most common):**
```python
def test_graph_repository_round_trip_and_schema_version(sqlite_db) -> None:
    repository = GraphRepository(sqlite_db)
    graph = build_graph()

    saved = repository.save(graph)
    loaded = repository.get(graph.graph_id)

    assert saved == graph
    assert loaded == graph
```

**Async tests:**
```python
@pytest.mark.asyncio
async def test_agent_runner_validates_output_and_checkpoints_thread_state() -> None:
    config = AgentConfig(...)
    provider = DeterministicProviderAdapter([ProviderResponse(content='...')])
    runner = AgentRunner(config, provider)

    result = await runner.run(DemoInput(query="hello", secret="top-secret"), thread_id="thread-1")

    assert result.output_data == {"answer": "done", "score": 2}
```

**API/service tests:**
```python
def test_run_creation_accepts_input_and_supplied_thread_id(sqlite_db) -> None:
    service, _ = deploy_service(sqlite_db, agent_graph(graph_id="graph-run-create"))
    # Wire test runner
    service.orchestrator.agent_runners["agent-step"] = BlockingAgentRunner(...)
    app = bootstrap_app(sqlite_db, deployment_ref=service.deployment.deployment_ref)
    app.state.bootstrap = service

    with TestClient(app) as client:
        response = client.post("/runs", json={...}, headers=operator_headers())
        assert response.status_code == 202
```

**Error testing:**
```python
def test_graph_repository_rejects_mutating_published_graph(sqlite_db) -> None:
    repository = GraphRepository(sqlite_db)
    graph = repository.create(build_graph())
    repository.publish(graph.graph_id, graph.version)

    with pytest.raises(GraphLifecycleError, match="immutable"):
        repository.save(published.model_copy(update={"name": "Mutated"}))
```

**Naming conventions for test functions:**
- `test_<what_is_being_tested>` -- descriptive, often a full sentence in snake_case
- Positive cases: `test_run_creation_accepts_input_and_supplied_thread_id`
- Negative cases: `test_run_creation_rejects_invalid_input`
- Error cases: `test_bootstrap_service_fails_for_missing_deployment`
- Status/state: `test_run_status_reports_running_and_completed_state`

## Mocking and Test Doubles

**No mocking framework.** The codebase uses hand-written test doubles defined as dataclasses.

**Test doubles in** `tests/service/helpers.py`:

```python
@dataclass(slots=True)
class BlockingAgentRunner:
    """Blocks until manually released -- used to test concurrent behavior."""
    started: threading.Event
    release: threading.Event
    output_data: dict[str, Any]

    async def run(self, input_payload, *, thread_id=None, runtime_context=None):
        self.started.set()
        await asyncio.to_thread(self.release.wait)
        return SimpleNamespace(output_data=dict(self.output_data), audit_record={...})


@dataclass(slots=True)
class FailingAgentRunner:
    """Always raises RuntimeError -- used to test failure paths."""
    started: threading.Event

    async def run(self, input_payload, *, thread_id=None, runtime_context=None):
        self.started.set()
        raise RuntimeError("boom")


@dataclass(slots=True)
class CountingFinishRunner:
    """Counts calls and records last input -- used to verify resume behavior."""
    call_count: int = 0
    last_input: dict[str, Any] | None = None

    async def run(self, input_payload, *, thread_id=None, runtime_context=None):
        self.call_count += 1
        return SimpleNamespace(output_data={"value": int(input_payload["value"]) + 1}, ...)
```

**Deterministic provider adapter** in source at `src/zeroth/agent_runtime/`:
```python
provider = DeterministicProviderAdapter([
    ProviderResponse(content='{"answer":"done","score":2}')
])
```

**`monkeypatch` usage** for targeted patching:
```python
monkeypatch.setattr("zeroth.service.bootstrap.deserialize_graph", fake_deserialize_graph)
```

**What to mock:**
- Agent runners (inject `BlockingAgentRunner`, `FailingAgentRunner`, `CountingFinishRunner`)
- Provider adapters (inject `DeterministicProviderAdapter`)
- Specific functions via `monkeypatch.setattr` (rare, only when needed)

**What NOT to mock:**
- Repositories and storage (use real `SQLiteDatabase` with `tmp_path`)
- FastAPI app (use real `TestClient` with real app)
- Pydantic models (use real models, not mocks)

## Fixtures and Factories

**Graph factory** in `tests/graph/test_models.py` (reused across test files):
```python
def build_graph() -> Graph:
    return Graph(
        graph_id="graph-1",
        name="Governed Demo",
        version=1,
        status=GraphStatus.DRAFT,
        entry_step="agent-step",
        nodes=[AgentNode(...), ExecutableUnitNode(...), HumanApprovalNode(...)],
        edges=[Edge(...), Edge(...)],
    )
```

**Service deployment helpers** in `tests/service/helpers.py`:
```python
def deploy_service(sqlite_db, graph, *, deployment_ref="service-run-api", ...):
    """Set up a full deployed service for API testing."""
    # Creates graph repo, contract registry, deploys, bootstraps service
    ...

def agent_graph(*, graph_id: str, node_id: str = "agent-step") -> Graph:
    """Minimal single-agent-node graph for tests."""
    ...

def approval_graph(*, graph_id: str, node_id: str = "approval-step") -> Graph:
    """Minimal single-approval-node graph for tests."""
    ...
```

**Auth helpers** in `tests/service/helpers.py`:
```python
def default_service_auth_config() -> ServiceAuthConfig: ...
def operator_headers() -> dict[str, str]: ...
def reviewer_headers() -> dict[str, str]: ...
def admin_headers() -> dict[str, str]: ...
```

**Inline test models** (defined at test file scope):
```python
class DemoInput(BaseModel):
    query: str
    secret: str

class DemoOutput(BaseModel):
    answer: str
    score: int = Field(ge=0)
```

**Wait helper** for async behavior in sync tests:
```python
def wait_for(predicate, *, timeout: float = 3.0) -> None:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        if predicate():
            return
        time.sleep(0.01)
    raise AssertionError("timed out waiting for condition")
```

## Coverage

**Requirements:** No coverage threshold enforced. No `pytest-cov` in dev dependencies.

**Coverage shape (by test file count per domain):**
- `tests/service/` -- 17 test files (heaviest: API integration, auth, RBAC, multi-tenant)
- `tests/execution_units/` -- 7 test files (manifests, adapters, sandbox, integrity)
- `tests/agent_runtime/` -- 5 test files (runner, tools, thread store, integration)
- `tests/graph/` -- 3 test files (models, validation, repository)
- `tests/conditions/` -- 3 test files (recorder, evaluator, branch)
- `tests/dispatch/` -- 3 test files (lease, recovery, worker)
- `tests/storage/` -- 3 test files (json, sqlite, redis)
- `tests/runs/` -- 2 test files (models, repository)
- `tests/observability/` -- 2 test files (correlation, metrics)
- `tests/secrets/` -- 2 test files (provider, data protection)
- `tests/policy/` -- 2 test files (guard, runtime enforcement)
- `tests/guardrails/` -- 2 test files (dead letter, rate limit)
- Other domains: 1 test file each

## Test Types

**Unit Tests:**
- Model serialization, validation, lifecycle transitions: `tests/graph/test_models.py`
- Repository CRUD, migration idempotency: `tests/graph/test_repository.py`, `tests/storage/test_sqlite.py`
- Condition evaluation, branch logic: `tests/conditions/`
- Policy guard enforcement: `tests/policy/test_guard.py`

**Integration Tests:**
- Full API request/response through FastAPI `TestClient`: `tests/service/test_run_api.py`
- Bootstrap wiring: `tests/service/test_app.py`
- Auth/RBAC middleware: `tests/service/test_auth_api.py`, `tests/service/test_rbac_api.py`
- Durable dispatch: `tests/service/test_durable_dispatch.py`

**End-to-End Tests:**
- Phase-level validation: `tests/service/test_e2e_phase4.py`, `tests/service/test_e2e_phase5.py`
- Live scenarios: `tests/live_scenarios/test_research_audit.py`

**Smoke Test:**
- `tests/test_smoke.py` -- verifies package imports and docstring

## Common Patterns

**Async Testing:**
```python
@pytest.mark.asyncio
async def test_agent_runner_retries_on_validation_error_then_succeeds() -> None:
    config = AgentConfig(...)
    provider = DeterministicProviderAdapter([...])
    runner = AgentRunner(config, provider)
    result = await runner.run({"query": "hello", "secret": "x"})
    assert result.output_data == {"answer": "done", "score": 7}
```

**Error Testing:**
```python
with pytest.raises(GraphLifecycleError, match="immutable"):
    repository.save(published.model_copy(update={"name": "Mutated"}))

with pytest.raises(AgentTimeoutError) as excinfo:
    await runner.run(DemoInput(query="hello", secret="top-secret"))
assert "timed out" in str(excinfo.value)
```

**API Response Testing:**
```python
with TestClient(app) as client:
    response = client.post("/runs", json={...}, headers=operator_headers())
    assert response.status_code == 202
    payload = response.json()
    assert payload["status"] == "queued"
```

**Concurrent behavior testing:**
```python
started = threading.Event()
release = threading.Event()
service.orchestrator.agent_runners["agent-step"] = BlockingAgentRunner(
    started=started, release=release, output_data={"value": 42}
)
# ... start run via API ...
wait_for(started.is_set)          # runner has started
release.set()                     # let it finish
wait_for(lambda: get_status() == "succeeded")
```

## Phase Evidence

The project stores verification artifacts outside pytest:
- `phases/phase-*/artifacts/*.txt` -- durable evidence for phase completion
- Referenced in `PROGRESS.md` as proof of work

---

*Testing analysis: 2026-04-05*
