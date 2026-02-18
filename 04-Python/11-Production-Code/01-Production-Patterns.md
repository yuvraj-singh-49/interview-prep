# Production-Ready Python Code

## Logging

### Basic Logging

```python
import logging

# Basic configuration
logging.basicConfig(level=logging.INFO)

# Log messages
logging.debug("Debug message")    # Not shown (level too low)
logging.info("Info message")      # Shown
logging.warning("Warning message")
logging.error("Error message")
logging.critical("Critical message")

# Logging levels (numeric values)
# DEBUG    = 10
# INFO     = 20
# WARNING  = 30 (default)
# ERROR    = 40
# CRITICAL = 50
```

### Proper Logging Configuration

```python
import logging
import sys

# Configure with format
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler('app.log')
    ]
)

# Get logger for your module
logger = logging.getLogger(__name__)

# Use the logger
logger.info("Application started")
logger.error("An error occurred", exc_info=True)  # Include traceback

# Format placeholders
# %(asctime)s    - Timestamp
# %(name)s       - Logger name
# %(levelname)s  - Level (INFO, ERROR, etc.)
# %(message)s    - Log message
# %(filename)s   - Source filename
# %(lineno)d     - Line number
# %(funcName)s   - Function name
# %(pathname)s   - Full path
```

### Logger Hierarchy

```python
import logging

# Logger names follow dot-separated hierarchy
# Like Python packages: myapp.module1.submodule

root = logging.getLogger()           # Root logger
app = logging.getLogger('myapp')     # Application logger
module = logging.getLogger('myapp.module1')  # Module logger

# Settings propagate down
app.setLevel(logging.DEBUG)          # Affects all myapp.* loggers

# Prevent propagation (if needed)
module.propagate = False             # Don't send to parent loggers

# Best practice: Use __name__ as logger name
logger = logging.getLogger(__name__)
# Automatically matches module hierarchy
```

### Handlers and Formatters

```python
import logging
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

# Create logger
logger = logging.getLogger('myapp')
logger.setLevel(logging.DEBUG)

# Console handler
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
console_format = logging.Formatter('%(levelname)s - %(message)s')
console_handler.setFormatter(console_format)

# File handler with rotation
file_handler = RotatingFileHandler(
    'app.log',
    maxBytes=10*1024*1024,  # 10 MB
    backupCount=5           # Keep 5 backup files
)
file_handler.setLevel(logging.DEBUG)
file_format = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
file_handler.setFormatter(file_format)

# Time-based rotation
daily_handler = TimedRotatingFileHandler(
    'app.log',
    when='midnight',
    interval=1,
    backupCount=30
)

# Add handlers to logger
logger.addHandler(console_handler)
logger.addHandler(file_handler)
```

### Structured Logging (JSON)

```python
import logging
import json
from datetime import datetime

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno,
        }

        # Add exception info if present
        if record.exc_info:
            log_obj['exception'] = self.formatException(record.exc_info)

        # Add extra fields
        if hasattr(record, 'extra'):
            log_obj.update(record.extra)

        return json.dumps(log_obj)

# Usage
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger = logging.getLogger('myapp')
logger.addHandler(handler)

# Log with extra context
logger.info("User action", extra={
    'user_id': 123,
    'action': 'login',
    'ip_address': '192.168.1.1'
})
```

### Logging Configuration from File

```python
# logging.ini
"""
[loggers]
keys=root,myapp

[handlers]
keys=console,file

[formatters]
keys=standard

[logger_root]
level=WARNING
handlers=console

[logger_myapp]
level=DEBUG
handlers=console,file
qualname=myapp
propagate=0

[handler_console]
class=StreamHandler
level=INFO
formatter=standard
args=(sys.stdout,)

[handler_file]
class=FileHandler
level=DEBUG
formatter=standard
args=('app.log', 'a')

[formatter_standard]
format=%(asctime)s - %(name)s - %(levelname)s - %(message)s
"""

import logging.config
logging.config.fileConfig('logging.ini')

# Or use dictConfig (more flexible)
import logging.config

LOGGING_CONFIG = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'standard': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard',
        },
    },
    'loggers': {
        '': {  # Root logger
            'handlers': ['console'],
            'level': 'INFO',
        },
        'myapp': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```

---

## Configuration Management

### Environment Variables

