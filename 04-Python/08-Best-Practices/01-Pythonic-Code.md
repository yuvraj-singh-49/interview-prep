# Pythonic Patterns and Best Practices

## The Zen of Python

```python
import this
"""
Beautiful is better than ugly.
Explicit is better than implicit.
Simple is better than complex.
Complex is better than complicated.
Flat is better than nested.
Sparse is better than dense.
Readability counts.
Special cases aren't special enough to break the rules.
Although practicality beats purity.
Errors should never pass silently.
Unless explicitly silenced.
In the face of ambiguity, refuse the temptation to guess.
There should be one-- and preferably only one --obvious way to do it.
Now is better than never.
Although never is often better than *right* now.
If the implementation is hard to explain, it's a bad idea.
If the implementation is easy to explain, it may be a good idea.
Namespaces are one honking great idea -- let's do more of those!
"""
```

---

## EAFP vs LBYL

### EAFP (Easier to Ask Forgiveness than Permission)

```python
# Pythonic: Try and handle exception
# Preferred in Python

def get_value(data, key):
    try:
        return data[key]
    except KeyError:
        return None

def open_file(filename):
    try:
        with open(filename) as f:
            return f.read()
    except FileNotFoundError:
        return None

def convert_to_int(value):
    try:
        return int(value)
    except (ValueError, TypeError):
        return 0
```

### LBYL (Look Before You Leap)

```python
# Check condition before action
# Common in other languages

def get_value(data, key):
    if key in data:
        return data[key]
    return None

def open_file(filename):
    import os
    if os.path.exists(filename):
        with open(filename) as f:
            return f.read()
    return None

def convert_to_int(value):
    if isinstance(value, str) and value.isdigit():
        return int(value)
    return 0
```

### When to Use Which

```python
"""
EAFP is preferred in Python when:
- Exception is unlikely (common case succeeds)
- Check is expensive (e.g., file exists vs. try to open)
- Race conditions possible (file deleted between check and use)

LBYL is acceptable when:
- Check is cheap and clear
- Exception would be expensive
- Better communicates intent
"""

# EAFP: File may be deleted between check and open
# This is the Python way
try:
    with open(filename) as f:
        return f.read()
except FileNotFoundError:
    return None

# LBYL: Simple attribute check
# Both are acceptable here
if hasattr(obj, 'method'):
    obj.method()

# vs
try:
    obj.method()
except AttributeError:
    pass
```

---

## Common Pythonic Patterns

### Iteration Patterns

```python
# Bad: Don't iterate with range(len())
for i in range(len(items)):
    print(items[i])

# Good: Iterate directly
for item in items:
    print(item)

# Need index? Use enumerate
for i, item in enumerate(items):
    print(f"{i}: {item}")

# Iterate multiple sequences? Use zip
for name, age in zip(names, ages):
    print(f"{name} is {age}")

# Iterate in reverse
for item in reversed(items):
    print(item)

# Iterate sorted
for item in sorted(items):
    print(item)

# Iterate dict items
for key, value in dictionary.items():
    print(f"{key}: {value}")
```

### String Operations

```python
# Bad: String concatenation in loop
result = ""
for item in items:
    result += str(item)

# Good: Use join
result = "".join(str(item) for item in items)

# Bad: Format with +
message = "Hello, " + name + "! You are " + str(age) + " years old."

# Good: f-strings
message = f"Hello, {name}! You are {age} years old."

# Multiline strings
# Bad
query = "SELECT * " + \
        "FROM users " + \
        "WHERE active = 1"

# Good
query = """
SELECT *
FROM users
WHERE active = 1
"""

# Or
query = (
    "SELECT * "
    "FROM users "
    "WHERE active = 1"
)
```

### Collection Operations

