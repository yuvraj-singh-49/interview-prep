# Object-Oriented Programming in Python

## Classes and Objects

### Basic Class Structure

```python
class Person:
    """A class representing a person."""

    # Class attribute (shared by all instances)
    species = "Homo sapiens"

    # Constructor/Initializer
    def __init__(self, name, age):
        # Instance attributes (unique to each instance)
        self.name = name
        self.age = age

    # Instance method
    def greet(self):
        return f"Hello, I'm {self.name}"

    # String representation
    def __repr__(self):
        return f"Person(name={self.name!r}, age={self.age})"

# Creating instances
alice = Person("Alice", 30)
bob = Person("Bob", 25)

# Accessing attributes
alice.name          # "Alice"
alice.species       # "Homo sapiens"
Person.species      # "Homo sapiens"

# Calling methods
alice.greet()       # "Hello, I'm Alice"

# Instance vs class attributes
alice.species = "Changed"  # Creates instance attribute
alice.species       # "Changed"
bob.species         # "Homo sapiens" (still class attribute)
Person.species      # "Homo sapiens"
```

### Understanding self

```python
class Example:
    def method(self):
        print(f"self is {self}")

e = Example()
e.method()        # self is <__main__.Example object at 0x...>

# method is actually a function
# e.method() is syntactic sugar for Example.method(e)
Example.method(e)  # Same result

# This is why you need self as first parameter
# Python doesn't have implicit 'this' like Java/JS
```

---

## Instance and Class Attributes

### Attribute Types

```python
class Counter:
    # Class attribute - shared by all instances
    total_instances = 0

    def __init__(self, start=0):
        # Instance attribute - unique to each instance
        self.count = start
        Counter.total_instances += 1

    def increment(self):
        self.count += 1

c1 = Counter(5)
c2 = Counter(10)

c1.count            # 5
c2.count            # 10
Counter.total_instances  # 2

# Gotcha: Mutable class attributes
class BadExample:
    items = []  # Shared list - usually a bug!

    def add(self, item):
        self.items.append(item)

b1 = BadExample()
b2 = BadExample()
b1.add(1)
b2.items            # [1] - Same list!

# Fix: Initialize mutable attributes in __init__
class GoodExample:
    def __init__(self):
        self.items = []  # Each instance gets its own list
```

### Attribute Access Methods

```python
class Demo:
    class_attr = "class"

    def __init__(self):
        self.instance_attr = "instance"

d = Demo()

# hasattr, getattr, setattr, delattr
hasattr(d, 'instance_attr')  # True
getattr(d, 'instance_attr')  # "instance"
getattr(d, 'missing', 'default')  # "default"
setattr(d, 'new_attr', 'value')
delattr(d, 'new_attr')

# __dict__ - instance's namespace
d.__dict__  # {'instance_attr': 'instance'}

# vars() - same as __dict__
vars(d)  # {'instance_attr': 'instance'}

# __slots__ - memory optimization
class SlottedClass:
    __slots__ = ['x', 'y']  # Only these attributes allowed

    def __init__(self, x, y):
        self.x = x
        self.y = y

s = SlottedClass(1, 2)
# s.z = 3  # AttributeError! Not in __slots__
# s.__dict__  # AttributeError! No __dict__ with __slots__
```

---

## Methods: Instance, Class, Static

```python
class MyClass:
    class_attr = "I'm a class attribute"

    def __init__(self, value):
        self.instance_attr = value

    # Instance method - has access to instance (self) and class
    def instance_method(self):
        return f"Instance method: {self.instance_attr}"

    # Class method - has access to class (cls), not instance
    @classmethod
    def class_method(cls):
        return f"Class method: {cls.class_attr}"

    # Static method - no access to instance or class
    @staticmethod
    def static_method(x, y):
        return f"Static method: {x + y}"

obj = MyClass("instance value")

# Calling methods
obj.instance_method()           # "Instance method: instance value"
MyClass.instance_method(obj)    # Same thing

obj.class_method()              # "Class method: I'm a class attribute"
MyClass.class_method()          # Same thing (cls is MyClass)

obj.static_method(1, 2)         # "Static method: 3"
MyClass.static_method(1, 2)     # Same thing
```

### When to Use Each

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    # Instance method - works with instance data
    def format(self):
        return f"{self.year}-{self.month:02d}-{self.day:02d}"

    # Class method - alternative constructor
    @classmethod
    def from_string(cls, date_string):
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)  # Creates instance of cls

    @classmethod
    def today(cls):
        import datetime
        t = datetime.date.today()
        return cls(t.year, t.month, t.day)

    # Static method - utility function related to class
    @staticmethod
    def is_valid_date(year, month, day):
        return 1 <= month <= 12 and 1 <= day <= 31

# Using the methods
d1 = Date(2024, 1, 15)
d2 = Date.from_string("2024-01-15")  # Alternative constructor
d3 = Date.today()

