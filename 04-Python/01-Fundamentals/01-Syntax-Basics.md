# Python Syntax Basics

## Why Python?

```
Python Philosophy (The Zen of Python - `import this`):

- Beautiful is better than ugly
- Explicit is better than implicit
- Simple is better than complex
- Readability counts
- There should be one obvious way to do it

Key Characteristics:
- Interpreted (no compilation step)
- Dynamically typed (types checked at runtime)
- Strongly typed (no implicit type coercion like JS)
- Indentation-based syntax (whitespace matters!)
- Everything is an object
```

---

## Variables and Assignment

```python
# Variables are references to objects (no declaration needed)
name = "Alice"          # String
age = 30                # Integer
salary = 75000.50       # Float
is_active = True        # Boolean

# Multiple assignment
x, y, z = 1, 2, 3
a = b = c = 0           # All reference same object

# Swap values (Pythonic way)
x, y = y, x

# Unpacking
first, *rest = [1, 2, 3, 4, 5]  # first=1, rest=[2,3,4,5]
first, *middle, last = [1, 2, 3, 4, 5]  # first=1, middle=[2,3,4], last=5

# Constants (by convention, no true constants in Python)
MAX_SIZE = 100          # UPPER_CASE signals "don't modify"
PI = 3.14159

# Naming conventions
variable_name = "snake_case for variables and functions"
ClassName = "PascalCase for classes"
_private = "leading underscore suggests internal use"
__mangled = "double underscore triggers name mangling"
__dunder__ = "double underscore both sides = magic methods"
```

---

## Data Types

### Numeric Types

```python
# Integers - unlimited precision!
small = 42
large = 10**100         # Python handles arbitrarily large integers
binary = 0b1010         # 10 in binary
octal = 0o12            # 10 in octal
hex_val = 0xFF          # 255 in hex

# Floats - 64-bit double precision (IEEE 754)
pi = 3.14159
scientific = 2.5e-3     # 0.0025
infinity = float('inf')
not_a_number = float('nan')

# Complex numbers
c = 3 + 4j
c.real                  # 3.0
c.imag                  # 4.0
abs(c)                  # 5.0 (magnitude)

# Type checking
type(42)                # <class 'int'>
isinstance(42, int)     # True
isinstance(42, (int, float))  # True (multiple types)

# Type conversion
int("42")               # 42
int(3.7)                # 3 (truncates, doesn't round)
float("3.14")           # 3.14
str(42)                 # "42"
bool(0)                 # False
bool("")                # False
bool([])                # False (empty collections are falsy)
```

### Strings

```python
# String creation
single = 'Hello'
double = "World"
multiline = """This is
a multiline
string"""
raw = r"C:\Users\name"  # Raw string (no escape processing)

# String operations
s = "Hello, World!"
len(s)                  # 13
s[0]                    # 'H' (indexing)
s[-1]                   # '!' (negative indexing from end)
s[0:5]                  # 'Hello' (slicing)
s[7:]                   # 'World!' (slice to end)
s[::2]                  # 'Hlo ol!' (every 2nd char)
s[::-1]                 # '!dlroW ,olleH' (reverse)

# Strings are immutable
# s[0] = 'h'            # TypeError!

# String methods (return new strings)
s.lower()               # 'hello, world!'
s.upper()               # 'HELLO, WORLD!'
s.strip()               # Remove whitespace from ends
s.split(', ')           # ['Hello', 'World!']
', '.join(['a', 'b'])   # 'a, b'
s.replace('World', 'Python')  # 'Hello, Python!'
s.startswith('Hello')   # True
s.endswith('!')         # True
s.find('World')         # 7 (index, -1 if not found)
s.index('World')        # 7 (raises ValueError if not found)
s.count('l')            # 3

# String formatting
name = "Alice"
age = 30

# f-strings (Python 3.6+) - PREFERRED
f"Name: {name}, Age: {age}"
f"Next year: {age + 1}"
f"Pi: {3.14159:.2f}"    # "Pi: 3.14" (2 decimal places)
f"{name:>10}"           # "     Alice" (right-align, width 10)
f"{name:<10}"           # "Alice     " (left-align)
f"{name:^10}"           # "  Alice   " (center)

# format() method
"Name: {}, Age: {}".format(name, age)
"Name: {n}, Age: {a}".format(n=name, a=age)

# % formatting (older style)
"Name: %s, Age: %d" % (name, age)

# String checks
"hello".isalpha()       # True (all letters)
"123".isdigit()         # True (all digits)
"hello123".isalnum()    # True (alphanumeric)
"   ".isspace()         # True
"Hello".istitle()       # True (title case)
```

