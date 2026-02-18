# Iterators, Generators, and Comprehensions

## Iteration Protocol

### How Python Iteration Works

```python
"""
Python iteration uses two protocols:
1. Iterable: Object that implements __iter__() returning an iterator
2. Iterator: Object that implements __next__() and __iter__()

When you write: for item in collection:
Python does:
    1. Calls iter(collection) → gets iterator
    2. Repeatedly calls next(iterator) → gets items
    3. Catches StopIteration → ends loop
"""

# Manual iteration
numbers = [1, 2, 3]

# Under the hood
iterator = iter(numbers)      # Calls numbers.__iter__()
print(next(iterator))         # 1
print(next(iterator))         # 2
print(next(iterator))         # 3
# next(iterator)              # StopIteration!

# This is what a for loop does
iterator = iter(numbers)
while True:
    try:
        item = next(iterator)
        print(item)
    except StopIteration:
        break
```

### Iterable vs Iterator

```python
"""
Iterable: Has __iter__() method
- Can be iterated over multiple times
- Examples: list, tuple, str, dict, set, range

Iterator: Has __iter__() AND __next__() methods
- Maintains state (current position)
- Can only be iterated once
- iter(iterator) returns itself
"""

# Lists are iterable, not iterators
my_list = [1, 2, 3]
iter(my_list)             # Returns list_iterator object
# next(my_list)           # TypeError! list is not an iterator

# Iterators ARE also iterable
iterator = iter(my_list)
iter(iterator) is iterator  # True (returns itself)

# You can iterate an iterable multiple times
for x in my_list: print(x)  # 1, 2, 3
for x in my_list: print(x)  # 1, 2, 3 (works again)

# Iterator is exhausted after one pass
it = iter(my_list)
list(it)                    # [1, 2, 3]
list(it)                    # [] (exhausted!)
```

### Creating Custom Iterators

```python
class CountUp:
    """Iterator that counts from start to end."""

    def __init__(self, start, end):
        self.current = start
        self.end = end

    def __iter__(self):
        return self

    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

counter = CountUp(1, 5)
for num in counter:
    print(num)  # 1, 2, 3, 4, 5

# Or separate iterable and iterator
class CountUpIterable:
    """Iterable (can be reused)."""

    def __init__(self, start, end):
        self.start = start
        self.end = end

    def __iter__(self):
        # Return a fresh iterator each time
        return CountUpIterator(self.start, self.end)

class CountUpIterator:
    """Iterator (maintains state)."""

    def __init__(self, start, end):
        self.current = start
        self.end = end

    def __iter__(self):
        return self

    def __next__(self):
        if self.current > self.end:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

# Can be reused
counter = CountUpIterable(1, 3)
list(counter)  # [1, 2, 3]
list(counter)  # [1, 2, 3] (works again!)
```

---

## Generators

### Generator Functions

```python
"""
Generators are a simple way to create iterators using yield.
When called, they return a generator object (which is an iterator).
State is automatically maintained between yields.
"""

def count_up(start, end):
    """Generator function."""
    current = start
    while current <= end:
        yield current       # Pause and return value
        current += 1        # Resume here on next()

# Call returns generator object (doesn't execute body)
gen = count_up(1, 5)
print(gen)                  # <generator object count_up at 0x...>

# Execute with iteration
for num in gen:
    print(num)              # 1, 2, 3, 4, 5

# Or manually
gen = count_up(1, 3)
next(gen)                   # 1
next(gen)                   # 2
next(gen)                   # 3
# next(gen)                 # StopIteration
```

### How yield Works

```python
def simple_generator():
    print("Before first yield")
    yield 1
    print("After first yield")
    yield 2
    print("After second yield")
    yield 3
    print("After third yield")

gen = simple_generator()
# Nothing printed yet - function hasn't executed

next(gen)
# Prints: "Before first yield"
# Returns: 1

next(gen)
# Prints: "After first yield"
# Returns: 2

next(gen)
# Prints: "After second yield"
# Returns: 3

# next(gen)
# Prints: "After third yield"
# Raises: StopIteration
```

