# Functions Deep Dive

## Function Basics

### Defining Functions

```python
# Basic function
def greet(name):
    """Docstring: Greet a person by name."""
    return f"Hello, {name}!"

# Call the function
result = greet("Alice")  # "Hello, Alice!"

# Functions are objects
print(greet)             # <function greet at 0x...>
print(type(greet))       # <class 'function'>
print(greet.__name__)    # "greet"
print(greet.__doc__)     # "Docstring: Greet a person by name."

# Assign to variable
say_hello = greet
say_hello("Bob")         # "Hello, Bob!"

# Pass as argument
def apply_function(func, value):
    return func(value)

apply_function(greet, "Charlie")  # "Hello, Charlie!"

# Return from function
def make_greeter(greeting):
    def greeter(name):
        return f"{greeting}, {name}!"
    return greeter

spanish_greeter = make_greeter("Hola")
spanish_greeter("Alice")  # "Hola, Alice!"
```

### Return Values

```python
# No return statement = returns None
def no_return():
    print("Hello")

result = no_return()     # Prints "Hello"
print(result)            # None

# Explicit return
def explicit_none():
    return None

# Multiple return values (actually returns tuple)
def get_stats(numbers):
    return min(numbers), max(numbers), sum(numbers) / len(numbers)

minimum, maximum, average = get_stats([1, 2, 3, 4, 5])

# Early return
def find_first_even(numbers):
    for num in numbers:
        if num % 2 == 0:
            return num
    return None  # Explicit return if not found
```

---

## Parameters and Arguments

### Types of Parameters

```python
# Positional parameters
def greet(first_name, last_name):
    return f"Hello, {first_name} {last_name}"

greet("Alice", "Smith")  # "Hello, Alice Smith"

# Default parameters
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"

greet("Alice")           # "Hello, Alice!"
greet("Alice", "Hi")     # "Hi, Alice!"

# IMPORTANT: Default values evaluated ONCE at function definition
# Mutable default argument gotcha:
def append_to(element, to=[]):  # DON'T DO THIS
    to.append(element)
    return to

append_to(1)  # [1]
append_to(2)  # [1, 2] - Same list!

# FIX: Use None as default
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to

# Keyword arguments
def greet(first_name, last_name, greeting="Hello"):
    return f"{greeting}, {first_name} {last_name}"

greet(first_name="Alice", last_name="Smith")
greet(greeting="Hi", first_name="Bob", last_name="Jones")
greet("Alice", last_name="Smith")  # Positional first, then keyword
# greet(first_name="Alice", "Smith")  # SyntaxError! Positional after keyword
```

### *args and **kwargs

```python
# *args - Variable positional arguments (tuple)
def sum_all(*args):
    print(type(args))    # <class 'tuple'>
    return sum(args)

sum_all(1, 2, 3)         # 6
sum_all(1, 2, 3, 4, 5)   # 15

# **kwargs - Variable keyword arguments (dict)
def print_info(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="Alice", age=30, city="NYC")

# Combining parameters (ORDER MATTERS!)
def func(pos1, pos2, *args, kw1, kw2="default", **kwargs):
    print(f"pos1={pos1}, pos2={pos2}")
    print(f"args={args}")
    print(f"kw1={kw1}, kw2={kw2}")
    print(f"kwargs={kwargs}")

func(1, 2, 3, 4, kw1="a", kw2="b", extra="c")
# pos1=1, pos2=2
# args=(3, 4)
# kw1=a, kw2=b
# kwargs={'extra': 'c'}

# Parameter order:
# 1. Regular positional
# 2. *args
# 3. Keyword-only (after *)
# 4. **kwargs
```

### Keyword-Only and Positional-Only

