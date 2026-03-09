# Classes and OOP in Python

**Related:** [data types](data_types_in_python.md) | [functions](functions_in_python.md) | [iterators & generators](iterators_and_generators.md) | [best practices](python_best_practices.md)

---

## Class Creation — type is the Metaclass

When you write `class Foo:`, Python calls `type()` to create the class object. Classes are themselves **objects** — instances of `type`.

```python
class Dog:
    species = "Canis familiaris"

    def __init__(self, name):
        self.name = name

# Equivalent to:
Dog = type('Dog', (object,), {
    'species': 'Canis familiaris',
    '__init__': lambda self, name: setattr(self, name, name),
})

type(Dog)       # <class 'type'>   — Dog is an instance of type
type(type)      # <class 'type'>   — type is its own metaclass
isinstance(Dog, type)  # True
```

```
Metaclass hierarchy:
┌──────────┐     instance of
│  type     │◄──────────────── type (type is an instance of itself)
│ (metaclass)│
└────┬──────┘
     │ instance of
     v
┌──────────┐     instance of
│   Dog    │◄──────────────── dog_obj = Dog("Rex")
│ (class)  │
└──────────┘
```

---

## __init__ vs __new__

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        # __new__ creates the instance (allocates memory)
        # Called BEFORE __init__
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, value):
        # __init__ initializes the instance (sets attributes)
        # Called AFTER __new__
        self.value = value

a = Singleton(1)
b = Singleton(2)
a is b          # True — same instance
a.value         # 2 — __init__ was called again on the same instance
```

**How to think about it:**
- `__new__` is the **constructor** — it creates and returns the object. Rarely overridden except for immutables and singletons.
- `__init__` is the **initializer** — it sets up the already-created object. This is what you override 99% of the time.

---

## MRO — Method Resolution Order (C3 Linearization)

Python uses the **C3 linearization algorithm** to determine the order in which base classes are searched for methods. This solves the diamond problem.

```python
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

D().method()    # "B" — B comes before C in the MRO
D.__mro__       # (D, B, C, A, object)
```

```
Diamond inheritance:
        A
       / \
      B   C
       \ /
        D

MRO: D → B → C → A → object
C3 guarantees:
  1. Children come before parents
  2. Left-to-right order of bases is preserved
  3. Monotonic: if X comes before Y in any linearization,
     X comes before Y in all subclass linearizations
```

### super() — Follows the MRO

```python
class A:
    def method(self):
        print("A.method")

class B(A):
    def method(self):
        print("B.method")
        super().method()    # calls NEXT in MRO, not necessarily parent

class C(A):
    def method(self):
        print("C.method")
        super().method()

class D(B, C):
    def method(self):
        print("D.method")
        super().method()

D().method()
# D.method
# B.method
# C.method   ← super() in B calls C, not A!
# A.method
```

**Key insight:** `super()` follows the MRO chain, NOT the class hierarchy. In B's `super().method()`, the next class in D's MRO after B is C, not A.

---

## Dunder Methods (Magic Methods)

Dunder methods let you define how objects behave with Python operators and built-in functions.

### Representation

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        """For developers — unambiguous, ideally eval()-able."""
        return f"Point({self.x}, {self.y})"

    def __str__(self):
        """For users — readable."""
        return f"({self.x}, {self.y})"

p = Point(3, 4)
repr(p)     # "Point(3, 4)"   — used in REPL, debugger
str(p)      # "(3, 4)"        — used in print()
print(p)    # (3, 4)          — calls __str__, falls back to __repr__
```

### Comparison and Hashing

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __eq__(self, other):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))

    def __lt__(self, other):
        """Enables sorting: sorted() and < operator."""
        return (self.x, self.y) < (other.x, other.y)

# With __eq__ and __lt__, you can sort:
points = [Point(3, 4), Point(1, 2), Point(3, 1)]
sorted(points)  # [Point(1, 2), Point(3, 1), Point(3, 4)]