### Generator Expression

```python
# List comprehension - creates list in memory
squares_list = [x**2 for x in range(1000000)]

# Generator expression - lazy, memory efficient
squares_gen = (x**2 for x in range(1000000))

# Only difference: () instead of []
# Generator doesn't store all values, computes on demand

import sys
sys.getsizeof(squares_list)  # ~8 MB
sys.getsizeof(squares_gen)   # ~100 bytes

# Use in functions that accept iterables
sum(x**2 for x in range(100))  # Parentheses optional here
```

### yield from

```python
# Delegating to sub-generators
def chain(*iterables):
    for iterable in iterables:
        yield from iterable     # yield each item from iterable

list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]

# Without yield from
def chain_old(*iterables):
    for iterable in iterables:
        for item in iterable:
            yield item

# yield from with generators
def count_down(n):
    while n > 0:
        yield n
        n -= 1

def count_up_then_down(max_val):
    yield from range(1, max_val + 1)
    yield from count_down(max_val)

list(count_up_then_down(3))  # [1, 2, 3, 3, 2, 1]

# Flattening nested structures
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # Recursive
        else:
            yield item

list(flatten([1, [2, 3, [4, 5]], 6]))  # [1, 2, 3, 4, 5, 6]
```

### Generator Methods

```python
def interactive_generator():
    value = yield "Ready"       # Can receive values
    yield f"Received: {value}"
    yield "Done"

gen = interactive_generator()
next(gen)                       # "Ready" (start generator)
gen.send("Hello")              # "Received: Hello"
next(gen)                       # "Done"

# .throw() - raise exception in generator
def gen_with_error():
    try:
        yield 1
        yield 2
    except ValueError:
        yield "Error caught"
    yield 3

g = gen_with_error()
next(g)                         # 1
g.throw(ValueError)             # "Error caught"
next(g)                         # 3

# .close() - close generator
def must_cleanup():
    try:
        yield 1
        yield 2
    finally:
        print("Cleanup!")

g = must_cleanup()
next(g)                         # 1
g.close()                       # Prints "Cleanup!"
```

---

## Comprehensions

### List Comprehensions

```python
# Basic syntax: [expression for item in iterable]
squares = [x**2 for x in range(5)]  # [0, 1, 4, 9, 16]

# With condition (filter)
evens = [x for x in range(10) if x % 2 == 0]  # [0, 2, 4, 6, 8]

# Multiple conditions
filtered = [x for x in range(100) if x % 2 == 0 if x % 3 == 0]
# [0, 6, 12, 18, 24, 30, ...]

# if-else (in expression, not filter)
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
# ['even', 'odd', 'even', 'odd', 'even']

# Nested loops
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Nested with condition
flat_evens = [num for row in matrix for num in row if num % 2 == 0]
# [2, 4, 6, 8]

# Equivalent nested for loops:
result = []
for row in matrix:
    for num in row:
        if num % 2 == 0:
            result.append(num)

# Creating 2D structures
matrix = [[i * j for j in range(4)] for i in range(4)]
# [[0, 0, 0, 0], [0, 1, 2, 3], [0, 2, 4, 6], [0, 3, 6, 9]]
```

### Dict Comprehensions

```python
# Basic: {key_expr: value_expr for item in iterable}
squares = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# From pairs
pairs = [('a', 1), ('b', 2), ('c', 3)]
d = {k: v for k, v in pairs}  # {'a': 1, 'b': 2, 'c': 3}

# With condition
evens = {x: x**2 for x in range(10) if x % 2 == 0}

# Inverting a dict
original = {'a': 1, 'b': 2, 'c': 3}
inverted = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}

# From two lists
keys = ['name', 'age', 'city']
values = ['Alice', 30, 'NYC']
d = {k: v for k, v in zip(keys, values)}
# {'name': 'Alice', 'age': 30, 'city': 'NYC'}

# Or simply:
d = dict(zip(keys, values))
```