```python
# Keyword-only arguments (after *)
def greet(name, *, greeting="Hello", punctuation="!"):
    return f"{greeting}, {name}{punctuation}"

greet("Alice")                     # "Hello, Alice!"
greet("Alice", greeting="Hi")      # "Hi, Alice!"
# greet("Alice", "Hi")             # TypeError! greeting is keyword-only

# Positional-only arguments (Python 3.8+, before /)
def greet(name, /, greeting="Hello"):
    return f"{greeting}, {name}!"

greet("Alice")           # OK
greet("Alice", "Hi")     # OK
# greet(name="Alice")    # TypeError! name is positional-only

# Combined
def func(pos_only, /, pos_or_kw, *, kw_only):
    pass

func(1, 2, kw_only=3)           # OK
func(1, pos_or_kw=2, kw_only=3) # OK
# func(pos_only=1, ...)         # Error
# func(1, 2, 3)                  # Error (kw_only must be keyword)
```

### Unpacking Arguments

```python
# Unpack list/tuple with *
def greet(first, last, greeting="Hello"):
    return f"{greeting}, {first} {last}"

names = ["Alice", "Smith"]
greet(*names)            # Same as greet("Alice", "Smith")

# Unpack dict with **
info = {"first": "Alice", "last": "Smith", "greeting": "Hi"}
greet(**info)            # Same as greet(first="Alice", last="Smith", greeting="Hi")

# Combine
args = ["Alice", "Smith"]
kwargs = {"greeting": "Howdy"}
greet(*args, **kwargs)   # "Howdy, Alice Smith"
```

---

## Scope and Namespaces

### LEGB Rule

```python
"""
LEGB Rule - Variable lookup order:
1. Local - Inside current function
2. Enclosing - Inside enclosing function (for nested functions)
3. Global - Module level
4. Built-in - Python built-ins (print, len, etc.)
"""

# Built-in
print  # Found in built-in namespace

# Global
x = "global"

def outer():
    # Enclosing (for inner())
    x = "enclosing"

    def inner():
        # Local
        x = "local"
        print(x)  # "local"

    inner()
    print(x)      # "enclosing"

outer()
print(x)          # "global"
```

### Global and Nonlocal

```python
# global - Modify global variable from inside function
count = 0

def increment():
    global count
    count += 1

increment()
print(count)  # 1

# Without global, this would fail:
def increment_fail():
    count += 1  # UnboundLocalError! count is local (due to assignment)

# nonlocal - Modify enclosing function's variable
def outer():
    count = 0

    def inner():
        nonlocal count
        count += 1

    inner()
    inner()
    return count

outer()  # 2

# Without nonlocal:
def outer_fail():
    count = 0

    def inner():
        count += 1  # Creates new local variable!

    inner()
    return count  # Still 0
```

### Variable Scope Gotchas

```python
# 1. Assignment creates local variable
x = 10
def func():
    print(x)  # UnboundLocalError if there's assignment below!
    x = 20    # This makes x local to entire function

# 2. Fix with explicit access or global
def func_fixed():
    global x
    print(x)
    x = 20

# 3. Mutable objects can be modified without global
data = []
def add_item(item):
    data.append(item)  # OK - modifying, not rebinding

add_item(1)
print(data)  # [1]

# 4. But rebinding requires global
data = []
def replace_data():
    global data
    data = [1, 2, 3]  # Rebinding, needs global
```

---

## Closures

### What is a Closure?

```python
"""
A closure is a function that:
1. Is nested inside another function
2. References variables from the enclosing function
3. Is returned (or passed) out of the enclosing function

The closure "closes over" the enclosing variables, keeping them alive.
"""

def make_multiplier(n):
    # n is in enclosing scope
    def multiplier(x):
        return x * n  # References n from enclosing scope
    return multiplier  # Return the inner function

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)   # 10
triple(5)   # 15

# Each closure has its own copy of n
print(double.__closure__)  # (<cell at 0x...: int object at 0x...>,)
print(double.__closure__[0].cell_contents)  # 2
print(triple.__closure__[0].cell_contents)  # 3
```

### Closure Use Cases

