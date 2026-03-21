---
name: python-testing
description: Python testing patterns using pytest — fixtures, mocking, parametrization, async tests, and TDD methodology. Always activate when the user is writing, fixing, or designing tests; setting up a test suite; asking about coverage; or working with pytest configuration. Trigger on phrases like "write a test", "add tests for", "how do I test", "mock this", "fixture for", "parametrize", "test coverage", "pytest setup", "TDD", "how do I assert", "test this function", or any request explicitly about testing Python code. Also activate proactively when the user is writing new Python code without any tests — TDD means tests come first.
---

# Python Testing Patterns

Comprehensive pytest patterns for Python applications — TDD, fixtures, mocking, parametrization, and async testing.

## Workflow

When this skill activates:

1. **Identify the task** — new test suite, specific test case, mocking problem, coverage gap, config setup, or TDD session.
2. **Apply TDD by default** when writing new code — red (failing test) → green (minimal implementation) → refactor.
3. **Navigate to the right section** below. For quick lookups, use the Quick Reference table.
4. **Read `references/patterns.md`** for: markers and test selection, async testing, exception testing, file I/O testing, test organization, API/DB patterns, and full pytest config.
5. **Flag anti-patterns proactively** if spotted in user's existing tests.

---

## Core Philosophy

### TDD Cycle

```python
# Step 1 RED: write a failing test
def test_parse_price():
    assert parse_price("$12.99") == 12.99

# Step 2 GREEN: write minimal code to pass
def parse_price(s: str) -> float:
    return float(s.lstrip("$"))

# Step 3 REFACTOR: improve without breaking the test
def parse_price(s: str) -> float:
    return float(s.replace("$", "").replace(",", "").strip())
```

### Coverage

```bash
pytest --cov=mypackage --cov-report=term-missing --cov-report=html
```

Target 80%+ overall; 100% on critical paths (auth, payment, data validation). Coverage measures lines executed — it does not prove correctness.

---

## Fixtures

### Basic and Setup/Teardown

```python
@pytest.fixture
def sample_user():
    return {"name": "Alice", "age": 30}

@pytest.fixture
def database():
    db = Database(":memory:")
    db.create_tables()
    yield db        # test runs here
    db.close()      # teardown — runs even if the test fails
```

### Fixture Scopes

```python
@pytest.fixture                      # function (default) — fresh per test
def temp_config(): ...

@pytest.fixture(scope="module")      # once per test module
def module_db(): ...

@pytest.fixture(scope="session")     # once per entire test run
def shared_resource(): ...
```

Use `scope="session"` for expensive resources (DB connections, ML models). Use `scope="function"` for anything that mutates state.

### conftest.py — Shared Fixtures

```python
# tests/conftest.py  — auto-discovered by pytest, no import needed
@pytest.fixture
def client():
    app = create_app(testing=True)
    with app.test_client() as c:
        yield c

@pytest.fixture
def auth_headers(client):
    resp = client.post("/api/login", json={"username": "test", "password": "test"})
    return {"Authorization": f"Bearer {resp.json['token']}"}
```

### Autouse and Parametrized Fixtures

```python
@pytest.fixture(autouse=True)
def reset_state():
    """Runs automatically before every test in scope."""
    Config.reset()
    yield
    Config.cleanup()

@pytest.fixture(params=["sqlite", "postgres"])
def db(request):
    """Test runs once per param value."""
    return make_db(request.param)
```

---

## Parametrization

```python
# Basic
@pytest.mark.parametrize("value,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
])
def test_upper(value, expected):
    assert value.upper() == expected

# With readable IDs — shows in output as test_email[valid] etc.
@pytest.mark.parametrize("addr,valid", [
    ("user@example.com", True),
    ("not-an-email",     False),
    ("@no-local.com",    False),
], ids=["valid", "missing-at", "missing-local"])
def test_email(addr, valid):
    assert is_valid_email(addr) is valid
```

Prefer `ids=` whenever the parameter values aren't self-explanatory in test output.

---

## Mocking

Use `pytest-mock`'s `mocker` fixture instead of raw `@patch` decorators for cleaner syntax and automatic cleanup:

```python
# Preferred: pytest-mock (pip install pytest-mock)
def test_api_call(mocker):
    mock = mocker.patch("mypackage.requests.get")
    mock.return_value.json.return_value = {"status": "ok"}

    result = fetch_data()

    mock.assert_called_once_with("https://api.example.com/data")
    assert result["status"] == "ok"

# Alternative: unittest.mock @patch decorator
from unittest.mock import patch, Mock, MagicMock, mock_open

@patch("mypackage.external_api_call")
def test_with_patch(api_mock):           # mock injected as last arg after self
    api_mock.return_value = {"status": "ok"}
    result = my_function()
    api_mock.assert_called_once()
    assert result["status"] == "ok"
```

### ⚠️ Decorator Stacking Order — Common Footgun

When stacking multiple `@patch` decorators, mocks are injected **bottom-up** (innermost decorator → first argument):

```python
@patch("mypackage.ServiceB")   # injected SECOND → mock_b
@patch("mypackage.ServiceA")   # injected FIRST  → mock_a
def test_stacked(mock_a, mock_b):
    ...  # mock_a = ServiceA, mock_b = ServiceB
```

This trips up nearly everyone. Prefer `mocker.patch()` which avoids argument ordering entirely.

### Side Effects, Exceptions, Properties

```python
# Raise an exception
mock.side_effect = ConnectionError("timeout")

# Return different values on successive calls
mock.side_effect = [{"page": 1}, {"page": 2}, StopIteration]

# Mock a property
from unittest.mock import PropertyMock
type(mock_obj).debug = PropertyMock(return_value=True)

# Mock context manager (file open)
mocker.patch("builtins.open", mock_open(read_data="file content"))
```

### Always Use autospec

```python
# autospec catches API misuse at mock definition time
mock_db = mocker.patch("mypackage.DBConnection", autospec=True)
mock_db.return_value.query("SELECT 1")
# Raises AttributeError if DBConnection has no query() — prevents silent test rot
```

### Mock vs MagicMock

- `Mock` — basic mock; magic methods (`__len__`, `__iter__`, `__enter__`) not pre-configured
- `MagicMock` — magic methods pre-configured; use for context managers, iterables, and containers
- `mocker.patch()` returns `MagicMock` by default

---

## Assertions

```python
assert result == expected           # equality
assert result is None               # identity
assert isinstance(result, list)     # type
assert "error" in message           # membership
assert 0 < score <= 100             # range

with pytest.raises(ValueError, match="invalid input"):
    validate("")                    # match= is a regex against str(exception)

with pytest.raises(CustomError) as exc_info:
    risky_call()
assert exc_info.value.code == 400   # inspect exception attributes
```

---

## Quick Reference

| Pattern | Use it for |
|---|---|
| `@pytest.fixture` | Reusable setup/teardown |
| `yield` in fixture | Teardown after yield runs even on failure |
| `scope="session"` | Expensive shared resources |
| `conftest.py` | Cross-module shared fixtures |
| `@pytest.mark.parametrize` | Multiple inputs, one test function |
| `ids=["name1", ...]` | Readable parametrize output |
| `mocker.patch()` | Mock with auto-cleanup (pytest-mock) |
| `autospec=True` | Catch wrong method signatures in mocks |
| `MagicMock` | Context managers, iterables |
| `mock.side_effect` | Exceptions or sequential return values |
| `pytest.raises(Exc, match=r"...")` | Assert exception type and message |
| `tmp_path` | Built-in temp directory (pathlib), auto-cleaned |
| `--lf` | Re-run only last failed tests |
| `-x` | Stop at first failure |
| `-k "pattern"` | Run tests matching name pattern |

---

## Best Practices

**DO:**
- Write tests before code (TDD: red → green → refactor)
- Test one behavior per test function
- Use descriptive names: `test_login_with_expired_token_returns_401`
- Use `autospec=True` on all mocks
- Use `tmp_path` for file I/O (never raw `tempfile` in tests)
- Test edge cases: `None`, empty string, empty list, boundary values
- Keep tests independent — no shared mutable state between tests

**DON'T:**
- Test implementation details — test observable behavior
- Use `@patch` stacks without carefully checking argument order
- Put `--disable-warnings` in `addopts` — it hides real problems
- Use `tmpdir` — it's deprecated; use `tmp_path` instead
- Catch exceptions inside tests — use `pytest.raises` instead
- Write tests that pass only in a specific order

---

For extended patterns (markers, async testing, API/DB testing, test organization, full pytest config), see `references/patterns.md`.