### Set Comprehensions

```python
# Basic: {expression for item in iterable}
unique_lengths = {len(word) for word in ['apple', 'banana', 'cat']}
# {5, 6, 3}

# With condition
evens = {x for x in range(10) if x % 2 == 0}
# {0, 2, 4, 6, 8}
```

### Generator Expressions vs Comprehensions

```python
# List comprehension: immediate, stores all values
list_comp = [x**2 for x in range(1000000)]

# Generator expression: lazy, memory efficient
gen_exp = (x**2 for x in range(1000000))

# When to use which:
# - List comprehension: Need to iterate multiple times, need len(), indexing
# - Generator expression: Single iteration, large data, or as function argument

# Generator expressions are great for:
sum(x**2 for x in range(1000))      # No extra memory
any(x > 100 for x in data)          # Short-circuits
all(x > 0 for x in data)            # Short-circuits
max(x for x in data if x % 2 == 0)  # Filter + find max
```

---

## itertools Module

### Infinite Iterators

```python
from itertools import count, cycle, repeat

# count(start=0, step=1) - infinite counting
for i in count(10, 2):  # 10, 12, 14, ...
    if i > 20:
        break
    print(i)

# cycle(iterable) - infinite cycling
colors = cycle(['red', 'green', 'blue'])
for _ in range(7):
    print(next(colors))  # red, green, blue, red, green, blue, red

# repeat(object, times=None) - repeat object
list(repeat('hello', 3))  # ['hello', 'hello', 'hello']
```

### Combinatoric Iterators

```python
from itertools import permutations, combinations, product

# permutations - all orderings
list(permutations([1, 2, 3]))
# [(1,2,3), (1,3,2), (2,1,3), (2,3,1), (3,1,2), (3,2,1)]

list(permutations([1, 2, 3], 2))  # Length 2
# [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]

# combinations - unordered selections (no repetition)
list(combinations([1, 2, 3, 4], 2))
# [(1,2), (1,3), (1,4), (2,3), (2,4), (3,4)]

# combinations_with_replacement
from itertools import combinations_with_replacement
list(combinations_with_replacement([1, 2], 2))
# [(1,1), (1,2), (2,2)]

# product - Cartesian product
list(product([1, 2], ['a', 'b']))
# [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

list(product([0, 1], repeat=3))  # Binary numbers
# [(0,0,0), (0,0,1), (0,1,0), (0,1,1), (1,0,0), (1,0,1), (1,1,0), (1,1,1)]
```

### Terminating Iterators

```python
from itertools import (
    chain, islice, takewhile, dropwhile,
    filterfalse, compress, groupby, accumulate
)

# chain - flatten multiple iterables
list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]
list(chain.from_iterable([[1, 2], [3, 4]]))  # [1, 2, 3, 4]

# islice - slice any iterable
list(islice(range(100), 5))           # [0, 1, 2, 3, 4]
list(islice(range(100), 2, 5))        # [2, 3, 4]
list(islice(range(100), 0, 10, 2))    # [0, 2, 4, 6, 8]

# takewhile / dropwhile
list(takewhile(lambda x: x < 5, [1, 3, 5, 2, 1]))
# [1, 3] - stops at first False

list(dropwhile(lambda x: x < 5, [1, 3, 5, 2, 1]))
# [5, 2, 1] - starts at first False

# filterfalse - opposite of filter
list(filterfalse(lambda x: x % 2, range(10)))
# [0, 2, 4, 6, 8] - keep where function is False

# compress - filter by selectors
list(compress('ABCDEF', [1, 0, 1, 0, 1, 1]))
# ['A', 'C', 'E', 'F']

# groupby - group consecutive elements
data = [('A', 1), ('A', 2), ('B', 3), ('B', 4), ('A', 5)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A', 1), ('A', 2)]
# B [('B', 3), ('B', 4)]
# A [('A', 5)]

# Note: groupby groups CONSECUTIVE items only!
# Sort first for global grouping:
sorted_data = sorted(data, key=lambda x: x[0])
for key, group in groupby(sorted_data, key=lambda x: x[0]):
    print(key, list(group))

# accumulate - running totals
list(accumulate([1, 2, 3, 4, 5]))
# [1, 3, 6, 10, 15]

import operator
list(accumulate([1, 2, 3, 4, 5], operator.mul))
# [1, 2, 6, 24, 120] - running product
```