```python
# 1. Function factories
def make_power(exponent):
    def power(base):
        return base ** exponent
    return power

square = make_power(2)
cube = make_power(3)
square(5)  # 25
cube(5)    # 125

# 2. Maintaining state
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

counter = make_counter()
counter()  # 1
counter()  # 2
counter()  # 3

# 3. Callbacks with context
def make_logger(prefix):
    def log(message):
        print(f"[{prefix}] {message}")
    return log

error_log = make_logger("ERROR")
info_log = make_logger("INFO")
error_log("Something went wrong")  # [ERROR] Something went wrong

# 4. Partial application (see also functools.partial)
def make_adder(a):
    def add(b):
        return a + b
    return add

add_5 = make_adder(5)
add_5(3)  # 8
```

### Late Binding Gotcha

```python
# Problem: All closures share the same variable
def make_functions():
    funcs = []
    for i in range(3):
        def func():
            return i  # All reference the same 'i'
        funcs.append(func)
    return funcs

funcs = make_functions()
[f() for f in funcs]  # [2, 2, 2] - All return 2!

# Fix 1: Default argument (evaluated at definition time)
def make_functions_fixed():
    funcs = []
    for i in range(3):
        def func(i=i):  # Capture current value of i
            return i
        funcs.append(func)
    return funcs

funcs = make_functions_fixed()
[f() for f in funcs]  # [0, 1, 2]

# Fix 2: Nested function
def make_functions_fixed2():
    def make_func(i):
        def func():
            return i
        return func
    return [make_func(i) for i in range(3)]

# Fix 3: functools.partial
from functools import partial

def return_val(i):
    return i

funcs = [partial(return_val, i) for i in range(3)]
```

---

## Decorators

### What is a Decorator?

```python
"""
A decorator is a function that:
1. Takes a function as argument
2. Returns a new function (usually wrapping the original)
3. Uses @ syntax for convenience

Decorators modify or enhance functions without changing their code.
"""

# Basic decorator
def my_decorator(func):
    def wrapper():
        print("Before")
        func()
        print("After")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Before
# Hello!
# After

# @ syntax is equivalent to:
def say_hello():
    print("Hello!")
say_hello = my_decorator(say_hello)
```

### Decorator with Arguments

```python
# Handle any function signature with *args, **kwargs
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

add(1, 2)  # Returns 3, with logging
```

### Preserving Function Metadata

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Preserves __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def greet(name):
    """Greet someone."""
    return f"Hello, {name}"

print(greet.__name__)  # "greet" (not "wrapper")
print(greet.__doc__)   # "Greet someone."
```

### Decorator with Parameters

```python
# Decorator that takes arguments requires another layer
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def say_hello():
    print("Hello!")

say_hello()  # Prints "Hello!" 3 times

# How it works:
# repeat(3) returns decorator
# decorator(say_hello) returns wrapper
# say_hello is now wrapper
```

### Common Decorator Patterns

```python
from functools import wraps
import time

# 1. Timing decorator
def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

# 2. Logging decorator
def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        args_repr = [repr(a) for a in args]
        kwargs_repr = [f"{k}={v!r}" for k, v in kwargs.items()]
        signature = ", ".join(args_repr + kwargs_repr)
        print(f"Calling {func.__name__}({signature})")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result!r}")
        return result
    return wrapper