```python
# Check if collection is empty
# Bad
if len(items) == 0:
    pass

# Good (empty collections are falsy)
if not items:
    pass

# Check if not empty
if items:
    pass

# Get item or default
# Bad
if key in dictionary:
    value = dictionary[key]
else:
    value = default

# Good
value = dictionary.get(key, default)

# Set default and get
value = dictionary.setdefault(key, [])
value.append(item)

# Merge dicts (Python 3.9+)
merged = dict1 | dict2

# Before 3.9
merged = {**dict1, **dict2}

# Remove duplicates preserving order
unique = list(dict.fromkeys(items))
```

### Unpacking

```python
# Swap variables
a, b = b, a

# Multiple assignment
x, y, z = 1, 2, 3

# Extended unpacking
first, *rest = [1, 2, 3, 4, 5]
first, *middle, last = [1, 2, 3, 4, 5]
*initial, last = [1, 2, 3, 4, 5]

# Ignore values
_, second, _ = (1, 2, 3)

# Unpack in function calls
args = [1, 2, 3]
func(*args)  # func(1, 2, 3)

kwargs = {"a": 1, "b": 2}
func(**kwargs)  # func(a=1, b=2)
```

### Conditional Expressions

```python
# Ternary operator
result = value if condition else default

# Default value
value = x or default  # If x is falsy, use default
value = x if x is not None else default  # Only if None

# Multiple conditions
# Bad
if condition1:
    result = value1
elif condition2:
    result = value2
else:
    result = default

# Good (if simple)
result = (
    value1 if condition1 else
    value2 if condition2 else
    default
)

# Or use dict mapping
result = {
    "case1": value1,
    "case2": value2,
}.get(key, default)
```

---

## Clean Code Patterns

### Function Design

```python
# Single responsibility
# Bad: Does too many things
def process_user(user_data):
    # Validate
    # Transform
    # Save to DB
    # Send email
    pass

# Good: Split into focused functions
def validate_user(user_data):
    pass

def transform_user(user_data):
    pass

def save_user(user):
    pass

def notify_user(user):
    pass

# Avoid boolean parameters
# Bad
def fetch_data(include_metadata=False):
    pass

# Good
def fetch_data():
    pass

def fetch_data_with_metadata():
    pass

# Or use explicit names
def fetch_data(fields=None):
    """
    fields: List of fields to include, e.g., ['id', 'name', 'metadata']
    """
    pass

# Return early
# Bad
def process(data):
    if data:
        # Many lines of processing
        # ...
        # ...
        return result
    else:
        return None

# Good
def process(data):
    if not data:
        return None

    # Processing here
    return result
```

### Avoid Magic Values

```python
# Bad
def calculate_discount(price, customer_type):
    if customer_type == 1:
        return price * 0.9
    elif customer_type == 2:
        return price * 0.8
    return price

# Good: Use constants or enums
from enum import Enum

class CustomerType(Enum):
    REGULAR = "regular"
    PREMIUM = "premium"
    VIP = "vip"

DISCOUNT_RATES = {
    CustomerType.REGULAR: 0.95,
    CustomerType.PREMIUM: 0.90,
    CustomerType.VIP: 0.80,
}

def calculate_discount(price, customer_type):
    rate = DISCOUNT_RATES.get(customer_type, 1.0)
    return price * rate
```

### Context Managers

```python
# Always use with for resources
# Bad
f = open('file.txt')
try:
    data = f.read()
finally:
    f.close()

# Good
with open('file.txt') as f:
    data = f.read()

# Multiple context managers
with open('input.txt') as fin, open('output.txt', 'w') as fout:
    fout.write(fin.read())

# Custom context manager
from contextlib import contextmanager

@contextmanager
def timing(label):
    start = time.time()
    try:
        yield
    finally:
        print(f"{label}: {time.time() - start:.2f}s")

with timing("Database query"):
    results = db.query(...)
```

---

## Design Patterns in Python

### Singleton

```python
# Method 1: Module-level (recommended)
# mysingleton.py
_instance = None

def get_instance():
    global _instance
    if _instance is None:
        _instance = MyClass()
    return _instance

# Method 2: Decorator
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Database:
    pass

# Method 3: Metaclass
class Singleton(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=Singleton):
    pass
```

