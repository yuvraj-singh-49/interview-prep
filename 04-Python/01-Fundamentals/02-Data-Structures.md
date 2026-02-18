# Python Built-in Data Structures

## Overview

```
Python Built-in Collections:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Sequences (Ordered)              Mappings           Sets                  │
│   ┌─────────────────┐             ┌─────────┐        ┌─────────┐           │
│   │ list   [1,2,3]  │ mutable     │ dict    │        │ set     │           │
│   │ tuple  (1,2,3)  │ immutable   │ {k: v}  │        │ {1,2,3} │           │
│   │ str    "abc"    │ immutable   └─────────┘        └─────────┘           │
│   │ range  range(5) │ immutable                      ┌─────────┐           │
│   └─────────────────┘                                │frozenset│           │
│                                                      │ immutable│           │
│                                                      └─────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘

Time Complexities:

┌─────────────────────────────────────────────────────────────────────────────┐
│ Operation          │ list      │ dict      │ set       │ tuple             │
├─────────────────────────────────────────────────────────────────────────────┤
│ Index/Key Access   │ O(1)      │ O(1) avg  │ N/A       │ O(1)              │
│ Search (in)        │ O(n)      │ O(1) avg  │ O(1) avg  │ O(n)              │
│ Insert/Append      │ O(1)*     │ O(1) avg  │ O(1) avg  │ N/A               │
│ Delete             │ O(n)      │ O(1) avg  │ O(1) avg  │ N/A               │
│ Iteration          │ O(n)      │ O(n)      │ O(n)      │ O(n)              │
└─────────────────────────────────────────────────────────────────────────────┘
* amortized
```

---

## Lists

### List Basics

```python
# Creation
empty_list = []
empty_list = list()
numbers = [1, 2, 3, 4, 5]
mixed = [1, "hello", 3.14, True, [1, 2]]  # Can mix types
from_range = list(range(5))  # [0, 1, 2, 3, 4]
from_string = list("hello")  # ['h', 'e', 'l', 'l', 'o']

# Accessing elements
numbers[0]      # 1 (first element)
numbers[-1]     # 5 (last element)
numbers[-2]     # 4 (second to last)

# Slicing [start:stop:step]
numbers[1:4]    # [2, 3, 4] (index 1 to 3)
numbers[:3]     # [1, 2, 3] (first 3)
numbers[2:]     # [3, 4, 5] (from index 2)
numbers[::2]    # [1, 3, 5] (every 2nd)
numbers[::-1]   # [5, 4, 3, 2, 1] (reversed)
numbers[1:4:2]  # [2, 4] (index 1 to 3, step 2)

# Slicing creates a COPY
original = [1, 2, 3]
copy = original[:]  # Shallow copy
copy[0] = 99
print(original)  # [1, 2, 3] (unchanged)

# Slice assignment
numbers = [1, 2, 3, 4, 5]
numbers[1:3] = [20, 30, 40]  # [1, 20, 30, 40, 4, 5]
numbers[1:4] = []            # [1, 4, 5] (delete)
```

### List Modification

```python
# Adding elements
lst = [1, 2, 3]
lst.append(4)           # [1, 2, 3, 4] - O(1) amortized
lst.insert(0, 0)        # [0, 1, 2, 3, 4] - O(n)
lst.extend([5, 6])      # [0, 1, 2, 3, 4, 5, 6] - O(k)
lst += [7, 8]           # Same as extend

# Removing elements
lst = [1, 2, 3, 2, 4]
lst.remove(2)           # [1, 3, 2, 4] - Removes first occurrence, O(n)
item = lst.pop()        # 4, lst = [1, 3, 2] - O(1) from end
item = lst.pop(0)       # 1, lst = [3, 2] - O(n) from beginning
del lst[0]              # lst = [2] - O(n)
lst.clear()             # lst = []

# Modifying
lst = [3, 1, 4, 1, 5]
lst.sort()              # [1, 1, 3, 4, 5] - In-place, returns None
lst.sort(reverse=True)  # [5, 4, 3, 1, 1]
lst.reverse()           # [1, 1, 3, 4, 5] - In-place

# Non-modifying (return new list)
lst = [3, 1, 4, 1, 5]
sorted(lst)             # [1, 1, 3, 4, 5] - Returns new list
sorted(lst, reverse=True)
list(reversed(lst))     # [5, 1, 4, 1, 3]

# Custom sorting
words = ['banana', 'apple', 'Cherry']
sorted(words)                           # ['Cherry', 'apple', 'banana']
sorted(words, key=str.lower)            # ['apple', 'banana', 'Cherry']
sorted(words, key=len)                  # ['apple', 'Cherry', 'banana']

# Sort by multiple criteria
people = [('Alice', 30), ('Bob', 25), ('Alice', 25)]
sorted(people, key=lambda x: (x[0], x[1]))  # Sort by name, then age
# [('Alice', 25), ('Alice', 30), ('Bob', 25)]
```

