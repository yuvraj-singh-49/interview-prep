# Python Internals

## Everything is an Object

### Object Structure

```python
"""
In Python, EVERYTHING is an object:
- Numbers, strings, functions, classes, modules
- Even type itself is an object

Every object has:
- Identity (id) - memory address
- Type - determines behavior
- Value - the actual data
"""

x = 42
id(x)           # Memory address (e.g., 140732856454064)
type(x)         # <class 'int'>
x               # 42 (value)

# Functions are objects
def greet():
    pass

type(greet)     # <class 'function'>
greet.__name__  # 'greet'
greet.__code__  # Code object with bytecode

# Classes are objects too
class MyClass:
    pass

type(MyClass)   # <class 'type'>
type(type)      # <class 'type'> (type is its own type!)

# The type hierarchy
#   object → base of everything
#   type → creates classes
#   type(type) → type (metaclass)
#   type(object) → type
#   object.__bases__ → () (empty tuple)
#   type.__bases__ → (object,)
```

### PyObject Structure (C Level)

```
Every Python object in CPython has this C structure:

┌─────────────────────────────────────────┐
│ PyObject                                │
├─────────────────────────────────────────┤
│ Py_ssize_t ob_refcnt  │ Reference count │
├─────────────────────────────────────────┤
│ PyTypeObject *ob_type │ Pointer to type │
├─────────────────────────────────────────┤
│ ... additional data ... │ Type-specific  │
└─────────────────────────────────────────┘

For variable-size objects (list, str, tuple):
┌─────────────────────────────────────────┐
│ PyVarObject                             │
├─────────────────────────────────────────┤
│ Py_ssize_t ob_refcnt  │ Reference count │
├─────────────────────────────────────────┤
│ PyTypeObject *ob_type │ Pointer to type │
├─────────────────────────────────────────┤
│ Py_ssize_t ob_size    │ Number of items │
├─────────────────────────────────────────┤
│ ... items ...         │ Actual data     │
└─────────────────────────────────────────┘
```

---

## Memory Management

### Reference Counting

```python
"""
Python primarily uses reference counting for memory management.
Each object tracks how many references point to it.
When count drops to 0, object is immediately deallocated.
"""

import sys

# Check reference count
a = [1, 2, 3]
sys.getrefcount(a)  # Returns count + 1 (for the argument itself)

# Reference counting in action
x = [1, 2, 3]       # refcount = 1
y = x               # refcount = 2 (same object)
z = x               # refcount = 3

del y               # refcount = 2
z = None            # refcount = 1
del x               # refcount = 0 → deallocated

# Circular references problem
class Node:
    def __init__(self):
        self.ref = None

a = Node()
b = Node()
a.ref = b           # a references b
b.ref = a           # b references a
del a, b            # Both have refcount 1 (from each other)
                    # Never reaches 0 → memory leak!
                    # This is why Python also has garbage collector
```

### Garbage Collection

```python
"""
Python's garbage collector handles:
1. Circular references (reference counting can't handle)
2. Generational collection (most objects die young)
"""

import gc

# Generational GC
# Generation 0: New objects (collected frequently)
# Generation 1: Survived one collection
# Generation 2: Long-lived objects (collected rarely)

gc.get_threshold()  # (700, 10, 10)
# Collect gen 0 after 700 allocations
# Collect gen 1 every 10 gen 0 collections
# Collect gen 2 every 10 gen 1 collections

# Manual garbage collection
gc.collect()        # Run full collection
gc.collect(0)       # Collect generation 0 only

# Disable/enable GC
gc.disable()
gc.enable()
gc.isenabled()

# Get unreachable objects
gc.garbage          # List of uncollectable objects (with __del__)

# Debug flags
gc.set_debug(gc.DEBUG_LEAK)  # Print info about leaks

# Weak references (don't increase refcount)
import weakref

class MyClass:
    pass

obj = MyClass()
ref = weakref.ref(obj)      # Weak reference
ref()                        # Returns obj (or None if collected)

del obj                      # Object can be collected
ref()                        # None
```

