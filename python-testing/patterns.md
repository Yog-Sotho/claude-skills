# Python Testing — Extended Patterns

Extended reference for markers, async testing, API/DB patterns, test organization, and pytest configuration.

---

## Markers and Test Selection

### Defining and Using Markers

```python
@pytest.mark.slow
def test_data_export():
    ...

@pytest.mark.integration
def test_payment_gateway():
    ...

@pytest.mark.unit
def test_price_parser():
    ...
```

```bash
pytest -m "not slow"               # skip slow tests in CI fast path
pytest -m integration              # run only integration tests
pytest -m "unit and not slow"      # compose expressions
pytest -k "test_user"              # match test name pattern
```

Always register markers in config to avoid `PytestUnknownMarkWarning`:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with -m 'not slow')",
    "integration: marks tests requiring external services",
    "unit: marks pure unit tests",
]
```

---

## Async Testing

### Setup

```bash
pip install pytest-asyncio
```

Configure `asyncio_mode = "auto"` to avoid decorating every async test:

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"    # auto-detects async test functions — no @pytest.mark.asyncio needed
```

### Async Tests and Fixtures

```python
# With asyncio_mode = "auto", no decorator needed
async def test_async_fetch():
    result = await fetch_data()
    assert result["status"] == "ok"

# Async fixtures require @pytest_asyncio.fixture (not @pytest.fixture)
import pytest_asyncio

@pytest_asyncio.fixture
async def async_client():
    app = create_app()
    async with app.test_client() as client:
        yield client

async def test_endpoint(async_client):
    response = await async_client.get("/api/users")
    assert response.status == 200
```

### Mocking Async Functions

```python
async def test_async_with_mock(mocker):
    mock = mocker.patch("mypackage.async_api_call", new_callable=AsyncMock)
    mock.return_value = {"data": "ok"}

    result = await call_service()

    mock.assert_awaited_once()
    assert result["data"] == "ok"
```

Use `AsyncMock` (from `unittest.mock`) for async callables. Regular `Mock` doesn't support `await`.

---

## Exception Testing

```python
def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_custom_error_code():
    with pytest.raises(CustomError) as exc_info:
        process(bad_input)
    assert exc_info.value.code == 422
    assert "invalid" in str(exc_info.value).lower()

# match= uses re.search against str(exception)
def test_error_message():
    with pytest.raises(ValueError, match=r"must be positive"):
        validate_age(-1)
```

---

## File I/O Testing

Always use `tmp_path` (built-in, pathlib-based, auto-cleaned). Never use `tmpdir` — it is deprecated.

```python
def test_write_report(tmp_path):
    output = tmp_path / "report.csv"
    write_report(output)
    assert output.exists()
    assert "header" in output.read_text()

def test_process_file(tmp_path):
    source = tmp_path / "input.txt"
    source.write_text("hello world")
    result = process_file(source)
    assert result == "HELLO WORLD"
```

For mocking file reads without touching disk:

```python
def test_read_config(mocker):
    mocker.patch("builtins.open", mock_open(read_data='{"debug": true}'))
    config = load_config("config.json")
    assert config["debug"] is True
```

---

## API Testing

### Flask / FastAPI

```python
# tests/conftest.py
@pytest.fixture
def client():
    app = create_app(testing=True)
    with app.test_client() as c:
        yield c

# FastAPI — use httpx
@pytest_asyncio.fixture
async def async_client():
    from httpx import AsyncClient
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
```

```python
def test_get_user(client):
    response = client.get("/api/users/1")
    assert response.status_code == 200
    assert response.json["id"] == 1

def test_create_user(client, auth_headers):
    response = client.post("/api/users",
        json={"name": "Alice", "email": "alice@example.com"},
        headers=auth_headers,
    )
    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

---

## Database Testing

Use `begin_nested()` + `rollback()` to keep tests fast and isolated (SQLAlchemy 2.0 syntax):

```python
@pytest.fixture
def db_session():
    with Session(engine) as session:
        session.begin_nested()      # savepoint
        yield session
        session.rollback()          # roll back to savepoint — DB left clean

def test_create_user(db_session):
    user = User(name="Alice", email="alice@example.com")
    db_session.add(user)
    db_session.flush()              # write to DB within transaction

    retrieved = db_session.get(User, user.id)
    assert retrieved.email == "alice@example.com"
```

---

## Test Organization

### Directory Structure

```
tests/
├── conftest.py            # shared fixtures — available to all tests automatically
├── unit/
│   ├── conftest.py        # unit-only fixtures
│   ├── test_models.py
│   └── test_services.py
├── integration/
│   ├── conftest.py        # integration-only fixtures (DB, API clients)
│   └── test_api.py
└── e2e/
    └── test_user_flows.py
```

### Test Classes

Group closely related tests in a class — useful when tests share fixtures or test one component:

```python
class TestUserService:
    @pytest.fixture(autouse=True)
    def setup(self):
        self.service = UserService(repo=FakeRepository())

    def test_create_user(self):
        user = self.service.create_user("Alice")
        assert user.name == "Alice"

    def test_create_user_duplicate_raises(self):
        self.service.create_user("Alice")
        with pytest.raises(DuplicateUserError):
            self.service.create_user("Alice")
```

---

## pytest Configuration

### pyproject.toml (preferred)

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
asyncio_mode = "auto"
addopts = [
    "--strict-markers",     # fail on unknown markers
    "-ra",                  # show summary of all non-passing tests
    "--cov=mypackage",
    "--cov-report=term-missing",
    "--cov-report=html",
]
markers = [
    "slow: marks tests as slow (deselect with -m 'not slow')",
    "integration: marks tests requiring external services",
    "unit: marks pure unit tests",
]
```

### pytest.ini (alternative)

```ini
[pytest]
testpaths = tests
addopts =
    --strict-markers
    -ra
    --cov=mypackage
    --cov-report=term-missing
markers =
    slow: marks tests as slow
    integration: marks tests requiring external services
    unit: marks pure unit tests
```

Note: `--disable-warnings` is intentionally excluded — it silences real deprecation warnings from your own code.

---

## Running Tests — Command Reference

```bash
pytest                              # run all tests
pytest tests/test_utils.py          # single file
pytest tests/test_utils.py::test_fn # single test
pytest -v                           # verbose output
pytest -x                           # stop at first failure
pytest --maxfail=3                  # stop after N failures
pytest --lf                         # re-run only last failed
pytest -k "user and not slow"       # name pattern filter
pytest -m "not slow"                # marker filter
pytest --pdb                        # drop into debugger on failure
pytest --tb=short                   # shorter tracebacks
pytest -n auto                      # parallel (pip install pytest-xdist)
```