### Boolean and None

```python
# Boolean values
True
False

# Falsy values (evaluate to False in boolean context)
False
None
0, 0.0, 0j              # Zero of any numeric type
"", '', """"""          # Empty strings
[], (), {}              # Empty collections
set()                   # Empty set

# Everything else is truthy
bool([1, 2])            # True
bool("hello")           # True
bool(-1)                # True

# None - represents absence of value
x = None
x is None               # True (use 'is', not '==')
x is not None           # False

# Boolean operations
a and b                 # True if both True
a or b                  # True if either True
not a                   # Negation

# Short-circuit evaluation
x = None
y = x or "default"      # y = "default" (x is falsy)

def expensive():
    print("Called!")
    return True

False and expensive()   # expensive() never called
True or expensive()     # expensive() never called
```

---

## Operators

### Arithmetic Operators

```python
# Basic arithmetic
10 + 3      # 13   Addition
10 - 3      # 7    Subtraction
10 * 3      # 30   Multiplication
10 / 3      # 3.333... Division (always returns float)
10 // 3     # 3    Floor division (integer result)
10 % 3      # 1    Modulo (remainder)
10 ** 3     # 1000 Exponentiation

# Augmented assignment
x = 10
x += 5      # x = x + 5 = 15
x -= 3      # x = x - 3 = 12
x *= 2      # x = x * 2 = 24
x //= 5     # x = x // 5 = 4

# Division gotchas
7 / 2       # 3.5 (true division)
7 // 2      # 3 (floor division)
-7 // 2     # -4 (floors toward negative infinity!)
int(-7 / 2) # -3 (truncates toward zero)

# divmod - get both quotient and remainder
divmod(17, 5)  # (3, 2) -> 17 = 5*3 + 2
```

### Comparison Operators

```python
# Value comparison
x == y      # Equal
x != y      # Not equal
x < y       # Less than
x <= y      # Less than or equal
x > y       # Greater than
x >= y      # Greater than or equal

# Chained comparisons (Pythonic!)
0 < x < 10              # True if x is between 0 and 10
a == b == c             # True if all are equal

# Identity comparison
x is y      # True if same object (same id())
x is not y  # True if different objects

# IMPORTANT: == vs is
a = [1, 2, 3]
b = [1, 2, 3]
c = a

a == b      # True (same values)
a is b      # False (different objects)
a is c      # True (same object)

# For small integers and interned strings, Python caches them
x = 256
y = 256
x is y      # True (cached)

x = 257
y = 257
x is y      # False (or True in some contexts - implementation detail!)
# NEVER rely on this - always use == for value comparison
```

### Logical Operators

```python
# and, or, not
True and False      # False
True or False       # True
not True            # False

# Short-circuit evaluation returns the deciding value
# (not always boolean!)
"hello" and "world"     # "world" (last truthy value)
"" and "world"          # "" (first falsy value)
"hello" or "world"      # "hello" (first truthy value)
"" or "world"           # "world" (first truthy value)
"" or "" or "default"   # "default"

# Common patterns
name = user_name or "Anonymous"  # Default value
result = data and data[0]        # Safe access (returns None if data is empty)
```

### Bitwise Operators

