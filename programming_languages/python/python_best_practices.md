# Python Best Practices (FAANG-Level Engineering)

**Related:** [Python as a language](python_as_a_language.md) | [data types](data_types_in_python.md) | [functions](functions_in_python.md) | [classes & OOP](classes_and_oop.md) | [concurrency](concurrency_in_python.md) | [iterators & generators](iterators_and_generators.md)

---

## 1. Performance Pitfalls

### 1.1 String Concatenation in Loops

```python
# BAD: O(n^2) — each += creates a new string object, copies all previous bytes
result = ""
for word in words:          # 10,000 words
    result += word + " "    # allocates a new string EVERY iteration

# GOOD: O(n) — list append is O(1) amortized, join allocates once
result = " ".join(words)

# GOOD: for complex building
parts = []
for word in words:
    parts.append(word.upper())  # O(1) amortized
result = " ".join(parts)        # single allocation at the end

# BEST: list comprehension + join
result = " ".join(word.upper() for word in words)
```

**Why += is O(n^2):** Strings are immutable. Each `+=` allocates a new string of size `len(result) + len(word)`, copies all bytes, then discards the old string. For n words of average length k: total bytes copied = k + 2k + 3k + ... + nk = O(n^2 * k).

### 1.2 list vs deque for Queue Operations

```python
from collections import deque

# BAD: pop(0) on a list is O(n) — shifts all elements left
queue = [1, 2, 3, 4, 5]
queue.pop(0)    # O(n) — shifts 4 elements

# GOOD: deque.popleft() is O(1)
queue = deque([1, 2, 3, 4, 5])
queue.popleft()  # O(1) — doubly-linked list under the hood
queue.append(6)  # O(1)
```

```
deque internal structure (doubly-linked list of fixed-size blocks):
┌────────┐   ┌────────┐   ┌────────┐
│ block  │──→│ block  │──→│ block  │
│ [a,b,c]│   │ [d,e,f]│   │ [g,h,i]│
│←───────│   │←───────│   │←───────│
└────────┘   └────────┘   └────────┘
Each block holds 64 items. O(1) append/pop on both ends.
O(n) random access (must traverse blocks).
```

### 1.3 Generator vs List for Large Data

```python
# BAD: creates entire list in memory
total = sum([x**2 for x in range(10_000_000)])  # ~80MB list of ints

# GOOD: generator expression — O(1) memory
total = sum(x**2 for x in range(10_000_000))    # computes one value at a time

# Rule: if you only need to iterate once, use a generator expression
# (no square brackets). If you need random access or len(), use a list.
```

### 1.4 Avoid Repeated Dictionary/Attribute Lookups in Loops

```python
# BAD: looks up math.sqrt on every iteration
import math
result = [math.sqrt(x) for x in range(100000)]

# GOOD: local reference — local variable lookup is faster than attribute lookup
sqrt = math.sqrt
result = [sqrt(x) for x in range(100000)]

# Why: LOAD_FAST (local) is O(1) array index.
# LOAD_ATTR (attribute) involves dict lookup on the module object.
# ~30% faster for tight loops.
```

### 1.5 in Operator: set vs list

```python
# BAD: O(n) linear scan
forbidden = ["spam", "eggs", "ham", ..., "10000th item"]
if word in forbidden:  # scans up to 10,000 items
    pass

# GOOD: O(1) hash lookup
forbidden = {"spam", "eggs", "ham", ..., "10000th item"}
if word in forbidden:  # single hash + probe
    pass

# For n=10,000: set lookup is ~500x faster
```

---

## 2. Memory Optimization

### 2.1 __slots__ for Many Instances

```python
# Without __slots__: each instance has a __dict__ (104+ bytes overhead)
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# With __slots__: no __dict__, ~50% less memory
class Point:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

# 1 million Points:
# Without slots: ~200MB
# With slots:    ~100MB
```

### 2.2 Generators for Streaming Data

```python
# BAD: reads entire file into memory
def read_large_file(path):
    with open(path) as f:
        return f.readlines()  # entire file in memory

# GOOD: generator — one line at a time
def read_large_file(path):
    with open(path) as f:
        for line in f:        # file objects are iterators
            yield line.strip()

# Process a 10GB log file with O(1) memory:
for line in read_large_file("/var/log/huge.log"):
    if "ERROR" in line:
        process(line)
```

### 2.3 sys.getsizeof and Memory Profiling

