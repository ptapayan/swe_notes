# Data Types in Python

**Related:** [Python as a language](python_as_a_language.md) | [lists & tuples](lists_and_tuples.md) | [dicts & sets](dictionaries_and_sets.md) | [strings](strings_in_python.md) | [classes & OOP](classes_and_oop.md)

---

## Everything Is an Object

Every value in Python is an object on the heap. There are no "primitives" like in Java or Go. Even `42` is a full `PyObject` with a reference count and type pointer.

```
PyObject (base of EVERY object):
┌──────────────────────────────┐
│ ob_refcnt: Py_ssize_t (8B)   │  <-- reference count
│ ob_type: *PyTypeObject (8B)  │  <-- pointer to the type object (int, str, etc.)
│ ... type-specific data ...   │
└──────────────────────────────┘

Minimum overhead per object: 16 bytes (refcount + type pointer)
+ 8 bytes for GC header on container objects (list, dict, set)
```

```python
x = 42
type(x)       # <class 'int'>
id(x)         # memory address of the object (e.g., 140234567890)
x.__class__   # <class 'int'> — same as type(x)

# Even functions and classes are objects:
type(print)   # <class 'builtin_function_or_method'>
type(int)     # <class 'type'> — int itself is an object of type 'type'
```

---

## int — Arbitrary Precision Integers

Python integers have **unlimited precision**. There is no overflow — they grow as large as memory allows.

### PyLongObject Internal Representation

```
Small int (|value| < 2^30):
┌──────────────┐
│ ob_refcnt    │
│ ob_type → int│
│ ob_size: 1   │  <-- number of "digits" (30-bit chunks)
│ ob_digit[0]  │  <-- the value stored in a single 30-bit digit
└──────────────┘

Large int (e.g., 2**100):
┌──────────────┐
│ ob_refcnt    │
│ ob_type → int│
│ ob_size: 4   │  <-- 4 digits needed
│ ob_digit[0]  │  <-- least significant 30-bit chunk
│ ob_digit[1]  │
│ ob_digit[2]  │
│ ob_digit[3]  │  <-- most significant chunk
└──────────────┘
```

CPython stores large integers as arrays of 30-bit "digits" (stored in 32-bit `uint32_t`). Arithmetic is schoolbook multiplication for small numbers, Karatsuba for large ones.

```python
import sys

sys.getsizeof(0)          # 28 bytes (object header + size field + 0 digits)
sys.getsizeof(1)          # 28 bytes (1 digit)
sys.getsizeof(2**30)      # 32 bytes (2 digits — crossed the 30-bit boundary)
sys.getsizeof(2**100)     # 44 bytes (4 digits)
sys.getsizeof(10**1000)   # 468 bytes (110 digits)

# No overflow — ever:
2 ** 1000  # works fine, returns a 302-digit number
```

### Small Integer Cache

CPython **caches integers from -5 to 256** as singletons. They are pre-allocated at interpreter startup and reused.

```python
a = 256
b = 256
a is b    # True — same object (cached)

a = 257
b = 257
a is b    # False — different objects (not cached)
# (may vary: the REPL sometimes interns larger ints within the same compilation unit)
```

---

## float — IEEE 754 Double Precision

Python `float` is a C `double` (64-bit IEEE 754). Always. There is no single-precision float.

```
PyFloatObject:
┌──────────────────────┐
│ ob_refcnt (8B)       │
│ ob_type → float (8B) │
│ ob_fval (8B)         │  <-- the actual double-precision value
└──────────────────────┘
Total: 24 bytes per float object
```

```python
import sys
sys.getsizeof(3.14)  # 24 bytes

# IEEE 754 quirks apply:
0.1 + 0.2            # 0.30000000000000004
0.1 + 0.2 == 0.3     # False!

# Use math.isclose for comparison:
import math
math.isclose(0.1 + 0.2, 0.3)  # True

# Special values:
float('inf')         # positive infinity
float('-inf')        # negative infinity
float('nan')         # Not a Number
float('nan') == float('nan')  # False (NaN != NaN by IEEE spec)
```

For exact decimal arithmetic (finance, currency), use `decimal.Decimal`:

```python
from decimal import Decimal
Decimal('0.1') + Decimal('0.2')  # Decimal('0.3') — exact
```

---

## bool — Subclass of int

`bool` is literally a subclass of `int`. `True` is `1`, `False` is `0`.

```python
isinstance(True, int)   # True
True + True             # 2
True * 10               # 10
False + 1               # 1

# This enables counting tricks:
nums = [1, 2, 3, 4, 5]
count_even = sum(1 for n in nums if n % 2 == 0)  # 2
# or equivalently:
count_even = sum(n % 2 == 0 for n in nums)  # 2 (True=1, False=0)
```

`True` and `False` are **singletons** — there is only one `True` object and one `False` object in the interpreter.

### Truthiness (Falsy Values)

```python
# These are ALL falsy:
bool(None)      # False
bool(False)     # False
bool(0)         # False
bool(0.0)       # False
bool("")        # False
bool([])        # False
bool(())        # False
bool({})        # False
bool(set())     # False

# Everything else is truthy:
bool(1)         # True
bool(-1)        # True
bool("hello")   # True
bool([0])       # True (non-empty list, even if contents are falsy)
```

**How it works under the hood:** Python calls the object's `__bool__()` method. If not defined, it falls back to `__len__()` — returning 0 means falsy. If neither is defined, the object is truthy.

---

## None — The Singleton