### Memory Optimization

```python
import sys

# Object sizes
sys.getsizeof(1)            # ~28 bytes (small int)
sys.getsizeof([])           # ~56 bytes (empty list)
sys.getsizeof({})           # ~64 bytes (empty dict)
sys.getsizeof("")           # ~49 bytes (empty string)

# __slots__ - reduce memory for many instances
class WithoutSlots:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ['x', 'y']
    def __init__(self, x, y):
        self.x = x
        self.y = y

# Without slots: each instance has __dict__ (~100+ bytes overhead)
# With slots: no __dict__, attributes stored in slots (~16 bytes per slot)

# Interning - reuse small integers and strings
a = 256
b = 256
a is b              # True (cached)

a = 257
b = 257
a is b              # False (or True in some contexts - implementation detail)

# String interning
a = "hello"
b = "hello"
a is b              # True (interned)

a = "hello world!"
b = "hello world!"
a is b              # May be False (spaces/special chars)

# Force interning
import sys
a = sys.intern("hello world!")
b = sys.intern("hello world!")
a is b              # True
```

---

## The Global Interpreter Lock (GIL)

### What is the GIL?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Global Interpreter Lock                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  GIL = Mutex that protects access to Python objects                        │
│                                                                             │
│  Only ONE thread can execute Python bytecode at a time                     │
│                                                                             │
│  Thread 1: ┌─Execute─┐ ┌───────Wait───────┐ ┌─Execute─┐                   │
│  Thread 2: └──Wait───┘ └──────Execute─────┘ └──Wait───┘                   │
│            └─────────────────────────────────────────────┘                 │
│                         Time →                                              │
│                                                                             │
│  WHY?                                                                       │
│  - CPython's memory management uses reference counting                      │
│  - Reference count operations must be atomic                               │
│  - GIL is simpler than fine-grained locking                               │
│  - Many C extensions depend on GIL                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### GIL Behavior

```python
import threading
import time

# GIL is released during:
# 1. I/O operations (file, network, sleep)
# 2. Certain C extension operations
# 3. Every N bytecode instructions (sys.getswitchinterval())

import sys
sys.getswitchinterval()     # Default: 0.005 seconds (5ms)
sys.setswitchinterval(0.01) # Change switch interval

# CPU-bound: GIL kills parallelism
def cpu_work(n):
    for _ in range(n):
        pass

# Threading won't help (may be slower due to GIL contention)
# I/O-bound: GIL is released, threading helps
def io_work():
    time.sleep(1)  # GIL released during sleep

# Solutions for CPU-bound:
# 1. multiprocessing (separate processes, separate GILs)
# 2. C extensions that release GIL
# 3. Cython with nogil
# 4. Use NumPy/Pandas (release GIL for computations)
```

### Checking for GIL

```python
# NumPy releases GIL during computations
import numpy as np
import threading

def numpy_work():
    arr = np.random.random(10000000)
    return np.sum(arr)  # GIL released during NumPy operations

# This WILL benefit from threading
threads = [threading.Thread(target=numpy_work) for _ in range(4)]
# Runs faster than sequential (NumPy releases GIL)
```

---

## Python Bytecode

### Understanding Bytecode

```python
"""
Python compiles source code to bytecode (.pyc files)
Bytecode is executed by the Python Virtual Machine (PVM)
"""

def add(a, b):
    return a + b

# View bytecode
import dis
dis.dis(add)
# Output:
#   2           0 LOAD_FAST                0 (a)
#               2 LOAD_FAST                1 (b)
#               4 BINARY_ADD
#               6 RETURN_VALUE

# Code object
add.__code__
add.__code__.co_code        # Raw bytecode bytes
add.__code__.co_consts      # Constants used
add.__code__.co_varnames    # Local variable names
add.__code__.co_stacksize   # Maximum stack size

# Compile to AST
import ast
tree = ast.parse("x = 1 + 2")
ast.dump(tree)

# Compile source to code object
code = compile("x = 1 + 2", "<string>", "exec")
exec(code)
print(x)  # 3
```