### List Methods Reference

```python
lst = [1, 2, 3, 2, 4]

# Information
len(lst)                # 5
lst.index(2)            # 1 (first occurrence)
lst.index(2, 2)         # 3 (search from index 2)
lst.count(2)            # 2 (occurrences)
2 in lst                # True
min(lst), max(lst)      # 1, 4
sum(lst)                # 12

# Copying
shallow = lst.copy()    # Same as lst[:]
shallow = list(lst)     # Same thing

# Deep copy (for nested structures)
import copy
deep = copy.deepcopy(nested_list)
```

### List as Stack and Queue

```python
# Stack (LIFO) - Use list
stack = []
stack.append(1)         # Push
stack.append(2)
stack.append(3)
stack.pop()             # 3 - O(1)
stack.pop()             # 2 - O(1)

# Queue (FIFO) - DON'T use list for this!
# list.pop(0) is O(n)

# Use deque instead
from collections import deque

queue = deque()
queue.append(1)         # Enqueue right
queue.append(2)
queue.popleft()         # 1 - O(1) dequeue left

# Double-ended queue
queue.appendleft(0)     # Add to left
queue.pop()             # Remove from right
```

---

## Tuples

### Tuple Basics

```python
# Creation
empty = ()
single = (1,)           # Comma needed for single element!
not_tuple = (1)         # This is just int 1 in parentheses
coords = (3, 4)
mixed = (1, "hello", [1, 2])  # Can contain mutable objects

# Packing and unpacking
point = 3, 4, 5         # Packing (parentheses optional)
x, y, z = point         # Unpacking

# Extended unpacking
first, *rest = (1, 2, 3, 4)     # first=1, rest=[2,3,4]
first, *middle, last = (1, 2, 3, 4, 5)

# Swap using tuple unpacking
a, b = b, a

# Accessing (same as lists)
t = (1, 2, 3, 4, 5)
t[0]                    # 1
t[-1]                   # 5
t[1:3]                  # (2, 3)

# Tuples are IMMUTABLE
# t[0] = 99             # TypeError!

# But mutable contents CAN be modified
t = (1, 2, [3, 4])
t[2].append(5)          # OK: t = (1, 2, [3, 4, 5])
```

### Tuple Methods and Uses

```python
t = (1, 2, 3, 2, 4)

# Methods (only 2!)
t.count(2)              # 2
t.index(2)              # 1

# Common operations
len(t)                  # 5
min(t), max(t)          # 1, 4
sum(t)                  # 12
2 in t                  # True

# Concatenation (creates new tuple)
(1, 2) + (3, 4)         # (1, 2, 3, 4)
(1, 2) * 3              # (1, 2, 1, 2, 1, 2)

# Conversion
list(t)                 # [1, 2, 3, 2, 4]
tuple([1, 2, 3])        # (1, 2, 3)
```

### Why Use Tuples?

```python
# 1. Immutable = Hashable = Can be dict keys
locations = {
    (0, 0): "origin",
    (1, 0): "east",
    (0, 1): "north"
}

# 2. Slightly faster than lists
# 3. Signal intent: "this shouldn't change"
# 4. Multiple return values
def get_min_max(numbers):
    return min(numbers), max(numbers)

min_val, max_val = get_min_max([1, 2, 3])

# 5. Named tuples for lightweight data structures
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x                     # 3
p.y                     # 4
p[0]                    # 3 (still indexable)
x, y = p                # Still unpackable

# Python 3.6+ typing.NamedTuple
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    z: float = 0.0      # Default value

p = Point(1, 2)
p.z                     # 0.0
```