### Factory

```python
# Simple factory
class Animal:
    pass

class Dog(Animal):
    def speak(self):
        return "Woof!"

class Cat(Animal):
    def speak(self):
        return "Meow!"

def create_animal(animal_type):
    animals = {
        "dog": Dog,
        "cat": Cat,
    }
    return animals.get(animal_type, Animal)()

# Factory with registration
class AnimalFactory:
    _animals = {}

    @classmethod
    def register(cls, name):
        def wrapper(animal_cls):
            cls._animals[name] = animal_cls
            return animal_cls
        return wrapper

    @classmethod
    def create(cls, name):
        return cls._animals[name]()

@AnimalFactory.register("dog")
class Dog:
    pass

@AnimalFactory.register("cat")
class Cat:
    pass
```

### Strategy

```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount):
        pass

class CreditCardPayment(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} with credit card"

class PayPalPayment(PaymentStrategy):
    def pay(self, amount):
        return f"Paid ${amount} with PayPal"

class ShoppingCart:
    def __init__(self, payment_strategy: PaymentStrategy):
        self.payment_strategy = payment_strategy
        self.items = []

    def checkout(self):
        total = sum(item.price for item in self.items)
        return self.payment_strategy.pay(total)

# Usage
cart = ShoppingCart(CreditCardPayment())
cart.checkout()

# Change strategy at runtime
cart.payment_strategy = PayPalPayment()
cart.checkout()
```

### Observer (using events)

```python
class Event:
    def __init__(self):
        self._handlers = []

    def subscribe(self, handler):
        self._handlers.append(handler)

    def unsubscribe(self, handler):
        self._handlers.remove(handler)

    def emit(self, *args, **kwargs):
        for handler in self._handlers:
            handler(*args, **kwargs)

class User:
    def __init__(self):
        self.on_login = Event()
        self.on_logout = Event()

    def login(self):
        print("User logged in")
        self.on_login.emit(self)

# Usage
user = User()
user.on_login.subscribe(lambda u: print(f"Welcome!"))
user.on_login.subscribe(lambda u: log_analytics("login"))
user.login()
```

---

## Type Hints

### Basic Type Hints

```python
from typing import List, Dict, Tuple, Set, Optional, Union

# Variable annotations
name: str = "Alice"
age: int = 30
active: bool = True

# Function annotations
def greet(name: str) -> str:
    return f"Hello, {name}"

# Complex types
def process(items: List[int]) -> Dict[str, int]:
    return {"sum": sum(items), "count": len(items)}

# Optional (can be None)
def find_user(id: int) -> Optional[User]:
    # Returns User or None
    pass

# Union (multiple types)
def parse(value: Union[str, int]) -> int:
    return int(value)

# Python 3.10+: Use | for Union
def parse(value: str | int) -> int:
    return int(value)

# Python 3.9+: Use built-in generics
def process(items: list[int]) -> dict[str, int]:
    pass
```

### Advanced Type Hints

```python
from typing import (
    Callable, TypeVar, Generic, Protocol,
    Literal, Final, TypedDict, Annotated
)

# Callable
Handler = Callable[[int, str], bool]

def register(handler: Handler) -> None:
    pass

# TypeVar (generic)
T = TypeVar('T')

def first(items: List[T]) -> T:
    return items[0]

# Generic class
class Stack(Generic[T]):
    def __init__(self):
        self._items: List[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

# Protocol (structural subtyping)
class Drawable(Protocol):
    def draw(self) -> None: ...

def render(item: Drawable) -> None:
    item.draw()

# Any class with draw() method works
class Circle:
    def draw(self) -> None:
        print("Drawing circle")

render(Circle())  # OK

# Literal
Mode = Literal["r", "w", "a"]

def open_file(path: str, mode: Mode) -> None:
    pass

# Final
MAX_SIZE: Final = 100

# TypedDict
class Point(TypedDict):
    x: int
    y: int

def plot(point: Point) -> None:
    print(point["x"], point["y"])
```