### More itertools

```python
# tee - copy iterator
from itertools import tee

iter1, iter2 = tee(range(5), 2)
list(iter1)  # [0, 1, 2, 3, 4]
list(iter2)  # [0, 1, 2, 3, 4]

# zip_longest - zip with fill value
from itertools import zip_longest

list(zip_longest([1, 2, 3], ['a', 'b'], fillvalue=None))
# [(1, 'a'), (2, 'b'), (3, None)]

# starmap - map with argument unpacking
from itertools import starmap

list(starmap(pow, [(2, 3), (3, 2), (4, 2)]))
# [8, 9, 16] - pow(2,3), pow(3,2), pow(4,2)
```

---

## Performance and Best Practices

### Memory Efficiency

```python
# Generators vs Lists
# Use generators for single-pass processing of large data

# BAD: Creates large list in memory
def process_file_bad(filename):
    lines = open(filename).readlines()  # All lines in memory
    return [line.upper() for line in lines]

# GOOD: Process line by line
def process_file_good(filename):
    with open(filename) as f:
        for line in f:
            yield line.upper()

# Chain generators for pipeline processing
def read_lines(filename):
    with open(filename) as f:
        for line in f:
            yield line.strip()

def filter_comments(lines):
    for line in lines:
        if not line.startswith('#'):
            yield line

def parse_csv(lines):
    for line in lines:
        yield line.split(',')

# Pipeline: reads one line at a time through entire pipeline
for row in parse_csv(filter_comments(read_lines('data.csv'))):
    process(row)
```

### When to Use What

```python
"""
Use LIST COMPREHENSION when:
- Need to iterate multiple times
- Need len(), indexing, or slicing
- Data fits in memory
- Result is used immediately

Use GENERATOR EXPRESSION when:
- Single iteration needed
- Processing large/infinite data
- Passing to function (sum, max, join, etc.)
- Memory is a concern

Use GENERATOR FUNCTION when:
- Complex logic needed
- Multiple yields
- Need to maintain state
- Building custom iterators
"""

# Examples
# List comprehension - need to use multiple times
squares = [x**2 for x in range(10)]
print(sum(squares), max(squares), len(squares))

# Generator expression - single use
total = sum(x**2 for x in range(1000000))

# Generator function - complex logic
def fibonacci(limit):
    a, b = 0, 1
    while a < limit:
        yield a
        a, b = b, a + b
```

### Interview Patterns

```python
# 1. Lazy evaluation for large data
def get_large_dataset():
    for i in range(1000000):
        yield expensive_operation(i)

# Only processes what's needed
first_match = next(x for x in get_large_dataset() if condition(x))

# 2. Infinite sequences
def generate_ids():
    n = 0
    while True:
        yield f"ID-{n:06d}"
        n += 1

id_gen = generate_ids()
next(id_gen)  # "ID-000000"
next(id_gen)  # "ID-000001"

# 3. Pipeline pattern
def pipeline(data, *functions):
    for func in functions:
        data = (func(x) for x in data)
    return data

result = pipeline(
    range(100),
    lambda x: x * 2,
    lambda x: x + 1,
    lambda x: x if x % 3 == 0 else None,
)
# Lazy evaluation through entire pipeline

# 4. Sliding window
from itertools import islice

def sliding_window(iterable, n):
    it = iter(iterable)
    window = list(islice(it, n))
    if len(window) == n:
        yield tuple(window)
    for x in it:
        window.pop(0)
        window.append(x)
        yield tuple(window)

list(sliding_window([1, 2, 3, 4, 5], 3))
# [(1, 2, 3), (2, 3, 4), (3, 4, 5)]
```