---

## Dictionaries

### Dict Basics

```python
# Creation
empty = {}
empty = dict()
person = {"name": "Alice", "age": 30}
from_pairs = dict([("a", 1), ("b", 2)])
from_keys = dict.fromkeys(["a", "b", "c"], 0)  # {a: 0, b: 0, c: 0}
comprehension = {x: x**2 for x in range(5)}

# Accessing
person["name"]          # "Alice"
person["city"]          # KeyError!
person.get("city")      # None (no error)
person.get("city", "Unknown")  # "Unknown" (default)

# Modifying
person["city"] = "NYC"  # Add/update
del person["city"]      # Delete (KeyError if missing)
removed = person.pop("age")  # Remove and return
removed = person.pop("missing", None)  # No error with default

# setdefault - get or set
data = {}
data.setdefault("count", 0)  # Sets to 0 if missing, returns value
data["count"] += 1

# update - merge dicts
person.update({"age": 31, "city": "LA"})
person |= {"age": 31, "city": "LA"}  # Python 3.9+

# Merge dicts (Python 3.9+)
dict1 = {"a": 1, "b": 2}
dict2 = {"b": 3, "c": 4}
merged = dict1 | dict2  # {"a": 1, "b": 3, "c": 4}
merged = {**dict1, **dict2}  # Also works (any Python 3)
```

### Dict Iteration

```python
d = {"a": 1, "b": 2, "c": 3}

# Keys (default)
for key in d:
    print(key)

for key in d.keys():
    print(key)

# Values
for value in d.values():
    print(value)

# Key-value pairs
for key, value in d.items():
    print(f"{key}: {value}")

# Dict maintains insertion order (Python 3.7+)
d = {}
d["first"] = 1
d["second"] = 2
d["third"] = 3
list(d.keys())  # ["first", "second", "third"]
```

### Dict Methods Reference

```python
d = {"a": 1, "b": 2}

# Views (reflect changes to dict)
d.keys()                # dict_keys(['a', 'b'])
d.values()              # dict_values([1, 2])
d.items()               # dict_items([('a', 1), ('b', 2)])

# Copy
shallow = d.copy()
shallow = dict(d)

# Clear
d.clear()               # {}

# Membership
"a" in d                # True (checks keys)
1 in d.values()         # True (checks values)
```

### Common Dict Patterns

```python
# Counting
from collections import Counter
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
count = Counter(words)  # Counter({'apple': 3, 'banana': 2, 'cherry': 1})
count.most_common(2)    # [('apple', 3), ('banana', 2)]

# Manual counting
count = {}
for word in words:
    count[word] = count.get(word, 0) + 1

# Grouping
from collections import defaultdict
by_length = defaultdict(list)
for word in words:
    by_length[len(word)].append(word)
# {5: ['apple', 'apple', 'apple'], 6: ['banana', 'cherry', 'banana']}

# Inverting a dict
original = {"a": 1, "b": 2, "c": 3}
inverted = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}

# Sorting dict by values
scores = {"Alice": 85, "Bob": 90, "Charlie": 80}
sorted_by_score = sorted(scores.items(), key=lambda x: x[1], reverse=True)
# [('Bob', 90), ('Alice', 85), ('Charlie', 80)]

# Dictionary with ordered insertion + access order
from collections import OrderedDict
od = OrderedDict()
od['a'] = 1
od['b'] = 2
od.move_to_end('a')     # Move 'a' to end
od.popitem(last=False)  # Pop from beginning (FIFO)
```

### defaultdict and Counter