Date.is_valid_date(2024, 13, 1)  # False
```

---

## Properties and Descriptors

### Properties

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius  # "Protected" by convention

    # Getter
    @property
    def radius(self):
        return self._radius

    # Setter
    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

    # Deleter
    @radius.deleter
    def radius(self):
        del self._radius

    # Computed property (read-only)
    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
c.radius            # 5 (calls getter)
c.radius = 10       # Calls setter
c.area              # 314.159... (computed)
# c.area = 100      # AttributeError (no setter)
del c.radius        # Calls deleter
```

### Property without Decorators

```python
class Rectangle:
    def __init__(self, width, height):
        self._width = width
        self._height = height

    def _get_width(self):
        return self._width

    def _set_width(self, value):
        self._width = value

    # property(fget, fset, fdel, doc)
    width = property(_get_width, _set_width)

    # Computed property
    def _get_area(self):
        return self._width * self._height

    area = property(_get_area)
```

### Descriptors (Advanced)

```python
"""
Descriptors are objects that define __get__, __set__, or __delete__
They control attribute access when used as class attributes
"""

class Positive:
    """Descriptor that ensures positive values."""

    def __set_name__(self, owner, name):
        self.name = name
        self.private_name = f'_{name}'

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        return getattr(obj, self.private_name, None)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive")
        setattr(obj, self.private_name, value)

class Rectangle:
    width = Positive()   # Descriptor as class attribute
    height = Positive()

    def __init__(self, width, height):
        self.width = width   # Calls Positive.__set__
        self.height = height

r = Rectangle(5, 3)
r.width             # 5 (calls Positive.__get__)
# r.width = -1      # ValueError
```

---

## Inheritance

### Basic Inheritance

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError("Subclass must implement")

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

dog = Dog("Buddy")
dog.speak()         # "Buddy says Woof!"

# isinstance and issubclass
isinstance(dog, Dog)     # True
isinstance(dog, Animal)  # True
isinstance(dog, Cat)     # False

issubclass(Dog, Animal)  # True
issubclass(Dog, Cat)     # False
```

### super() and Method Resolution

```python
class Animal:
    def __init__(self, name):
        self.name = name
        print(f"Animal.__init__({name})")

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)  # Call parent's __init__
        self.breed = breed
        print(f"Dog.__init__({name}, {breed})")

dog = Dog("Buddy", "Labrador")
# Output:
# Animal.__init__(Buddy)
# Dog.__init__(Buddy, Labrador)

# Method overriding
class Animal:
    def speak(self):
        return "Some sound"

class Dog(Animal):
    def speak(self):
        parent_sound = super().speak()
        return f"{parent_sound}... Woof!"

Dog().speak()  # "Some sound... Woof!"
```

### Multiple Inheritance and MRO

```python
class A:
    def method(self):
        print("A.method")

class B(A):
    def method(self):
        print("B.method")
        super().method()

class C(A):
    def method(self):
        print("C.method")
        super().method()

class D(B, C):
    def method(self):
        print("D.method")
        super().method()

d = D()
d.method()
# Output:
# D.method
# B.method
# C.method
# A.method

# Method Resolution Order (MRO)
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

# MRO follows C3 linearization algorithm
# Ensures each class appears once, and child before parent
```

### Mixins

```python
"""
Mixins are classes designed to add functionality through inheritance
They're not meant to be instantiated alone
"""

class JsonMixin:
    """Add JSON serialization to any class."""
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class PrintableMixin:
    """Add pretty printing to any class."""
    def pprint(self):
        from pprint import pformat
        return pformat(self.__dict__)