```python
import os

# Get environment variable
api_key = os.environ.get('API_KEY')
debug = os.environ.get('DEBUG', 'false').lower() == 'true'
port = int(os.environ.get('PORT', 8080))

# Required environment variable
api_key = os.environ['API_KEY']  # Raises KeyError if missing

# Better: explicit check
api_key = os.environ.get('API_KEY')
if not api_key:
    raise ValueError("API_KEY environment variable required")

# Using python-dotenv for .env files
# pip install python-dotenv
from dotenv import load_dotenv

load_dotenv()  # Load from .env file
api_key = os.environ.get('API_KEY')

# .env file (add to .gitignore!)
"""
API_KEY=your-secret-key
DATABASE_URL=postgresql://localhost/mydb
DEBUG=true
"""
```

### Configuration Classes

```python
from dataclasses import dataclass
from typing import Optional
import os

@dataclass
class Config:
    """Application configuration."""

    # Required
    api_key: str
    database_url: str

    # Optional with defaults
    debug: bool = False
    log_level: str = "INFO"
    port: int = 8080
    workers: int = 4

    @classmethod
    def from_env(cls) -> 'Config':
        """Load configuration from environment variables."""
        return cls(
            api_key=cls._require_env('API_KEY'),
            database_url=cls._require_env('DATABASE_URL'),
            debug=os.environ.get('DEBUG', 'false').lower() == 'true',
            log_level=os.environ.get('LOG_LEVEL', 'INFO'),
            port=int(os.environ.get('PORT', 8080)),
            workers=int(os.environ.get('WORKERS', 4)),
        )

    @staticmethod
    def _require_env(name: str) -> str:
        value = os.environ.get(name)
        if not value:
            raise ValueError(f"Required environment variable {name} not set")
        return value

# Usage
config = Config.from_env()
print(config.debug)
print(config.port)
```

### Pydantic Settings (Recommended)

```python
# pip install pydantic-settings
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    """Application settings with validation."""

    api_key: str
    database_url: str
    debug: bool = False
    log_level: str = "INFO"
    port: int = Field(default=8080, ge=1, le=65535)
    workers: int = Field(default=4, ge=1)

    class Config:
        env_file = '.env'
        env_file_encoding = 'utf-8'
        case_sensitive = False  # API_KEY or api_key both work

# Auto-loads from environment and .env file
settings = Settings()

# Validation happens automatically
# Invalid port will raise ValidationError
```

### Config Files (YAML/TOML)

```python
# config.yaml
"""
database:
  host: localhost
  port: 5432
  name: myapp

logging:
  level: INFO
  format: "%(asctime)s - %(message)s"

features:
  enable_cache: true
  max_connections: 100
"""

import yaml

def load_config(path: str) -> dict:
    with open(path) as f:
        return yaml.safe_load(f)

config = load_config('config.yaml')
db_host = config['database']['host']

# TOML (Python 3.11+ has built-in tomllib)
import tomllib  # Python 3.11+

with open('config.toml', 'rb') as f:
    config = tomllib.load(f)

# Or use tomli for older Python
# pip install tomli
```

---

## CLI Tools (argparse)

### Basic argparse

```python
import argparse

def main():
    parser = argparse.ArgumentParser(
        description='Process some files.',
        epilog='Example: %(prog)s input.txt -o output.txt'
    )

    # Positional argument
    parser.add_argument('input', help='Input file path')

    # Optional arguments
    parser.add_argument('-o', '--output', help='Output file path')
    parser.add_argument('-v', '--verbose', action='store_true',
                       help='Enable verbose output')
    parser.add_argument('-n', '--number', type=int, default=10,
                       help='Number of items (default: 10)')

    args = parser.parse_args()

    print(f"Input: {args.input}")
    print(f"Output: {args.output}")
    print(f"Verbose: {args.verbose}")
    print(f"Number: {args.number}")

if __name__ == '__main__':
    main()
```

### Advanced argparse

