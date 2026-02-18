# Error Handling and Testing

## Exception Handling

### Basic Exception Handling

```python
# try/except/else/finally
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero")
except (TypeError, ValueError) as e:
    print(f"Type or value error: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
else:
    # Runs if no exception occurred
    print(f"Result: {result}")
finally:
    # Always runs (cleanup)
    print("Cleanup complete")

# Catching all exceptions (avoid in production)
try:
    risky_operation()
except:  # Bare except - catches everything including KeyboardInterrupt
    pass

# Better: catch Exception (not KeyboardInterrupt, SystemExit)
try:
    risky_operation()
except Exception as e:
    print(f"Error: {e}")
```

### Exception Hierarchy

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── StopIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── AssertionError
    ├── AttributeError
    ├── BufferError
    ├── EOFError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── MemoryError
    ├── NameError
    │   └── UnboundLocalError
    ├── OSError
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── RuntimeError
    │   └── RecursionError
    ├── TypeError
    ├── ValueError
    │   └── UnicodeError
    └── Warning
```

### Raising Exceptions

```python
# Raise exception
raise ValueError("Invalid value")

# Re-raise current exception
try:
    risky_operation()
except Exception:
    log_error()
    raise  # Re-raises the same exception

# Raise with cause (exception chaining)
try:
    int("not a number")
except ValueError as e:
    raise RuntimeError("Conversion failed") from e
# RuntimeError: Conversion failed
#   Caused by: ValueError: invalid literal...

# Suppress original exception (rarely needed)
raise RuntimeError("New error") from None
```

### Custom Exceptions

```python
# Simple custom exception
class ValidationError(Exception):
    """Raised when validation fails."""
    pass

# Custom exception with attributes
class APIError(Exception):
    """API request failed."""

    def __init__(self, message, status_code=None, response=None):
        super().__init__(message)
        self.status_code = status_code
        self.response = response

    def __str__(self):
        return f"APIError({self.status_code}): {super().__str__()}"

# Usage
try:
    raise APIError("Not found", status_code=404)
except APIError as e:
    print(e.status_code)  # 404

# Exception hierarchy for your library
class MyLibraryError(Exception):
    """Base exception for my library."""
    pass

class ConfigError(MyLibraryError):
    """Configuration error."""
    pass

class ConnectionError(MyLibraryError):
    """Connection error."""
    pass

# Users can catch all with MyLibraryError
# Or specific ones with subclasses
```

### Context Managers for Error Handling

```python
from contextlib import contextmanager

# Using class
class FileHandler:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode

    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        # Return True to suppress exception, False to propagate
        if exc_type is ValueError:
            print(f"Suppressed ValueError: {exc_val}")
            return True
        return False

# Using decorator
@contextmanager
def file_handler(filename, mode):
    f = open(filename, mode)
    try:
        yield f
    finally:
        f.close()

# Suppress specific exceptions
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('file.txt')  # No error if file doesn't exist
```

### Exception Best Practices

```python
# 1. Be specific with exception types
# Bad
try:
    value = data['key']
except:
    value = None

# Good
try:
    value = data['key']
except KeyError:
    value = None

# 2. Don't use exceptions for flow control
# Bad
try:
    return items[index]
except IndexError:
    return None

# Good (EAFP - Easier to Ask Forgiveness than Permission)
# Actually, the above IS Pythonic. But for simple checks:
if index < len(items):
    return items[index]
return None

# 3. Log exceptions properly
import logging

try:
    risky_operation()
except Exception:
    logging.exception("Operation failed")  # Includes traceback

# 4. Clean up resources with finally or context managers
# Bad
f = open('file.txt')
try:
    data = f.read()
finally:
    f.close()

# Good
with open('file.txt') as f:
    data = f.read()

# 5. Document exceptions
def divide(a, b):
    """Divide a by b.

    Raises:
        ZeroDivisionError: If b is zero.
        TypeError: If a or b is not numeric.
    """
    return a / b
```

---

## Assertions

```python
# Assert for debugging (disabled with python -O)
assert condition, "Error message"

# Examples
def calculate_average(numbers):
    assert len(numbers) > 0, "Cannot calculate average of empty list"
    return sum(numbers) / len(numbers)

# Don't use assert for:
# - Input validation (use exceptions)
# - Data validation in production
# Assert is for catching programming errors

# Good use: invariants, preconditions during development
def binary_search(arr, target):
    assert arr == sorted(arr), "Array must be sorted"
    # ... implementation
```

---

## Testing with unittest

### Basic unittest

```python
import unittest

def add(a, b):
    return a + b