```python
from collections import defaultdict, Counter

# defaultdict - auto-creates missing values
dd = defaultdict(list)
dd["key"].append(1)     # No KeyError, creates empty list first
dd["key"].append(2)
# {'key': [1, 2]}

dd = defaultdict(int)   # Default value is 0
dd["count"] += 1
dd["count"] += 1
# {'count': 2}

dd = defaultdict(lambda: "N/A")
dd["missing"]           # "N/A"

# Counter
c = Counter("mississippi")
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

c.most_common(2)        # [('i', 4), ('s', 4)]
c["i"]                  # 4
c["z"]                  # 0 (missing keys return 0)

# Counter arithmetic
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
c1 + c2                 # Counter({'a': 4, 'b': 3})
c1 - c2                 # Counter({'a': 2})
c1 & c2                 # Counter({'a': 1, 'b': 1}) (min)
c1 | c2                 # Counter({'a': 3, 'b': 2}) (max)

list(c1.elements())     # ['a', 'a', 'a', 'b']
```

---

## Sets

### Set Basics

```python
# Creation
empty = set()           # NOT {} (that's a dict)
numbers = {1, 2, 3}
from_list = set([1, 2, 2, 3])  # {1, 2, 3} - duplicates removed
from_string = set("hello")     # {'h', 'e', 'l', 'o'}
comprehension = {x**2 for x in range(5)}  # {0, 1, 4, 9, 16}

# Sets contain only HASHABLE (immutable) elements
# {1, "hello", (1,2)}   # OK
# {1, "hello", [1,2]}   # TypeError! Lists not hashable

# frozenset - immutable set
fs = frozenset([1, 2, 3])
# fs.add(4)             # AttributeError!
# Can be used as dict key or set element
nested = {frozenset([1, 2]), frozenset([3, 4])}
```

### Set Operations

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

# Union (elements in either)
a | b                   # {1, 2, 3, 4, 5, 6}
a.union(b)

# Intersection (elements in both)
a & b                   # {3, 4}
a.intersection(b)

# Difference (elements in a but not b)
a - b                   # {1, 2}
a.difference(b)

# Symmetric difference (elements in either but not both)
a ^ b                   # {1, 2, 5, 6}
a.symmetric_difference(b)

# Subset/superset
{1, 2} <= {1, 2, 3}     # True (subset)
{1, 2}.issubset({1, 2, 3})
{1, 2, 3} >= {1, 2}     # True (superset)
{1, 2, 3}.issuperset({1, 2})

# Proper subset (strict)
{1, 2} < {1, 2, 3}      # True
{1, 2} < {1, 2}         # False

# Disjoint (no common elements)
{1, 2}.isdisjoint({3, 4})  # True
```

### Set Modification

```python
s = {1, 2, 3}

# Adding
s.add(4)                # {1, 2, 3, 4}
s.update([5, 6])        # {1, 2, 3, 4, 5, 6}
s |= {7, 8}             # Same as update

# Removing
s.remove(1)             # Raises KeyError if missing
s.discard(99)           # No error if missing
item = s.pop()          # Remove and return arbitrary element
s.clear()               # Empty set

# In-place operations
a = {1, 2, 3}
a |= {4, 5}             # Union update
a &= {2, 3, 4}          # Intersection update
a -= {4}                # Difference update
a ^= {1, 5}             # Symmetric difference update
```

### Set Use Cases

```python
# 1. Remove duplicates (doesn't preserve order)
items = [1, 2, 2, 3, 3, 3]
unique = list(set(items))  # [1, 2, 3]

# Preserve order (Python 3.7+)
unique = list(dict.fromkeys(items))  # [1, 2, 3]

# 2. Fast membership testing
allowed = {"admin", "user", "guest"}
if role in allowed:     # O(1) vs O(n) for list
    pass

# 3. Finding common/different elements
users_a = {"alice", "bob", "charlie"}
users_b = {"bob", "david", "eve"}

both = users_a & users_b       # {"bob"}
only_a = users_a - users_b     # {"alice", "charlie"}
all_users = users_a | users_b  # {"alice", "bob", "charlie", "david", "eve"}

# 4. Removing items from list efficiently
items = [1, 2, 3, 4, 5]
to_remove = {2, 4}
filtered = [x for x in items if x not in to_remove]  # [1, 3, 5]
```

---

## Collections Module

### Additional Data Structures

```python
from collections import deque, defaultdict, Counter, OrderedDict, ChainMap