# IMPORTANT: If you define __eq__, Python sets __hash__ to None
# (makes instances unhashable). You must explicitly define __hash__
# if you want to use instances as dict keys or set members.
```

### Container Protocol

```python
class Matrix:
    def __init__(self, data):
        self._data = data

    def __getitem__(self, key):
        """Enables m[i] and m[i:j] syntax."""
        return self._data[key]

    def __len__(self):
        """Enables len(m)."""
        return len(self._data)

    def __contains__(self, item):
        """Enables `x in m` syntax."""
        return item in self._data

    def __iter__(self):
        """Enables `for x in m` syntax."""
        return iter(self._data)
```

---

## Descriptors — How property() Works

A descriptor is any object that defines `__get__`, `__set__`, or `__delete__`. This is the mechanism behind `property`, `classmethod`, `staticmethod`, and `__slots__`.

```python
class Validator:
    """A descriptor that validates values."""
    def __init__(self, min_val, max_val):
        self.min_val = min_val
        self.max_val = max_val

    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        return getattr(obj, f'_{self.name}', None)

    def __set__(self, obj, value):
        if not self.min_val <= value <= self.max_val:
            raise ValueError(f"{self.name} must be between {self.min_val} and {self.max_val}")
        setattr(obj, f'_{self.name}', value)

class Student:
    grade = Validator(0, 100)   # descriptor instance on the CLASS

    def __init__(self, name, grade):
        self.name = name
        self.grade = grade   # triggers Validator.__set__

s = Student("Alice", 95)   # OK
s = Student("Bob", 150)    # ValueError!
```

### How property() Uses Descriptors

```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

    @property
    def area(self):
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
c.radius        # 5 (calls getter via descriptor protocol)
c.area          # 78.53... (computed property, no setter — read-only)
c.radius = 10   # calls setter via descriptor protocol
c.radius = -1   # ValueError
```

---

## __slots__ — Memory Optimization

By default, each instance stores attributes in a `__dict__` (a dict object — 64+ bytes overhead). `__slots__` replaces the dict with a fixed-size struct.

```python
class PointWithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class PointWithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

import sys
sys.getsizeof(PointWithDict(1, 2))   # 48 bytes + 104 bytes for __dict__
sys.getsizeof(PointWithSlots(1, 2))  # 48 bytes total (no __dict__)
```

```
Without __slots__:
┌──────────────────┐
│ ob_refcnt         │
│ ob_type → class  │
│ __dict__ → ──────┼──→ dict object (64+ bytes)
│ __weakref__      │       {"x": 1, "y": 2}
└──────────────────┘

With __slots__:
┌──────────────────┐
│ ob_refcnt         │
│ ob_type → class  │
│ x: 1 (slot)     │  ← stored directly as descriptor
│ y: 2 (slot)     │  ← no dict overhead
└──────────────────┘
```

**Trade-offs:**
- Saves ~50-60% memory per instance (significant with millions of objects)
- Slightly faster attribute access (descriptor lookup vs dict lookup)
- Cannot add arbitrary attributes dynamically
- Must redeclare `__slots__` in subclasses

---

## Dataclasses (Python 3.7+)

Dataclasses auto-generate `__init__`, `__repr__`, `__eq__`, and more.

```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float

    # Auto-generated:
    # def __init__(self, x: float, y: float): self.x = x; self.y = y
    # def __repr__(self): return f"Point(x={self.x}, y={self.y})"
    # def __eq__(self, other): return (self.x, self.y) == (other.x, other.y)

@dataclass(frozen=True)    # immutable + hashable
class FrozenPoint:
    x: float
    y: float

@dataclass(order=True)     # generates __lt__, __le__, __gt__, __ge__
class Student:
    gpa: float
    name: str = field(compare=False)  # excluded from comparison