class Person(JsonMixin, PrintableMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age

p = Person("Alice", 30)
p.to_json()    # '{"name": "Alice", "age": 30}'
p.pprint()     # "{'age': 30, 'name': 'Alice'}"
```

---

## Magic Methods (Dunder Methods)

### Object Lifecycle

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        """Create and return new instance (rarely overridden)"""
        print("__new__ called")
        instance = super().__new__(cls)
        return instance

    def __init__(self, value):
        """Initialize instance (called after __new__)"""
        print("__init__ called")
        self.value = value

    def __del__(self):
        """Called when instance is garbage collected"""
        print("__del__ called")

obj = MyClass(42)
# __new__ called
# __init__ called

del obj
# __del__ called (eventually, when GC runs)
```

### String Representations

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        """Official string representation (for developers)
        Should be unambiguous, ideally can recreate object"""
        return f"Person({self.name!r}, {self.age})"

    def __str__(self):
        """Informal string representation (for users)
        Should be readable"""
        return f"{self.name}, {self.age} years old"

    def __format__(self, spec):
        """Custom formatting with format() or f-strings"""
        if spec == "short":
            return self.name
        return str(self)

p = Person("Alice", 30)
repr(p)             # "Person('Alice', 30)"
str(p)              # "Alice, 30 years old"
print(p)            # "Alice, 30 years old" (uses __str__)
f"{p:short}"        # "Alice"
```

### Comparison Methods

```python
from functools import total_ordering

@total_ordering  # Fill in other comparisons from __eq__ and __lt__
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __eq__(self, other):
        if not isinstance(other, Person):
            return NotImplemented
        return self.age == other.age

    def __lt__(self, other):
        if not isinstance(other, Person):
            return NotImplemented
        return self.age < other.age

    def __hash__(self):
        """Required if __eq__ is defined and you want hashable objects"""
        return hash(self.age)

alice = Person("Alice", 30)
bob = Person("Bob", 25)

alice == bob    # False
alice > bob     # True
alice >= bob    # True (from @total_ordering)
sorted([alice, bob])  # [bob, alice]
```

### Container Methods

```python
class MyList:
    def __init__(self, items):
        self._items = list(items)

    def __len__(self):
        """len(obj)"""
        return len(self._items)

    def __getitem__(self, index):
        """obj[index]"""
        return self._items[index]

    def __setitem__(self, index, value):
        """obj[index] = value"""
        self._items[index] = value

    def __delitem__(self, index):
        """del obj[index]"""
        del self._items[index]

    def __contains__(self, item):
        """item in obj"""
        return item in self._items

    def __iter__(self):
        """for item in obj"""
        return iter(self._items)

    def __reversed__(self):
        """reversed(obj)"""
        return reversed(self._items)

ml = MyList([1, 2, 3])
len(ml)         # 3
ml[0]           # 1
ml[1] = 20      # [1, 20, 3]
2 in ml         # False (now 20)
list(ml)        # [1, 20, 3]
```

### Arithmetic Methods

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __add__(self, other):
        """self + other"""
        return Vector(self.x + other.x, self.y + other.y)

    def __radd__(self, other):
        """other + self (when other doesn't know how)"""
        return self.__add__(other)

    def __iadd__(self, other):
        """self += other (in-place)"""
        self.x += other.x
        self.y += other.y
        return self

    def __mul__(self, scalar):
        """self * scalar"""
        return Vector(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar):
        """scalar * self"""
        return self.__mul__(scalar)

    def __neg__(self):
        """-self"""
        return Vector(-self.x, -self.y)

    def __abs__(self):
        """abs(self)"""
        return (self.x**2 + self.y**2) ** 0.5

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(1, 2)
v2 = Vector(3, 4)
v1 + v2         # Vector(4, 6)
v1 * 3          # Vector(3, 6)
3 * v1          # Vector(3, 6) (uses __rmul__)
abs(v1)         # 2.236...
```

### Callable Objects

```python
class Multiplier:
    def __init__(self, factor):
        self.factor = factor

    def __call__(self, value):
        """Make instance callable like a function"""
        return value * self.factor

double = Multiplier(2)
double(5)       # 10
double(100)     # 200

# Useful for stateful functions
class Counter:
    def __init__(self):
        self.count = 0

    def __call__(self):
        self.count += 1
        return self.count

counter = Counter()
counter()       # 1
counter()       # 2
callable(counter)  # True
```

### Context Managers

```python
class FileHandler:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        """Called when entering 'with' block"""
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when exiting 'with' block"""
        self.file.close()
        # Return True to suppress exception, False to propagate
        return False

with FileHandler('test.txt', 'w') as f:
    f.write('Hello')
# File is automatically closed

# Easier way with contextlib
from contextlib import contextmanager

@contextmanager
def file_handler(filename, mode):
    f = open(filename, mode)
    try:
        yield f
    finally:
        f.close()
```

---

## Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    """Abstract base class - cannot be instantiated directly"""

    @abstractmethod
    def area(self):
        """Subclasses must implement this"""
        pass

    @abstractmethod
    def perimeter(self):
        """Subclasses must implement this"""
        pass

    def describe(self):
        """Concrete method - can be used by subclasses"""
        return f"I am a shape with area {self.area()}"

# shape = Shape()  # TypeError: Can't instantiate abstract class

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)

rect = Rectangle(5, 3)
rect.area()         # 15
rect.describe()     # "I am a shape with area 15"
```

---

## Dataclasses (Python 3.7+)

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float

# Automatically generates:
# __init__, __repr__, __eq__

p1 = Point(1, 2)
p2 = Point(1, 2)
p1 == p2            # True
repr(p1)            # "Point(x=1, y=2)"

# With more options
@dataclass(frozen=True)  # Immutable
class FrozenPoint:
    x: float
    y: float

# Default values and field()
@dataclass
class Person:
    name: str
    age: int = 0
    friends: list = field(default_factory=list)  # Mutable default

# Ordering
@dataclass(order=True)
class Student:
    sort_index: int = field(init=False, repr=False)
    name: str
    grade: float

    def __post_init__(self):
        self.sort_index = self.grade

students = [Student("Alice", 90), Student("Bob", 85)]
sorted(students)  # Sorted by grade
```

---

## Interview Questions

```python
# Q1: What's the difference between __new__ and __init__?
# __new__ creates the instance, __init__ initializes it
# __new__ is rarely overridden (singleton pattern, immutables)

# Q2: Implement a singleton
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Q3: What's the MRO for this class?
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
# D -> B -> C -> A -> object

# Q4: When would you use __slots__?
# Memory optimization when creating many instances
# Prevents adding arbitrary attributes

# Q5: Difference between @staticmethod and @classmethod?
# @staticmethod: No access to class or instance
# @classmethod: Access to class (cls), commonly for alternative constructors
```