`None` is Python's null value. It is a **singleton** — there is exactly one `None` object in the interpreter.

```python
x = None
x is None     # True — always use `is` to compare with None, not ==
type(None)    # <class 'NoneType'>

# None is the default return value of functions:
def f():
    pass

result = f()
result is None  # True
```

**Why `is` instead of `==`:** The `==` operator calls `__eq__`, which can be overridden by any class to return `True` for `None`. The `is` operator checks identity (same object in memory), which cannot be faked.

---

## str — Unicode Strings

Strings are **immutable sequences of Unicode code points**. CPython uses PEP 393 compact representation internally.

```python
s = "Hello, World!"
type(s)         # <class 'str'>
len(s)          # 13 (number of characters, NOT bytes)

# Immutable:
# s[0] = 'h'   # TypeError: 'str' object does not support item assignment
```

**Related:** [strings in Python](strings_in_python.md) for deep dive on Unicode internals, PEP 393, encoding.

---

## bytes and bytearray

```python
# bytes — immutable sequence of integers 0-255
b = b"hello"
b[0]            # 104 (ASCII value of 'h')
# b[0] = 72    # TypeError: immutable

# bytearray — mutable version
ba = bytearray(b"hello")
ba[0] = 72      # OK
bytes(ba)       # b'Hello'

# Conversion:
s = "hello"
encoded = s.encode('utf-8')    # str -> bytes
decoded = encoded.decode('utf-8')  # bytes -> str
```

---

## type() vs isinstance()

```python
x = True

type(x) == bool    # True
type(x) == int     # False — type() checks EXACT type

isinstance(x, bool)  # True
isinstance(x, int)   # True — isinstance() respects inheritance
isinstance(x, (int, float))  # True — can check multiple types
```

**Rule of thumb:** Use `isinstance()` in production code. Use `type()` only when you need to distinguish exact types (rare). `isinstance()` respects subclassing and abstract base classes.

---

## Mutability Overview

```
Immutable:                     Mutable:
┌─────────────────────┐       ┌─────────────────────┐
│ int                 │       │ list                │
│ float               │       │ dict                │
│ bool                │       │ set                 │
│ str                 │       │ bytearray           │
│ tuple               │       │ user-defined objects│
│ frozenset           │       │ (by default)        │
│ bytes               │       │                     │
└─────────────────────┘       └─────────────────────┘
```

**Why immutability matters:**
- Immutable objects can be dictionary keys and set members (they're hashable)
- Immutable objects are inherently thread-safe
- CPython can intern/cache immutable objects (small ints, short strings)

```python
# Mutable default argument trap:
def append_to(element, lst=[]):  # BAD — default list is shared!
    lst.append(element)
    return lst

append_to(1)  # [1]
append_to(2)  # [1, 2] — NOT [2]! Same list object reused

# Fix: use None sentinel
def append_to(element, lst=None):
    if lst is None:
        lst = []
    lst.append(element)
    return lst
```

---

## Memory Sizes at a Glance

```python
import sys

sys.getsizeof(None)        # 16 bytes
sys.getsizeof(True)        # 28 bytes
sys.getsizeof(0)           # 28 bytes
sys.getsizeof(3.14)        # 24 bytes
sys.getsizeof("")          # 49 bytes (empty string — header overhead)
sys.getsizeof("hello")    # 54 bytes
sys.getsizeof([])          # 56 bytes (empty list)
sys.getsizeof(())          # 40 bytes (empty tuple)
sys.getsizeof({})          # 64 bytes (empty dict)
sys.getsizeof(set())       # 216 bytes (empty set — pre-allocated hash table)
```

**How to think about it:** Every Python object carries 16+ bytes of overhead (refcount + type pointer). A simple `int` is 28 bytes vs 8 bytes in C. This is the cost of "everything is an object."

---

## Practical Examples / LeetCode Patterns

### Type Checking in Interview Problems

```python
# Flatten a nested list (LeetCode 341 concept)
def flatten(nested):
    result = []
    for item in nested:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result

flatten([1, [2, [3, 4], 5], 6])  # [1, 2, 3, 4, 5, 6]
```

### Using bool as int (Counting Pattern)

```python
# Count negative numbers in a matrix (LeetCode 1351)
def countNegatives(grid):
    return sum(val < 0 for row in grid for val in row)
    # val < 0 returns True (=1) or False (=0), sum counts the Trues
```

### Float Precision in Binary Search

```python
# Sqrt(x) — LeetCode 69
def mySqrt(x):
    if x < 2:
        return x
    lo, hi = 1, x // 2
    while lo <= hi:
        mid = (lo + hi) // 2
        if mid * mid == x:
            return mid
        elif mid * mid < x:
            lo = mid + 1
        else:
            hi = mid - 1
    return hi

# Using int arithmetic avoids float precision issues entirely
```

---

## Key Takeaways

| Type | Internal | Size | Key Fact |
|---|---|---|---|
| int | PyLongObject, 30-bit digit array | 28+ bytes | Arbitrary precision, no overflow. Cached -5 to 256 |
| float | C double (IEEE 754) | 24 bytes | Always 64-bit. Use `math.isclose()` for comparison |
| bool | Subclass of int | 28 bytes | `True` is `1`, `False` is `0`. Singletons |
| None | NoneType singleton | 16 bytes | Always compare with `is`, never `==` |
| str | PEP 393 compact Unicode | 49+ bytes | Immutable. Variable internal encoding |
| Everything | PyObject with refcount + type ptr | 16+ bytes overhead | No primitives — everything is a heap object |