```

**How to think about it:** Dataclasses are a code generator, not a base class. The `@dataclass` decorator inspects the class annotations and writes methods at class creation time.

---

## Abstract Base Classes (ABCs)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Must be implemented by subclasses."""
        pass

    @abstractmethod
    def perimeter(self) -> float:
        pass

    def describe(self):
        """Concrete method — inherited as-is."""
        return f"Area: {self.area()}, Perimeter: {self.perimeter()}"

# Shape()  # TypeError: Can't instantiate abstract class

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        import math
        return math.pi * self.radius ** 2

    def perimeter(self):
        import math
        return 2 * math.pi * self.radius
```

---

## Protocols — Structural Subtyping (Python 3.8+)

Protocols are Python's formalization of duck typing. A class satisfies a Protocol if it has the right methods — no explicit inheritance required.

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

class Square:
    def draw(self) -> None:
        print("Drawing square")

def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())  # works — Circle has .draw()
render(Square())  # works — Square has .draw()
# Neither Circle nor Square explicitly inherits from Drawable
```

**This is like Go's interfaces** — structural typing checked by static type checkers (mypy), not at runtime.

---

## Composition vs Inheritance

```python
# BAD: deep inheritance hierarchy
class Animal:
    def move(self): ...
class FlyingAnimal(Animal):
    def fly(self): ...
class SwimmingAnimal(Animal):
    def swim(self): ...
class Duck(FlyingAnimal, SwimmingAnimal):  # diamond problem!
    pass

# GOOD: composition
class Duck:
    def __init__(self):
        self.flying = FlyingAbility()
        self.swimming = SwimmingAbility()

    def fly(self):
        return self.flying.fly()

    def swim(self):
        return self.swimming.swim()
```

**Rule of thumb:** Use inheritance for "is-a" relationships (rare). Use composition for "has-a" relationships (common). Prefer protocols over ABCs when you just need structural typing.

---

## Practical Examples / LeetCode Patterns

### Custom Sortable Objects (LeetCode 179: Largest Number)

```python
from functools import cmp_to_key

def largestNumber(nums):
    # Custom comparison: compare concatenated strings
    nums_str = [str(n) for n in nums]
    nums_str.sort(key=cmp_to_key(lambda a, b: -1 if a+b > b+a else 1))
    result = ''.join(nums_str)
    return '0' if result[0] == '0' else result
```

### Iterator Pattern (LeetCode 284: Peeking Iterator)

```python
class PeekingIterator:
    def __init__(self, iterator):
        self.iterator = iterator
        self._next = next(self.iterator, None)
        self._has_next = self._next is not None

    def peek(self):
        return self._next

    def next(self):
        val = self._next
        try:
            self._next = next(self.iterator)
        except StopIteration:
            self._next = None
            self._has_next = False
        return val

    def hasNext(self):
        return self._has_next
```

### Dataclass for Graph Nodes

```python
from dataclasses import dataclass, field

@dataclass
class TreeNode:
    val: int
    left: 'TreeNode | None' = None
    right: 'TreeNode | None' = None

# Clean construction:
root = TreeNode(1, TreeNode(2), TreeNode(3, TreeNode(4)))
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Metaclass | `type` creates classes. Classes are instances of `type` |
| MRO | C3 linearization. `super()` follows MRO, not parent hierarchy |
| __init__ vs __new__ | `__new__` creates (allocates), `__init__` initializes. Override `__init__` 99% of the time |
| Dunders | `__eq__` + `__hash__` for dict keys. `__lt__` for sorting. `__repr__` for debugging |
| Descriptors | The mechanism behind `property`, `classmethod`, `staticmethod` |
| __slots__ | Replaces `__dict__` with fixed slots. ~50% memory savings per instance |
| Dataclasses | Auto-generate `__init__`, `__repr__`, `__eq__`. Use `frozen=True` for immutable |
| Protocols | Structural subtyping (like Go interfaces). No explicit inheritance needed |
| Composition | Prefer over inheritance. Use "has-a" not "is-a" |