# 3. Retry decorator
def retry(max_attempts=3, delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def unreliable_function():
    import random
    if random.random() < 0.7:
        raise ValueError("Random failure")
    return "Success"

# 4. Caching/Memoization
def memoize(func):
    cache = {}
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Better: use functools.lru_cache
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# 5. Authorization decorator
def require_auth(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        if not is_authenticated():  # Your auth check
            raise PermissionError("Not authenticated")
        return func(*args, **kwargs)
    return wrapper
```

### Stacking Decorators

```python
@decorator1
@decorator2
@decorator3
def func():
    pass

# Equivalent to:
func = decorator1(decorator2(decorator3(func)))

# Order matters! Decorators apply bottom-up, execute top-down
@timer
@log_calls
def my_function():
    pass

# log_calls wraps my_function
# timer wraps the log_calls wrapper
# Execution: timer → log_calls → my_function
```

### Class-Based Decorators

```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hello():
    print("Hello!")

say_hello()  # Call 1 of say_hello
say_hello()  # Call 2 of say_hello
print(say_hello.count)  # 2
```

---

## Lambda Functions

### Lambda Basics

```python
# Lambda = anonymous function (single expression)
add = lambda x, y: x + y
add(2, 3)  # 5

# Equivalent to:
def add(x, y):
    return x + y

# Syntax: lambda arguments: expression
# - No 'def' or 'return' keyword
# - Single expression only (no statements)
# - Implicitly returns the result

# Multiple arguments
multiply = lambda x, y, z: x * y * z

# Default arguments
greet = lambda name, greeting="Hello": f"{greeting}, {name}"

# No arguments
get_pi = lambda: 3.14159
```

### Lambda Use Cases

```python
# 1. Sorting key
people = [("Alice", 30), ("Bob", 25), ("Charlie", 35)]
sorted(people, key=lambda x: x[1])  # Sort by age

# 2. filter/map/reduce
numbers = [1, 2, 3, 4, 5]

# Filter evens
evens = list(filter(lambda x: x % 2 == 0, numbers))  # [2, 4]

# Double each
doubled = list(map(lambda x: x * 2, numbers))  # [2, 4, 6, 8, 10]

# Sum all
from functools import reduce
total = reduce(lambda acc, x: acc + x, numbers)  # 15

# 3. Inline callbacks
button.on_click(lambda event: print(f"Clicked at {event.position}"))

# 4. Conditional expression
classify = lambda x: "positive" if x > 0 else ("negative" if x < 0 else "zero")

# When NOT to use lambda (prefer def):
# - Complex logic
# - Multiple statements
# - Need docstring
# - Reused multiple times
```

---

## functools Module

```python
from functools import partial, reduce, lru_cache, wraps

# partial - Fix some arguments
def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)
square(5)  # 25

# reduce - Reduce iterable to single value
from functools import reduce
numbers = [1, 2, 3, 4, 5]
product = reduce(lambda x, y: x * y, numbers)  # 120

# lru_cache - Memoization
@lru_cache(maxsize=128)
def expensive_func(n):
    print(f"Computing {n}")
    return n ** 2

expensive_func(4)  # Computes
expensive_func(4)  # Returns cached value

expensive_func.cache_info()  # CacheInfo(hits=1, misses=1, ...)
expensive_func.cache_clear()

# cache (Python 3.9+) - Same as lru_cache(maxsize=None)
from functools import cache

@cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

---

## Interview Questions

### Common Questions

```python
# Q1: What's the output?
def func(a, b=[]):
    b.append(a)
    return b

print(func(1))  # [1]
print(func(2))  # [1, 2] - Same list!
print(func(3))  # [1, 2, 3]

# Q2: Fix the closure issue
funcs = [lambda x: x * i for i in range(3)]
# All return x * 2!

# Fix:
funcs = [lambda x, i=i: x * i for i in range(3)]

# Q3: Explain this decorator
def decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# Q4: Write a decorator that caches results
def memoize(func):
    cache = {}
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

# Q5: What's the difference between @staticmethod and @classmethod?
class MyClass:
    @staticmethod
    def static_method():
        pass  # No access to class or instance

    @classmethod
    def class_method(cls):
        pass  # Access to class, not instance
```

### Performance Considerations

```python
# Lambda vs def - No performance difference
# But named functions are easier to debug (better stack traces)

# Function call overhead
# Python function calls are expensive (~100-200ns)
# Avoid calling functions in tight loops if possible

# Prefer built-ins when possible
# sum(list) is faster than reduce(lambda x,y: x+y, list)
# max/min are faster than sorting

# Use lru_cache for expensive computations
# But remember: cache can grow large, use maxsize wisely
```