```python
import sys

# sys.getsizeof only shows SHALLOW size (not nested objects)
lst = [[1, 2, 3] for _ in range(1000)]
sys.getsizeof(lst)       # ~8056 bytes (just the pointer array)
# Actual memory: 8056 + 1000 * (list overhead + 3 * int overhead) ≈ 128KB

# For deep size, use pympler:
from pympler import asizeof
asizeof.asizeof(lst)     # ~128000 bytes (includes nested objects)
```

---

## 3. Profiling

### 3.1 cProfile — Function-Level CPU Profiling

```python
import cProfile

def main():
    data = [i**2 for i in range(100000)]
    sorted_data = sorted(data, reverse=True)
    return sum(sorted_data[:100])

cProfile.run('main()')
# Outputs: ncalls, tottime, percall, cumtime, filename:lineno(function)
```

```bash
# From command line:
python -m cProfile -s cumulative script.py

# Save to file and analyze:
python -m cProfile -o profile.prof script.py
python -m pstats profile.prof
```

### 3.2 line_profiler — Line-Level CPU Profiling

```python
# Install: pip install line_profiler
# Decorate function with @profile, then run:
# kernprof -l -v script.py

@profile
def slow_function():
    total = 0
    for i in range(100000):     # Line 3: 40% of time
        total += i ** 2          # Line 4: 60% of time
    return total
```

### 3.3 memory_profiler — Line-Level Memory Profiling

```python
# Install: pip install memory_profiler
# Run: python -m memory_profiler script.py

from memory_profiler import profile

@profile
def memory_hungry():
    a = [1] * (10**6)           # +7.6 MiB
    b = [2] * (2 * 10**7)      # +152.6 MiB
    del b                       # -152.6 MiB
    return a
```

---

## 4. Testing (pytest Patterns)

### 4.1 Basic pytest Structure

```python
# test_calculator.py
import pytest
from calculator import add, divide

def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        divide(1, 0)

# Parametrized tests (like Go table-driven tests):
@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (-1, -2, -3),
    (0, 0, 0),
    (100, -100, 0),
])
def test_add_parametrized(a, b, expected):
    assert add(a, b) == expected
```

### 4.2 Fixtures

```python
import pytest

@pytest.fixture
def database():
    """Setup and teardown for database connection."""
    db = connect_to_test_db()
    db.create_tables()
    yield db               # test runs here
    db.drop_tables()
    db.close()

@pytest.fixture
def sample_users(database):
    """Depends on database fixture — pytest resolves dependency graph."""
    users = [User("Alice"), User("Bob")]
    database.insert_many(users)
    return users

def test_find_user(database, sample_users):
    user = database.find("Alice")
    assert user.name == "Alice"
```

### 4.3 Mocking

```python
from unittest.mock import patch, MagicMock

def test_api_call():
    with patch('mymodule.requests.get') as mock_get:
        mock_get.return_value = MagicMock(
            status_code=200,
            json=lambda: {"key": "value"}
        )
        result = mymodule.fetch_data("https://api.example.com")
        assert result == {"key": "value"}
        mock_get.assert_called_once_with("https://api.example.com")

# patch as decorator:
@patch('mymodule.database')
def test_with_mock_db(mock_db):
    mock_db.query.return_value = [{"id": 1, "name": "Alice"}]
    result = mymodule.get_users()
    assert len(result) == 1
```

---

## 5. Type Checking (mypy)

```python
# Type hints catch bugs before runtime:
def process(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Run: mypy script.py
# mypy catches:
# - Wrong argument types
# - Missing return statements
# - Incorrect dict key/value types
# - None checks (Optional handling)
```

```bash
# mypy configuration in pyproject.toml:
# [tool.mypy]
# strict = true
# warn_return_any = true
# disallow_untyped_defs = true
```

---

## 6. Common Anti-Patterns

### 6.1 Mutable Default Arguments

```python
# BAD: default list is shared across all calls
def append_to(element, lst=[]):
    lst.append(element)
    return lst

append_to(1)  # [1]
append_to(2)  # [1, 2] — NOT [2]!

# FIX: use None sentinel
def append_to(element, lst=None):
    if lst is None:
        lst = []
    lst.append(element)
    return lst
```

### 6.2 Bare except

```python
# BAD: catches SystemExit, KeyboardInterrupt, GeneratorExit
try:
    do_something()
except:
    pass  # silently swallows EVERYTHING, including Ctrl+C

# GOOD: catch specific exceptions
try:
    do_something()
except (ValueError, TypeError) as e:
    logger.error(f"Invalid input: {e}")
except Exception as e:
    logger.error(f"Unexpected error: {e}")
    raise  # re-raise to not swallow the error
```