### Common Bytecode Instructions

```python
"""
LOAD_FAST     - Load local variable onto stack
LOAD_GLOBAL   - Load global variable
LOAD_CONST    - Load constant
STORE_FAST    - Store top of stack to local
STORE_GLOBAL  - Store to global
BINARY_ADD    - Pop two values, push sum
CALL_FUNCTION - Call function with N arguments
RETURN_VALUE  - Return top of stack
JUMP_IF_TRUE  - Conditional jump
POP_TOP       - Discard top of stack
"""

# Example: Understanding loops
import dis

def loop_example():
    total = 0
    for i in range(10):
        total += i
    return total

dis.dis(loop_example)
# Shows: GET_ITER, FOR_ITER, INPLACE_ADD, JUMP_BACKWARD, etc.
```

---

## Attribute Access

### Attribute Lookup Order

```python
"""
When you access obj.attr, Python searches in this order:

1. Data descriptors from type(obj).__mro__
2. Instance __dict__
3. Non-data descriptors and class attributes from type(obj).__mro__
4. __getattr__ (if defined)
"""

class Descriptor:
    def __get__(self, obj, type=None):
        return "from descriptor"
    def __set__(self, obj, value):
        pass  # Data descriptor (has __set__)

class NonDataDescriptor:
    def __get__(self, obj, type=None):
        return "from non-data descriptor"
    # No __set__ = non-data descriptor

class MyClass:
    data_desc = Descriptor()
    non_data_desc = NonDataDescriptor()
    class_attr = "class attribute"

    def __init__(self):
        self.__dict__['data_desc'] = "instance"
        self.__dict__['non_data_desc'] = "instance"
        self.class_attr = "instance"  # Shadows class attr

obj = MyClass()
obj.data_desc       # "from descriptor" (data descriptor wins)
obj.non_data_desc   # "instance" (instance wins over non-data)
obj.class_attr      # "instance" (instance shadows class)
```

### __getattr__ vs __getattribute__

```python
class MyClass:
    def __init__(self):
        self.existing = "value"

    def __getattribute__(self, name):
        """Called for EVERY attribute access"""
        print(f"Getting {name}")
        return super().__getattribute__(name)

    def __getattr__(self, name):
        """Called only when attribute not found normally"""
        return f"{name} not found, returning default"

obj = MyClass()
obj.existing        # Prints "Getting existing", returns "value"
obj.missing         # Prints "Getting missing", then "missing not found..."

# __getattribute__ gotcha: infinite recursion
class Broken:
    def __getattribute__(self, name):
        return self.name  # Calls __getattribute__ again!

# Fix: Use super() or object.__getattribute__
class Fixed:
    def __getattribute__(self, name):
        return object.__getattribute__(self, name)
```

---

## Metaclasses

### What are Metaclasses?

```python
"""
Classes are objects created by metaclasses.
type is the default metaclass.
type(class) → its metaclass

   type
     │
     └── creates → MyClass (class)
                      │
                      └── creates → instance (object)
"""

# Classes are instances of type
class MyClass:
    pass

type(MyClass)       # <class 'type'>
isinstance(MyClass, type)  # True

# Creating class dynamically with type()
MyClass = type('MyClass', (object,), {'x': 1, 'method': lambda self: 42})
# type(name, bases, dict)

obj = MyClass()
obj.x               # 1
obj.method()        # 42
```

### Custom Metaclass

```python
class MyMeta(type):
    """Custom metaclass"""

    def __new__(mcs, name, bases, namespace):
        """Called when class is created"""
        print(f"Creating class: {name}")
        # Modify namespace before class creation
        namespace['added_by_meta'] = True
        return super().__new__(mcs, name, bases, namespace)

    def __init__(cls, name, bases, namespace):
        """Called after class is created"""
        print(f"Initializing class: {name}")
        super().__init__(name, bases, namespace)

    def __call__(cls, *args, **kwargs):
        """Called when class is instantiated"""
        print(f"Creating instance of {cls.__name__}")
        return super().__call__(*args, **kwargs)

class MyClass(metaclass=MyMeta):
    pass
# Output:
# Creating class: MyClass
# Initializing class: MyClass

obj = MyClass()
# Output:
# Creating instance of MyClass

MyClass.added_by_meta  # True
```