```python
import argparse

def main():
    parser = argparse.ArgumentParser()

    # Mutually exclusive options
    group = parser.add_mutually_exclusive_group()
    group.add_argument('-v', '--verbose', action='store_true')
    group.add_argument('-q', '--quiet', action='store_true')

    # Choices
    parser.add_argument('--format', choices=['json', 'csv', 'xml'],
                       default='json')

    # Multiple values
    parser.add_argument('--files', nargs='+', help='Files to process')
    parser.add_argument('--coords', nargs=2, type=float,
                       metavar=('X', 'Y'), help='Coordinates')

    # Count occurrences
    parser.add_argument('-d', '--debug', action='count', default=0,
                       help='Debug level (-d, -dd, -ddd)')

    # Subcommands
    subparsers = parser.add_subparsers(dest='command')

    # 'create' subcommand
    create_parser = subparsers.add_parser('create', help='Create item')
    create_parser.add_argument('name', help='Item name')

    # 'delete' subcommand
    delete_parser = subparsers.add_parser('delete', help='Delete item')
    delete_parser.add_argument('id', type=int, help='Item ID')

    args = parser.parse_args()

    if args.command == 'create':
        print(f"Creating: {args.name}")
    elif args.command == 'delete':
        print(f"Deleting ID: {args.id}")

if __name__ == '__main__':
    main()
```

### Click (Modern Alternative)

```python
# pip install click
import click

@click.command()
@click.argument('input_file', type=click.Path(exists=True))
@click.option('-o', '--output', type=click.Path(), help='Output file')
@click.option('-v', '--verbose', is_flag=True, help='Verbose mode')
@click.option('-n', '--number', default=10, type=int, help='Number of items')
def process(input_file, output, verbose, number):
    """Process INPUT_FILE and optionally save to OUTPUT."""
    click.echo(f"Processing {input_file}")
    if verbose:
        click.echo(f"Verbose mode enabled")

if __name__ == '__main__':
    process()

# With subcommands
@click.group()
def cli():
    """My CLI application."""
    pass

@cli.command()
@click.argument('name')
def create(name):
    """Create a new item."""
    click.echo(f"Creating {name}")

@cli.command()
@click.argument('id', type=int)
@click.confirmation_option(prompt='Are you sure?')
def delete(id):
    """Delete an item."""
    click.echo(f"Deleting {id}")

if __name__ == '__main__':
    cli()
```

---

## Debugging

### pdb - Python Debugger

```python
# Method 1: Set breakpoint in code
def buggy_function(x):
    result = []
    for i in range(x):
        import pdb; pdb.set_trace()  # Breakpoint
        result.append(i * 2)
    return result

# Method 2: Python 3.7+ built-in
def buggy_function(x):
    result = []
    for i in range(x):
        breakpoint()  # Same as pdb.set_trace()
        result.append(i * 2)
    return result

# Method 3: Run script with debugger
# python -m pdb script.py

# pdb commands:
# n (next)     - Execute next line
# s (step)     - Step into function
# c (continue) - Continue to next breakpoint
# l (list)     - Show current code
# p expr       - Print expression
# pp expr      - Pretty print
# w (where)    - Show stack trace
# u (up)       - Move up stack frame
# d (down)     - Move down stack frame
# b line       - Set breakpoint at line
# b func       - Set breakpoint at function
# cl           - Clear breakpoints
# q (quit)     - Quit debugger
# h (help)     - Show help

# Post-mortem debugging
import pdb

try:
    buggy_code()
except Exception:
    pdb.post_mortem()  # Debug at point of exception
```

### Debugging with logging

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def complex_function(data):
    logger.debug(f"Input data: {data}")

    processed = transform(data)
    logger.debug(f"After transform: {processed}")

    result = calculate(processed)
    logger.info(f"Final result: {result}")

    return result

# Debug decorator
import functools

def debug(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        logger.debug(f"Calling {func.__name__}")
        logger.debug(f"  args: {args}")
        logger.debug(f"  kwargs: {kwargs}")
        result = func(*args, **kwargs)
        logger.debug(f"  returned: {result}")
        return result
    return wrapper

@debug
def my_function(x, y):
    return x + y
```

### Profiling

```python
import cProfile
import pstats
import io

# Profile a function
def profile_function(func, *args, **kwargs):
    profiler = cProfile.Profile()
    profiler.enable()
    result = func(*args, **kwargs)
    profiler.disable()

    # Print stats
    stream = io.StringIO()
    stats = pstats.Stats(profiler, stream=stream)
    stats.sort_stats('cumulative')
    stats.print_stats(20)  # Top 20
    print(stream.getvalue())

    return result

# Or use as context manager
from contextlib import contextmanager

@contextmanager
def profile():
    profiler = cProfile.Profile()
    profiler.enable()
    try:
        yield
    finally:
        profiler.disable()
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)

# Usage
with profile():
    expensive_operation()

# Command line
# python -m cProfile -s cumulative script.py

