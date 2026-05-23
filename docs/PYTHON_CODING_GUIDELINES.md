# Python Coding Guidelines

## Error Handling

This is the most important section. Get this right.

### Rules

- **Never use bare `except:`** — it catches `KeyboardInterrupt` and `SystemExit`
- **Catch the narrowest exception possible** — `except json.JSONDecodeError:`, not `except Exception:`
- **Never swallow exceptions** — `except SomeError: pass` hides bugs. Log it or re-raise.
- **Preserve exception chains** — `raise AppError("msg") from original_error`
- **Don't catch just to log and re-raise identically** — let exceptions propagate to where they're handled
- **`except Exception as e:` only at top-level handlers** — entry points, CLI wrappers, API route handlers

### Structure

Use `try/except/else/finally` correctly:

```python
try:
    result = parse_config(path)
except FileNotFoundError:
    logger.error("config not found: %s", path)
    raise
except json.JSONDecodeError as e:
    raise ConfigError(f"invalid config: {path}") from e
else:
    # runs only on success — keep non-error code out of try
    apply_config(result)
finally:
    # always runs — cleanup only, prefer context managers
    cleanup()
```

### Custom Exceptions

Create domain exceptions. Never match on exception message strings.

```python
class AppError(Exception):
    """Base exception for application errors."""

class NotFoundError(AppError):
    pass

class ValidationError(AppError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"{field}: {message}")
```

## Formatting & Tooling

- **Ruff** for linting + formatting — replaces flake8, black, isort, pyupgrade in one tool
- **mypy** or **pyright** for type checking — run in CI, type hints without enforcement are comments
- All config in `pyproject.toml` — no `setup.py`, `setup.cfg`, `requirements.txt`

### Ruff Config

```toml
[tool.ruff]
target-version = "py312"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "S"]
```

## Type Hints

Modern syntax only (3.10+):

```python
# Yes
def find(name: str, limit: int = 10) -> list[dict[str, str]] | None: ...
def process(items: list[str]) -> None: ...

# No
from typing import Optional, List, Dict
def find(name: str) -> Optional[List[Dict[str, str]]]: ...
```

- `str | None` not `Optional[str]`
- `list[str]` not `List[str]`
- Annotate function signatures and return types
- Don't over-annotate locals — type checkers infer them
- 3.12+: use `def f[T](x: T) -> T` instead of `TypeVar`

## Project Structure

Use `src/` layout:

```text
myproject/
├── pyproject.toml
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── config.py
│       ├── storage/
│       │   ├── __init__.py
│       │   ├── base.py        # Protocol definition
│       │   ├── s3.py
│       │   ├── filesystem.py
│       │   └── memory.py
│       └── service.py
└── tests/
    ├── conftest.py
    └── test_service.py
```

- `src/` layout prevents accidental local imports
- Mirror `src/` layout in `tests/`

## Dependency Management

- **uv** for all new projects — handles venvs, lockfiles, Python versions
- `uv add` to add deps (updates `pyproject.toml`)
- Never `pip install` directly — always go through `pyproject.toml`

## Interfaces: Protocol vs ABC

- **Default to `Protocol`** — structural subtyping, no inheritance required. This is Python's equivalent of Go
  interfaces.
- **ABC only when** you need shared concrete methods or runtime enforcement

```python
from typing import Protocol

class BlobStore(Protocol):
    def put(self, key: str, data: bytes) -> None: ...
    def get(self, key: str) -> bytes: ...
    def delete(self, key: str) -> None: ...

# Any class with matching methods satisfies this — no inheritance needed
class S3Store:
    def put(self, key: str, data: bytes) -> None: ...
    def get(self, key: str) -> bytes: ...
    def delete(self, key: str) -> None: ...

class MemoryStore:
    def __init__(self) -> None:
        self._data: dict[str, bytes] = {}
    def put(self, key: str, data: bytes) -> None:
        self._data[key] = data
    def get(self, key: str) -> bytes:
        return self._data[key]
    def delete(self, key: str) -> None:
        del self._data[key]

def upload(store: BlobStore, key: str, data: bytes) -> None:
    store.put(key, data)
```

## Testing

Use pytest exclusively.

```python
import pytest

@pytest.mark.parametrize("input,expected", [
    ("valid@email.com", True),
    ("no-at-sign", False),
    ("", False),
])
def test_validate_email(input: str, expected: bool) -> None:
    assert validate_email(input) == expected
```

- `@pytest.mark.parametrize` for data-driven tests
- Fixtures with `yield` for setup/teardown
- Shared fixtures in `conftest.py`
- Scope fixtures deliberately: `function` (default), `session` (expensive resources)

## Logging

- **Never use `print()` for operational output** — use `logging` (stdlib) or `structlog`
- Log to stdout, let infrastructure route
- Use structured logging in production (JSON)

```python
import structlog

logger = structlog.get_logger()

logger.info("order.created", order_id=order.id, customer=order.customer_id)
```

## Anti-Patterns to Reject

These are patterns AI agents produce most often. Reject them on sight.

| Anti-Pattern | Fix |
| ------------ | --- |
| Bare `except:` or `except Exception: pass` | Catch specific exceptions, always log or re-raise |
| Mutable default arguments (`def f(x=[])`) | `def f(x: list[str] \| None = None)` |
| `import *` | Explicit imports always |
| God classes | Split by responsibility |
| ABC when Protocol suffices | Protocol by default |
| No error context in exceptions | `raise X("msg") from original` |
| Catching to log and re-raise identically | Let exceptions propagate |