### Metaclass Use Cases

```python
# 1. Singleton pattern
class Singleton(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class MyClass(metaclass=Singleton):
    pass

a = MyClass()
b = MyClass()
a is b  # True

# 2. Class registration
registry = {}

class RegisterMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if name != 'Base':
            registry[name] = cls
        return cls

class Base(metaclass=RegisterMeta):
    pass

class Plugin1(Base):
    pass

class Plugin2(Base):
    pass

print(registry)  # {'Plugin1': <class 'Plugin1'>, 'Plugin2': <class 'Plugin2'>}

# 3. ABC uses metaclass
from abc import ABCMeta, abstractmethod

class Interface(metaclass=ABCMeta):
    @abstractmethod
    def method(self):
        pass
```

---

## Performance Optimization

### Profiling

```python
import cProfile
import pstats

# Profile a function
def slow_function():
    total = 0
    for i in range(1000000):
        total += i
    return total

cProfile.run('slow_function()')

# More detailed profiling
profiler = cProfile.Profile()
profiler.enable()
slow_function()
profiler.disable()

stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # Top 10

# Line profiler (pip install line_profiler)
# @profile  # Decorator
# def my_function():
#     ...
# Run with: kernprof -l -v script.py

# Memory profiler (pip install memory_profiler)
# @profile
# def my_function():
#     ...
# Run with: python -m memory_profiler script.py
```

### Common Optimizations

```python
# 1. Use local variables (faster than global)
def slow():
    for i in range(1000000):
        len([1,2,3])  # Global lookup each time

def fast():
    _len = len  # Local reference
    for i in range(1000000):
        _len([1,2,3])

# 2. Use list comprehensions (faster than loops)
# Slow
result = []
for i in range(1000):
    result.append(i * 2)

# Fast
result = [i * 2 for i in range(1000)]

# 3. String concatenation
# Slow
s = ""
for word in words:
    s += word  # O(n²)

# Fast
s = "".join(words)  # O(n)

# 4. Use sets for membership testing
# Slow: O(n)
if x in [1, 2, 3, 4, 5]:
    pass

# Fast: O(1)
if x in {1, 2, 3, 4, 5}:
    pass

# 5. Use generators for large data
# Memory heavy
sum([x*x for x in range(10000000)])

# Memory efficient
sum(x*x for x in range(10000000))

# 6. __slots__ for many instances
class Point:
    __slots__ = ['x', 'y']
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

---

## Interview Questions

```python
# Q1: What happens when you create a class?
"""
1. Python executes class body (creates namespace dict)
2. Calls metaclass.__new__(mcs, name, bases, namespace)
3. Calls metaclass.__init__(cls, name, bases, namespace)
4. Returns the new class object
"""

# Q2: Explain Python memory management
"""
1. Reference counting (primary): Objects track reference count
2. Garbage collection: Handles circular references
3. Memory pools: Small objects from pools (pymalloc)
4. Object caching: Small integers, interned strings
"""

# Q3: Why can't you use lists as dict keys?
"""
Lists are mutable, so their hash would change.
Dict keys must be hashable (immutable hash).
Tuples can be keys (if contents are hashable).
"""

# Q4: How does Python dict work internally?
"""
Hash table with open addressing:
1. Compute hash(key)
2. Find slot: hash % table_size
3. If collision, probe next slots
4. Store (hash, key, value) tuple
5. Resize when 2/3 full
"""

# Q5: Explain attribute lookup order
"""
1. Data descriptors (have __get__ and __set__)
2. Instance __dict__
3. Non-data descriptors and class attributes
4. __getattr__ (fallback)
"""
```