# timeit for micro-benchmarks
import timeit

# Time a statement
timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)

# Time a function
def my_function():
    return sum(range(1000))

timeit.timeit(my_function, number=10000)
```

### Memory Profiling

```python
# pip install memory_profiler

from memory_profiler import profile

@profile
def memory_hungry():
    a = [1] * (10 ** 6)
    b = [2] * (2 * 10 ** 7)
    del b
    return a

# Run with: python -m memory_profiler script.py

# Or use tracemalloc (built-in)
import tracemalloc

tracemalloc.start()

# ... your code ...

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')

print("[ Top 10 memory consumers ]")
for stat in top_stats[:10]:
    print(stat)
```

---

## Error Recovery Patterns

### Retry with Backoff

```python
import time
import random
from functools import wraps

def retry(max_attempts=3, backoff_factor=2, exceptions=(Exception,)):
    """Decorator for retrying with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 0
            while attempt < max_attempts:
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    attempt += 1
                    if attempt >= max_attempts:
                        raise
                    sleep_time = backoff_factor ** attempt + random.random()
                    print(f"Attempt {attempt} failed: {e}. Retrying in {sleep_time:.2f}s")
                    time.sleep(sleep_time)
        return wrapper
    return decorator

@retry(max_attempts=3, exceptions=(ConnectionError, TimeoutError))
def fetch_data(url):
    # May fail temporarily
    return requests.get(url)

# Using tenacity library (more features)
# pip install tenacity
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=10))
def fetch_data(url):
    return requests.get(url)
```

### Circuit Breaker

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing if service recovered

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        with self.lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise Exception("Circuit is OPEN")

        try:
            result = func(*args, **kwargs)
            with self.lock:
                self.failures = 0
                self.state = CircuitState.CLOSED
            return result
        except Exception as e:
            with self.lock:
                self.failures += 1
                self.last_failure_time = time.time()
                if self.failures >= self.failure_threshold:
                    self.state = CircuitState.OPEN
            raise

# Usage
breaker = CircuitBreaker()

def risky_call():
    return external_service()

try:
    result = breaker.call(risky_call)
except Exception as e:
    # Handle failure
    pass
```

### Graceful Degradation

```python
def get_user_data(user_id):
    """Get user data with fallbacks."""
    try:
        # Primary: Database
        return database.get_user(user_id)
    except DatabaseError:
        logger.warning(f"Database unavailable for user {user_id}")

    try:
        # Fallback 1: Cache
        return cache.get(f"user:{user_id}")
    except CacheError:
        logger.warning(f"Cache unavailable for user {user_id}")

    try:
        # Fallback 2: Default data
        return get_default_user_data(user_id)
    except Exception as e:
        logger.error(f"All fallbacks failed: {e}")
        raise

# Feature flags for degradation
class FeatureFlags:
    def __init__(self):
        self.flags = {
            'enable_recommendations': True,
            'enable_analytics': True,
        }

    def is_enabled(self, flag):
        return self.flags.get(flag, False)

flags = FeatureFlags()

def get_recommendations(user_id):
    if not flags.is_enabled('enable_recommendations'):
        return []  # Feature disabled
    return recommendation_service.get(user_id)
```

---

## Interview Questions

```python
# Q1: How do you handle configuration in Python applications?
"""
1. Environment variables (os.environ, python-dotenv)
2. Config files (YAML, TOML, JSON)
3. Pydantic Settings (validation, typing)
4. Never hardcode secrets
5. Separate configs for dev/staging/prod
"""

# Q2: Explain logging best practices
"""
1. Use module-level loggers: logging.getLogger(__name__)
2. Configure once at application start
3. Use appropriate levels (DEBUG, INFO, WARNING, ERROR)
4. Include context in log messages
5. Use structured logging (JSON) for production
6. Rotate log files to prevent disk fill
"""

# Q3: How do you debug production issues?
"""
1. Comprehensive logging with request IDs
2. Error tracking (Sentry, etc.)
3. Metrics and alerting
4. Distributed tracing for microservices
5. Health check endpoints
6. Feature flags for quick rollback
"""

# Q4: What is a circuit breaker pattern?
"""
Prevents cascading failures in distributed systems:
- CLOSED: Normal operation
- OPEN: After N failures, reject requests immediately
- HALF_OPEN: After timeout, allow one request to test
If test succeeds → CLOSED, else → OPEN
"""
```