```python
# Bitwise operations (work on integers)
a = 0b1010  # 10
b = 0b1100  # 12

a & b       # 0b1000 = 8  (AND)
a | b       # 0b1110 = 14 (OR)
a ^ b       # 0b0110 = 6  (XOR)
~a          # -11 (NOT, two's complement)
a << 2      # 0b101000 = 40 (left shift)
a >> 1      # 0b0101 = 5 (right shift)

# Common uses
# Check if even/odd
n & 1       # 0 if even, 1 if odd

# Check if power of 2
n > 0 and (n & (n-1)) == 0

# Set/clear/toggle bits
flags |= mask   # Set bits
flags &= ~mask  # Clear bits
flags ^= mask   # Toggle bits
```

### Membership and Sequence Operators

```python
# in / not in
'a' in 'abc'            # True
1 in [1, 2, 3]          # True
'key' in {'key': 'val'} # True (checks keys)
1 not in [2, 3, 4]      # True

# Sequence operations
[1, 2] + [3, 4]         # [1, 2, 3, 4] (concatenation)
[1, 2] * 3              # [1, 2, 1, 2, 1, 2] (repetition)
"ab" * 3                # "ababab"

# Gotcha with repetition!
matrix = [[0] * 3] * 3  # DON'T do this!
# All rows reference the same list!
matrix[0][0] = 1
print(matrix)           # [[1, 0, 0], [1, 0, 0], [1, 0, 0]]

# Correct way:
matrix = [[0] * 3 for _ in range(3)]
```

---

## Control Flow

### if / elif / else

```python
# Basic if statement
x = 10
if x > 0:
    print("Positive")
elif x < 0:
    print("Negative")
else:
    print("Zero")

# Ternary expression (conditional expression)
result = "even" if x % 2 == 0 else "odd"

# Nested ternary (avoid for readability)
result = "positive" if x > 0 else ("negative" if x < 0 else "zero")

# Match statement (Python 3.10+) - Pattern matching
def http_status(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:  # Multiple patterns
            return "Server Error"
        case _:                 # Default case
            return "Unknown"

# Match with destructuring
def process_point(point):
    match point:
        case (0, 0):
            return "Origin"
        case (x, 0):
            return f"On X-axis at {x}"
        case (0, y):
            return f"On Y-axis at {y}"
        case (x, y):
            return f"Point at ({x}, {y})"
        case _:
            return "Not a point"
```

### Loops

```python
# for loop - iterates over sequences
for i in [1, 2, 3]:
    print(i)

for char in "hello":
    print(char)

for key in {'a': 1, 'b': 2}:  # Iterates over keys
    print(key)

# range() - generates sequence of numbers
for i in range(5):            # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 5):         # 2, 3, 4
    print(i)

for i in range(0, 10, 2):     # 0, 2, 4, 6, 8 (step=2)
    print(i)

for i in range(5, 0, -1):     # 5, 4, 3, 2, 1 (countdown)
    print(i)

# enumerate() - get index and value
for index, value in enumerate(['a', 'b', 'c']):
    print(f"{index}: {value}")  # 0: a, 1: b, 2: c

for index, value in enumerate(['a', 'b', 'c'], start=1):
    print(f"{index}: {value}")  # 1: a, 2: b, 3: c

# zip() - iterate over multiple sequences in parallel
names = ['Alice', 'Bob']
ages = [30, 25]
for name, age in zip(names, ages):
    print(f"{name} is {age}")

# while loop
count = 0
while count < 5:
    print(count)
    count += 1

# break - exit loop
for i in range(10):
    if i == 5:
        break
    print(i)  # 0, 1, 2, 3, 4

# continue - skip to next iteration
for i in range(5):
    if i == 2:
        continue
    print(i)  # 0, 1, 3, 4

# else clause (executes if loop completes without break)
for i in range(5):
    if i == 10:
        break
else:
    print("Loop completed")  # This prints

for i in range(5):
    if i == 3:
        break
else:
    print("Loop completed")  # This doesn't print

# Common pattern: search with else
def find_item(items, target):
    for item in items:
        if item == target:
            print(f"Found {target}")
            break
    else:
        print(f"{target} not found")
```