class TestAdd(unittest.TestCase):
    """Test cases for add function."""

    def setUp(self):
        """Run before each test method."""
        self.data = [1, 2, 3]

    def tearDown(self):
        """Run after each test method."""
        self.data = None

    @classmethod
    def setUpClass(cls):
        """Run once before all tests in class."""
        cls.shared_resource = "shared"

    @classmethod
    def tearDownClass(cls):
        """Run once after all tests in class."""
        cls.shared_resource = None

    def test_add_positive(self):
        """Test adding positive numbers."""
        self.assertEqual(add(2, 3), 5)

    def test_add_negative(self):
        """Test adding negative numbers."""
        self.assertEqual(add(-1, -1), -2)

    def test_add_zero(self):
        """Test adding zero."""
        self.assertEqual(add(0, 5), 5)

if __name__ == '__main__':
    unittest.main()
```

### unittest Assertions

```python
class TestAssertions(unittest.TestCase):

    def test_equality(self):
        self.assertEqual(a, b)          # a == b
        self.assertNotEqual(a, b)       # a != b

    def test_truth(self):
        self.assertTrue(x)              # bool(x) is True
        self.assertFalse(x)             # bool(x) is False

    def test_identity(self):
        self.assertIs(a, b)             # a is b
        self.assertIsNot(a, b)          # a is not b
        self.assertIsNone(x)            # x is None
        self.assertIsNotNone(x)         # x is not None

    def test_membership(self):
        self.assertIn(a, b)             # a in b
        self.assertNotIn(a, b)          # a not in b

    def test_types(self):
        self.assertIsInstance(a, SomeClass)
        self.assertNotIsInstance(a, SomeClass)

    def test_comparison(self):
        self.assertGreater(a, b)        # a > b
        self.assertGreaterEqual(a, b)   # a >= b
        self.assertLess(a, b)           # a < b
        self.assertLessEqual(a, b)      # a <= b

    def test_exceptions(self):
        with self.assertRaises(ValueError):
            int("not a number")

        with self.assertRaises(ValueError) as context:
            int("not a number")
        self.assertIn("invalid literal", str(context.exception))

    def test_almost_equal(self):
        self.assertAlmostEqual(1.1, 1.100001, places=4)

    def test_regex(self):
        self.assertRegex("hello123", r'\d+')
        self.assertNotRegex("hello", r'\d+')
```

### Test Organization

```python
# Skip tests
class TestSkipping(unittest.TestCase):

    @unittest.skip("Demonstrating skip")
    def test_skipped(self):
        pass

    @unittest.skipIf(sys.version_info < (3, 10), "Requires Python 3.10+")
    def test_version_specific(self):
        pass

    @unittest.skipUnless(sys.platform == 'linux', "Linux only")
    def test_linux_only(self):
        pass

    @unittest.expectedFailure
    def test_expected_to_fail(self):
        self.assertEqual(1, 2)

# Subtests
class TestSubtests(unittest.TestCase):

    def test_multiple_values(self):
        for value in [1, 2, 3, 4, 5]:
            with self.subTest(value=value):
                self.assertTrue(value < 10)
```

---

## Testing with pytest

### Basic pytest

```python
# test_example.py
def add(a, b):
    return a + b

# Simple test function (no class needed)
def test_add_positive():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, -1) == -2

# Run with: pytest test_example.py

# Test class (no need to inherit)
class TestAdd:
    def test_positive(self):
        assert add(2, 3) == 5

    def test_negative(self):
        assert add(-1, -1) == -2
```

### pytest Fixtures

```python
import pytest

# Basic fixture
@pytest.fixture
def sample_data():
    return {"name": "Alice", "age": 30}

def test_name(sample_data):
    assert sample_data["name"] == "Alice"

# Fixture with setup and teardown
@pytest.fixture
def database():
    db = Database()
    db.connect()
    yield db  # Test runs here
    db.disconnect()

# Fixture scope
@pytest.fixture(scope="module")  # function, class, module, session
def expensive_resource():
    return create_expensive_resource()

# Fixture with parameters
@pytest.fixture(params=[1, 2, 3])
def number(request):
    return request.param

def test_is_positive(number):
    assert number > 0  # Runs 3 times

# Auto-use fixture
@pytest.fixture(autouse=True)
def setup_logging():
    logging.basicConfig(level=logging.DEBUG)

# Built-in fixtures
def test_temp_file(tmp_path):
    file = tmp_path / "test.txt"
    file.write_text("hello")
    assert file.read_text() == "hello"

def test_capture_output(capsys):
    print("hello")
    captured = capsys.readouterr()
    assert captured.out == "hello\n"