---

## Code Quality Tools

### linters and Formatters

```bash
# Black - Code formatter
pip install black
black my_file.py
black --check .  # Check only

# isort - Import sorter
pip install isort
isort my_file.py

# flake8 - Linter
pip install flake8
flake8 my_file.py

# pylint - Comprehensive linter
pip install pylint
pylint my_file.py

# mypy - Type checker
pip install mypy
mypy my_file.py

# Combine with pre-commit
# .pre-commit-config.yaml
# repos:
#   - repo: https://github.com/psf/black
#     rev: 23.1.0
#     hooks:
#       - id: black
#   - repo: https://github.com/pycqa/isort
#     rev: 5.12.0
#     hooks:
#       - id: isort
```

### pyproject.toml Configuration

```toml
[tool.black]
line-length = 88
target-version = ['py310']

[tool.isort]
profile = "black"
line_length = 88

[tool.mypy]
python_version = "3.10"
strict = true
ignore_missing_imports = true

[tool.pylint.messages_control]
disable = ["C0114", "C0115", "C0116"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --cov=src"
```

---

## Performance Tips

```python
# 1. Use built-in functions (implemented in C)
sum(items)          # vs manual loop
max(items)          # vs manual comparison
any(x > 0 for x in items)
all(x > 0 for x in items)

# 2. List comprehensions over loops
# Faster (C implementation)
squares = [x**2 for x in range(1000)]

# Slower (Python loop)
squares = []
for x in range(1000):
    squares.append(x**2)

# 3. Use generators for large data
# Memory efficient
sum(x**2 for x in range(10000000))

# Memory heavy
sum([x**2 for x in range(10000000)])

# 4. Local variables are faster
def slow():
    for _ in range(1000000):
        len([1,2,3])  # Global lookup

def fast():
    _len = len  # Local reference
    for _ in range(1000000):
        _len([1,2,3])

# 5. Use appropriate data structures
# O(1) lookup
if x in my_set:  # set
    pass

# O(n) lookup
if x in my_list:  # list
    pass

# 6. String joining
# O(n)
''.join(strings)

# O(n²)
result = ''
for s in strings:
    result += s

# 7. Use collections.deque for queues
from collections import deque
q = deque()
q.append(1)     # O(1)
q.popleft()     # O(1) vs list.pop(0) is O(n)

# 8. __slots__ for many instances
class Point:
    __slots__ = ['x', 'y']  # Saves memory
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

---

## Interview Checklist

```python
"""
Python Interview Preparation Checklist:

Fundamentals:
☐ Variables, data types, operators
☐ Control flow (if/elif/else, loops, match)
☐ String formatting (f-strings)
☐ Data structures (list, dict, set, tuple)
☐ Comprehensions (list, dict, set, generator)

Functions:
☐ *args, **kwargs
☐ Scope (LEGB rule)
☐ Closures
☐ Decorators
☐ Lambda functions
☐ functools (partial, lru_cache)

OOP:
☐ Classes and instances
☐ Class vs instance attributes
☐ @staticmethod vs @classmethod
☐ Inheritance and MRO
☐ Magic methods (__init__, __repr__, __eq__, etc.)
☐ Properties and descriptors
☐ Abstract base classes
☐ Dataclasses

Iterators/Generators:
☐ Iterator protocol (__iter__, __next__)
☐ Generator functions (yield)
☐ Generator expressions
☐ itertools module

Concurrency:
☐ GIL and its implications
☐ threading module
☐ multiprocessing module
☐ asyncio and async/await
☐ When to use which

Internals:
☐ Everything is an object
☐ Reference counting
☐ Garbage collection
☐ Memory optimization (__slots__)

Error Handling:
☐ try/except/else/finally
☐ Custom exceptions
☐ Context managers

Testing:
☐ unittest basics
☐ pytest fixtures and markers
☐ Mocking

Best Practices:
☐ EAFP vs LBYL
☐ PEP 8 style guide
☐ Type hints
☐ Common design patterns
"""
```