# deque - double-ended queue with O(1) operations on both ends
d = deque([1, 2, 3])
d.append(4)             # Right: [1, 2, 3, 4]
d.appendleft(0)         # Left: [0, 1, 2, 3, 4]
d.pop()                 # 4, from right
d.popleft()             # 0, from left
d.rotate(1)             # [3, 1, 2] - rotate right
d.rotate(-1)            # [1, 2, 3] - rotate left

# Bounded deque (like circular buffer)
d = deque(maxlen=3)
d.extend([1, 2, 3])     # [1, 2, 3]
d.append(4)             # [2, 3, 4] - 1 is dropped

# ChainMap - search multiple dicts
defaults = {"color": "red", "size": "medium"}
user = {"color": "blue"}
settings = ChainMap(user, defaults)
settings["color"]       # "blue" (from user)
settings["size"]        # "medium" (from defaults)
```

### heapq - Heap/Priority Queue

```python
import heapq

# Min-heap operations
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
# heap = [1, 3, 2] (internal representation)

heapq.heappop(heap)     # 1 (smallest)
heapq.heappop(heap)     # 2

# Convert list to heap in-place
nums = [3, 1, 4, 1, 5, 9]
heapq.heapify(nums)     # O(n)

# n largest/smallest
heapq.nlargest(3, nums)   # [9, 5, 4]
heapq.nsmallest(3, nums)  # [1, 1, 3]

# For MAX-heap, negate values
max_heap = []
heapq.heappush(max_heap, -5)
heapq.heappush(max_heap, -3)
heapq.heappush(max_heap, -7)
-heapq.heappop(max_heap)  # 7 (largest)

# Priority queue with tuples (priority, item)
pq = []
heapq.heappush(pq, (2, "medium"))
heapq.heappush(pq, (1, "high"))
heapq.heappush(pq, (3, "low"))
heapq.heappop(pq)       # (1, "high")
```

---

## Interview Patterns

### Choosing the Right Data Structure

```python
"""
Decision tree:

Need order?
├── Yes
│   ├── Need mutation?
│   │   ├── Yes → list
│   │   └── No → tuple
│   └── Need fast ends?
│       └── Yes → deque
└── No
    ├── Key-value pairs?
    │   ├── Yes → dict
    │   └── Need defaults? → defaultdict
    └── Unique values only?
        ├── Need mutation? → set
        └── No → frozenset

Need to count? → Counter
Need heap/priority queue? → heapq
Need FIFO queue? → deque (or queue.Queue for threading)
"""
```

### Common Patterns

```python
# Two-pointer technique
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

# Sliding window with dict
def longest_unique_substring(s):
    seen = {}
    start = 0
    max_length = 0
    for end, char in enumerate(s):
        if char in seen and seen[char] >= start:
            start = seen[char] + 1
        seen[char] = end
        max_length = max(max_length, end - start + 1)
    return max_length

# Frequency map pattern
def is_anagram(s1, s2):
    return Counter(s1) == Counter(s2)

# Nested dict with defaultdict
def group_by_category_and_type(items):
    result = defaultdict(lambda: defaultdict(list))
    for item in items:
        result[item['category']][item['type']].append(item)
    return result

# LRU Cache implementation
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
```

### Time Complexity Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Data Structure    │ Access    │ Search    │ Insert    │ Delete             │
├─────────────────────────────────────────────────────────────────────────────┤
│ list              │ O(1)      │ O(n)      │ O(1)*     │ O(n)               │
│ tuple             │ O(1)      │ O(n)      │ N/A       │ N/A                │
│ dict              │ O(1)      │ O(1)      │ O(1)      │ O(1)               │
│ set               │ N/A       │ O(1)      │ O(1)      │ O(1)               │
│ deque             │ O(n)      │ O(n)      │ O(1)      │ O(1)               │
│ heap (via list)   │ O(1) min  │ O(n)      │ O(log n)  │ O(log n)           │
└─────────────────────────────────────────────────────────────────────────────┘
* Amortized O(1) for append, O(n) for insert at arbitrary position
Note: dict/set are average case; worst case is O(n) due to hash collisions
```