def test_monkeypatch(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key")
    assert os.environ["API_KEY"] == "test-key"
```

### pytest Parametrize

```python
import pytest

# Parametrized tests
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    assert add(a, b) == expected

# Multiple parametrize (cartesian product)
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    # Runs 4 times: (1,10), (1,20), (2,10), (2,20)
    assert x * y == x * y

# IDs for better test names
@pytest.mark.parametrize("input,expected", [
    pytest.param("hello", 5, id="simple_word"),
    pytest.param("", 0, id="empty_string"),
    pytest.param("hello world", 11, id="with_space"),
])
def test_length(input, expected):
    assert len(input) == expected
```

### pytest Markers

```python
import pytest

# Built-in markers
@pytest.mark.skip(reason="Not implemented yet")
def test_skipped():
    pass

@pytest.mark.skipif(sys.version_info < (3, 10), reason="Needs 3.10+")
def test_version():
    pass

@pytest.mark.xfail(reason="Known bug")
def test_expected_fail():
    assert False

# Custom markers (register in pytest.ini or pyproject.toml)
@pytest.mark.slow
def test_slow_operation():
    time.sleep(10)

@pytest.mark.integration
def test_database():
    pass

# Run specific markers: pytest -m "slow"
# Skip markers: pytest -m "not slow"

# pytest.ini
# [pytest]
# markers =
#     slow: marks tests as slow
#     integration: marks as integration test
```

### Testing Exceptions

```python
import pytest

def test_raises_value_error():
    with pytest.raises(ValueError):
        int("not a number")

def test_raises_with_message():
    with pytest.raises(ValueError) as exc_info:
        int("not a number")
    assert "invalid literal" in str(exc_info.value)

def test_raises_match():
    with pytest.raises(ValueError, match=r"invalid literal"):
        int("not a number")

# Check no exception raised
def test_no_exception():
    result = add(1, 2)  # Should not raise
    assert result == 3
```

### Mocking with pytest

```python
from unittest.mock import Mock, patch, MagicMock

# Basic mock
mock = Mock()
mock.method.return_value = 42
mock.method()  # 42
mock.method.assert_called_once()

# Patching
class UserService:
    def get_user(self, user_id):
        return api.fetch_user(user_id)

def test_get_user():
    with patch('module.api.fetch_user') as mock_fetch:
        mock_fetch.return_value = {"id": 1, "name": "Alice"}
        service = UserService()
        user = service.get_user(1)
        assert user["name"] == "Alice"
        mock_fetch.assert_called_once_with(1)

# Patch as decorator
@patch('module.api.fetch_user')
def test_get_user(mock_fetch):
    mock_fetch.return_value = {"id": 1, "name": "Alice"}
    # ...

# MagicMock for magic methods
mock = MagicMock()
mock.__len__.return_value = 5
len(mock)  # 5

# Spec to ensure mock matches interface
mock = Mock(spec=SomeClass)
# mock.nonexistent_method()  # AttributeError

# pytest-mock plugin (simpler syntax)
def test_with_mocker(mocker):
    mock_fetch = mocker.patch('module.api.fetch_user')
    mock_fetch.return_value = {"id": 1}
    # ...
```

---

## Test Organization

### Project Structure

```
my_project/
├── src/
│   └── my_module/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # Shared fixtures
│   ├── test_core.py
│   ├── test_utils.py
│   └── integration/
│       ├── __init__.py
│       └── test_api.py
├── pytest.ini
└── pyproject.toml
```

### conftest.py

```python
# tests/conftest.py
import pytest

@pytest.fixture(scope="session")
def database():
    """Database connection for all tests."""
    db = Database()
    db.connect()
    yield db
    db.disconnect()

@pytest.fixture
def user(database):
    """Create test user."""
    user = database.create_user("test@example.com")
    yield user
    database.delete_user(user.id)

# Shared fixtures are automatically available to all tests
```

### pyproject.toml Configuration

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --tb=short"
markers = [
    "slow: marks tests as slow",
    "integration: marks as integration test",
]

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "raise NotImplementedError",
]
```

---

## Test Coverage

```bash
# Install
pip install pytest-cov

# Run with coverage
pytest --cov=my_module tests/

# Coverage report
pytest --cov=my_module --cov-report=html tests/

# Coverage configuration in pyproject.toml
# [tool.coverage.run]
# source = ["src"]
# branch = true
#
# [tool.coverage.report]
# fail_under = 80
```

---

## Interview Questions

```python
# Q1: What's the difference between try/except and assert?
"""
try/except: Handle expected runtime errors (recoverable)
assert: Debug check for programming errors (can be disabled with -O)

Use exceptions for input validation, external errors.
Use asserts for invariants during development.
"""

# Q2: How would you test code that depends on time?
from unittest.mock import patch
from datetime import datetime

@patch('module.datetime')
def test_time_dependent(mock_datetime):
    mock_datetime.now.return_value = datetime(2024, 1, 1, 12, 0, 0)
    # Test code that uses datetime.now()

# Q3: How do you test async code?
import pytest

@pytest.mark.asyncio
async def test_async_function():
    result = await async_function()
    assert result == expected

# Q4: What's the difference between setUp and fixtures?
"""
setUp (unittest):
- Runs before each test method
- Inheritance-based

fixtures (pytest):
- Dependency injection
- Can have different scopes (function, class, module, session)
- Can be parametrized
- More composable and reusable
"""

# Q5: How do you test private methods?
"""
Generally, don't test private methods directly.
Test through public API.
If you feel need to test private methods, consider:
- Moving logic to separate testable class
- Making method public if it has independent value
"""
```