### Loop Techniques

```python
# Iterating with index (avoid when possible)
# Bad:
for i in range(len(items)):
    print(items[i])

# Good:
for item in items:
    print(item)

# When you need index:
for i, item in enumerate(items):
    print(f"{i}: {item}")

# Iterating over dict
d = {'a': 1, 'b': 2, 'c': 3}

for key in d:           # Keys
    print(key)

for value in d.values():    # Values
    print(value)

for key, value in d.items():  # Key-value pairs
    print(f"{key}: {value}")

# Reverse iteration
for item in reversed([1, 2, 3]):
    print(item)  # 3, 2, 1

# Sorted iteration
for item in sorted([3, 1, 2]):
    print(item)  # 1, 2, 3

for item in sorted([3, 1, 2], reverse=True):
    print(item)  # 3, 2, 1
```

---

## Input/Output

```python
# Console output
print("Hello")                      # Hello
print("Hello", "World")             # Hello World (space-separated)
print("Hello", "World", sep=", ")   # Hello, World (custom separator)
print("Hello", end="")              # No newline
print("Line 1", "Line 2", sep="\n") # Each on new line

# Formatted output
name = "Alice"
age = 30
print(f"Name: {name}, Age: {age}")

# Console input
name = input("Enter your name: ")   # Always returns string
age = int(input("Enter your age: "))  # Convert to int

# File I/O
# Writing
with open('file.txt', 'w') as f:    # 'w' = write (creates/overwrites)
    f.write("Hello\n")
    f.write("World\n")

# Reading
with open('file.txt', 'r') as f:    # 'r' = read (default)
    content = f.read()              # Entire file as string

with open('file.txt', 'r') as f:
    lines = f.readlines()           # List of lines (with \n)

with open('file.txt', 'r') as f:
    for line in f:                  # Iterate line by line (memory efficient)
        print(line.strip())

# File modes
'r'     # Read (default)
'w'     # Write (creates/overwrites)
'a'     # Append
'x'     # Exclusive create (fails if exists)
'b'     # Binary mode ('rb', 'wb')
't'     # Text mode (default)
'+'     # Read and write ('r+', 'w+')

# Context manager ensures file is closed
# Even if an exception occurs
```

---

## Interview Quick Reference

### Common Gotchas

```python
# 1. Mutable default arguments
def append_to(element, to=[]):  # DON'T!
    to.append(element)
    return to

append_to(1)  # [1]
append_to(2)  # [1, 2] - Same list!

# Fix:
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to

# 2. Late binding closures
funcs = [lambda x: x * i for i in range(3)]
[f(2) for f in funcs]  # [4, 4, 4] - All use i=2!

# Fix:
funcs = [lambda x, i=i: x * i for i in range(3)]
[f(2) for f in funcs]  # [0, 2, 4]

# 3. Integer division
5 / 2   # 2.5 (float division)
5 // 2  # 2 (floor division)
-5 // 2 # -3 (floors toward negative infinity!)

# 4. String immutability
s = "hello"
# s[0] = 'H'  # TypeError!
s = 'H' + s[1:]  # Create new string

# 5. is vs ==
a = [1, 2, 3]
b = [1, 2, 3]
a == b  # True (same values)
a is b  # False (different objects)
```

### Performance Tips

```python
# String concatenation
# Bad (O(n^2)):
s = ""
for item in items:
    s += str(item)

# Good (O(n)):
s = "".join(str(item) for item in items)

# Membership testing
# Bad (O(n)):
if x in [1, 2, 3, 4, 5]:
    pass

# Good (O(1)):
if x in {1, 2, 3, 4, 5}:  # Set lookup
    pass

# Multiple comparisons
# Instead of:
if x == 1 or x == 2 or x == 3:
    pass

# Use:
if x in {1, 2, 3}:
    pass
```