### 6.3 Using == for None/True/False

```python
# BAD: can be fooled by __eq__ override
if x == None:    # a class could define __eq__ to return True for None
    pass
if x == True:    # 1 == True is True!
    pass

# GOOD: use `is` for singletons
if x is None:
    pass
if x is True:    # only if you really need to distinguish True from 1
    pass
```

### 6.4 Not Using Context Managers

```python
# BAD: file may not be closed if exception occurs
f = open("data.txt")
data = f.read()
f.close()               # skipped if exception above

# GOOD: context manager guarantees cleanup
with open("data.txt") as f:
    data = f.read()      # f.close() called automatically, even on exception
```

---

## 7. Pythonic Idioms

```python
# Swap without temp variable:
a, b = b, a

# Ternary expression:
x = "even" if n % 2 == 0 else "odd"

# Chained comparison:
if 0 < x < 100:    # instead of: if x > 0 and x < 100

# Enumerate instead of range(len()):
for i, item in enumerate(items):
    print(f"{i}: {item}")

# zip for parallel iteration:
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# Dictionary unpacking merge (3.9+):
merged = {**defaults, **overrides}  # or: defaults | overrides

# Walrus operator (3.8+):
if (n := len(data)) > 10:
    print(f"Too many items: {n}")

# any() / all():
if any(score > 90 for score in scores):
    print("Someone got an A")

if all(score >= 60 for score in scores):
    print("Everyone passed")

# Collections unpacking:
first, *rest = [1, 2, 3, 4, 5]   # first=1, rest=[2,3,4,5]
first, *middle, last = [1, 2, 3, 4, 5]  # middle=[2,3,4]
```

---

## 8. Production Patterns

### 8.1 Structured Logging

```python
import logging
import json

# BAD: unstructured — impossible to parse at scale
logging.info(f"User {user_id} logged in from {ip}")

# GOOD: structured (JSON) — queryable in log aggregators
logger = logging.getLogger(__name__)
logger.info("user_login", extra={
    "user_id": user_id,
    "ip": ip,
    "latency_ms": latency_ms,
})

# Or use structlog:
import structlog
log = structlog.get_logger()
log.info("user_login", user_id=user_id, ip=ip)
```

### 8.2 Context Managers for Resource Management

```python
from contextlib import contextmanager

@contextmanager
def database_transaction(connection):
    cursor = connection.cursor()
    try:
        yield cursor
        connection.commit()
    except Exception:
        connection.rollback()
        raise
    finally:
        cursor.close()

with database_transaction(conn) as cursor:
    cursor.execute("INSERT INTO users VALUES (%s)", (name,))
    # auto-commits on success, auto-rollbacks on exception
```

### 8.3 Error Handling Strategy

```python
# Define domain exceptions:
class AppError(Exception):
    """Base exception for application errors."""
    pass

class NotFoundError(AppError):
    pass

class ValidationError(AppError):
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

# Use in business logic:
def get_user(user_id: str) -> User:
    user = db.find(user_id)
    if user is None:
        raise NotFoundError(f"User {user_id} not found")
    return user

# Handle at the boundary (API layer):
try:
    user = get_user(user_id)
except NotFoundError:
    return Response(status=404)
except ValidationError as e:
    return Response(status=400, body={"error": e.message})
except AppError:
    return Response(status=500)
```

---

## Quick Reference: Decision Table

| Situation | Do This | Not This |
|---|---|---|
| Build string in loop | `"".join(parts)` | `result += s` |
| Queue (FIFO) | `collections.deque` | `list.pop(0)` |
| Membership testing | `set` or `dict` | `x in list` |
| Large data iteration | Generator expression | List comprehension |
| Many instances of same class | `__slots__` | Regular `__dict__` |
| Close files/connections | `with` statement | Manual `.close()` |
| Compare with None | `is None` | `== None` |
| Default mutable argument | `None` sentinel | `def f(lst=[])` |
| Catch exceptions | Specific types | Bare `except:` |
| CPU-bound parallelism | `ProcessPoolExecutor` | `threading` |
| I/O-bound concurrency (many) | `asyncio` | Thread per connection |
| Memoize pure function | `@lru_cache` | Manual dict cache |
| Sort by custom key | `key=lambda x: ...` | Custom `__lt__` everywhere |
| Profile performance | `cProfile` first, then `line_profiler` | Guessing |
