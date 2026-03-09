# Functions in Python

**Related:** [Python as a language](python_as_a_language.md) | [classes & OOP](classes_and_oop.md) | [iterators & generators](iterators_and_generators.md) | [best practices](python_best_practices.md)

---

## Functions Are First-Class Objects

In Python, functions are **objects** — instances of the `function` type. They can be assigned to variables, stored in data structures, passed as arguments, and returned from other functions.

```python
def greet(name):
    return f"Hello, {name}"

# A function is an object:
type(greet)             # <class 'function'>
greet.__name__          # 'greet'
greet.__code__          # <code object greet at 0x...>
greet.__code__.co_varnames  # ('name',) — local variable names

# Assign to variable:
f = greet
f("Alice")              # "Hello, Alice"

# Store in data structure:
ops = {'+': lambda a, b: a + b, '-': lambda a, b: a - b}
ops['+'](3, 4)          # 7
```

### What's Happening in Memory

```
greet variable (in global namespace dict):
┌───────────────────────────┐
│ "greet" → ptr ────────────┼──→ function object:
└───────────────────────────┘    ┌───────────────────────┐
                                  │ ob_refcnt             │
                                  │ ob_type → function    │
                                  │ __name__: "greet"     │
                                  │ __code__ → ──────────┼──→ code object:
                                  │ __globals__ → ───────┼─→   (bytecode, consts,
                                  │ __defaults__: None    │      varnames, etc.)
                                  │ __closure__: None     │
                                  └───────────────────────┘
```

---

## *args and **kwargs

```python
def func(*args, **kwargs):
    print(args)     # tuple of positional args
    print(kwargs)   # dict of keyword args

func(1, 2, 3, x=10, y=20)
# args = (1, 2, 3)
# kwargs = {'x': 10, 'y': 20}
```

**What's happening under the hood:** CPython collects extra positional arguments into a new `tuple` (allocated on the heap). Extra keyword arguments go into a new `dict`. This means calling with `*args` or `**kwargs` always allocates.

```python
# Unpacking (spreading):
def add(a, b, c):
    return a + b + c

args = (1, 2, 3)
add(*args)          # unpacks tuple into positional args

kwargs = {'a': 1, 'b': 2, 'c': 3}
add(**kwargs)       # unpacks dict into keyword args
```

### Argument Passing Semantics

Python passes arguments by **"assignment"** — the parameter name is bound to the same object the caller passed. It's neither pure pass-by-value nor pass-by-reference:

```python
def modify(lst, num):
    lst.append(4)   # mutates the SAME list object (caller sees change)
    num = 99        # rebinds local name `num` (caller's variable unaffected)

my_list = [1, 2, 3]
my_num = 42
modify(my_list, my_num)
print(my_list)  # [1, 2, 3, 4] — modified!
print(my_num)   # 42 — unchanged
```

**How to think about it:** Parameters receive the same object reference. Mutating the object is visible to the caller. Rebinding the name (assignment) only affects the local scope.

---

## Scope Rules: LEGB

Python resolves names in four scopes, searched in order:

```
L — Local:     function body
E — Enclosing: enclosing function (for closures/nested functions)
G — Global:    module level
B — Built-in:  builtins module (print, len, etc.)

┌──────────────────────────────────────┐
│ Built-in (print, len, range, ...)    │
│  ┌──────────────────────────────┐    │
│  │ Global (module-level names)   │    │
│  │  ┌──────────────────────┐    │    │
│  │  │ Enclosing (outer fn)  │    │    │
│  │  │  ┌──────────────┐    │    │    │
│  │  │  │ Local (inner) │    │    │    │
│  │  │  └──────────────┘    │    │    │
│  │  └──────────────────────┘    │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)    # "local" — L

    inner()
    print(x)        # "enclosing" — E (inner's assignment didn't affect this)

outer()
print(x)            # "global" — G
```

### global and nonlocal

```python
count = 0

def increment():
    global count     # without this, `count = count + 1` creates a LOCAL variable
    count += 1

def outer():
    x = 10
    def inner():
        nonlocal x   # refers to the enclosing scope's x
        x += 1
    inner()
    print(x)         # 11
```

---

## Closures

A closure is a function that **captures variables from its enclosing scope**. The captured variables are stored in **cell objects**.

```python
def make_counter():
    count = 0               # captured by inner function

    def increment():
        nonlocal count
        count += 1
        return count

    return increment

counter = make_counter()
counter()   # 1
counter()   # 2
counter()   # 3
```

### What's Happening in Memory (Cell Objects)

```
make_counter() returns → increment function object:
┌───────────────────────────┐
│ __closure__: (cell_obj,)  │  ← tuple of cell objects
└──────────┼────────────────┘
           v
     ┌──────────────┐
     │ cell object   │
     │ cell_contents → 0 (then 1, 2, 3, ...)
     └──────────────┘

The cell object is an extra level of indirection.
Both the enclosing function and the closure point to the SAME cell.
When the closure modifies `count`, it modifies the cell's contents.
```

```python
counter.__closure__                    # (<cell at 0x...>,)
counter.__closure__[0].cell_contents   # current value of count
```

### Common Closure Trap (Loop Variables)

```python
# BUG: all functions see the SAME variable `i`
funcs = []
for i in range(5):
    funcs.append(lambda: i)

[f() for f in funcs]  # [4, 4, 4, 4, 4] — all see final value of i

# FIX: use default argument to capture current value
funcs = []
for i in range(5):
    funcs.append(lambda i=i: i)  # default arg evaluated at definition time

[f() for f in funcs]  # [0, 1, 2, 3, 4]
```

---

## Decorators

A decorator is **syntactic sugar for wrapping a function with another function** (a higher-order function).

```python
def timer(func):
    import time
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function(n):
    return sum(range(n))

# @timer is equivalent to:
# slow_function = timer(slow_function)
```

### Preserving Function Metadata

```python
from functools import wraps

def timer(func):
    @wraps(func)  # copies __name__, __doc__, __module__ from func to wrapper
    def wrapper(*args, **kwargs):
        # ... timing logic ...
        return func(*args, **kwargs)
    return wrapper

# Without @wraps: slow_function.__name__ would be "wrapper"
# With @wraps:    slow_function.__name__ is "slow_function"
```

### Decorator with Arguments

```python
def retry(max_attempts=3):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
        return wrapper
    return decorator

@retry(max_attempts=5)
def flaky_api_call():
    # ...
    pass

# Equivalent to: flaky_api_call = retry(max_attempts=5)(flaky_api_call)
# Three levels of nesting: retry() returns decorator, decorator(func) returns wrapper.
```

---

## Lambda Functions

Anonymous, single-expression functions. They return the expression's value automatically.

```python
square = lambda x: x ** 2
square(5)   # 25

# Commonly used with sorted, map, filter:
pairs = [(1, 'b'), (2, 'a'), (3, 'c')]
sorted(pairs, key=lambda p: p[1])  # [(2, 'a'), (1, 'b'), (3, 'c')]

# Lambda limitations:
# - Single expression only (no statements, no assignments)
# - No type annotations
# - Hard to debug (shows as <lambda> in tracebacks)
```

---

## Generators (yield)

A generator function uses `yield` instead of `return`. It produces values **lazily** — one at a time, suspending execution between yields.

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a         # suspends here, returns value
        a, b = b, a + b # resumes here on next()

gen = fibonacci()
next(gen)   # 0
next(gen)   # 1
next(gen)   # 1
next(gen)   # 2
```

### Generator Frame Suspension

```
Generator object:
┌─────────────────────────────┐
│ gi_frame → frame object     │  <-- suspended execution frame
│ gi_code  → code object      │  <-- bytecode
│ gi_running: False            │
└─────────────────────────────┘

The frame object preserves ALL local variables between yields.
This is why generators are memory-efficient: they don't compute
all values upfront — they compute on demand.
```

**Related:** [iterators & generators](iterators_and_generators.md) for deep dive on generator protocol, `send()`, async generators.

---

## Type Hints

Type hints are **annotations** — they have NO runtime effect in CPython. They are used by static type checkers (mypy, pyright).

```python
def greet(name: str) -> str:
    return f"Hello, {name}"

# Complex types:
from typing import Optional, Union

def find(lst: list[int], target: int) -> Optional[int]:
    """Returns index or None."""
    try:
        return lst.index(target)
    except ValueError:
        return None

# Python 3.10+: use | instead of Union
def process(data: str | bytes) -> str:
    ...
```

---

## functools Utilities

### lru_cache (Memoization)

```python
from functools import lru_cache

@lru_cache(maxsize=128)  # maxsize=None for unlimited cache
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

fib(100)           # instant — O(n) instead of O(2^n)
fib.cache_info()   # CacheInfo(hits=98, misses=101, maxsize=128, currsize=101)
fib.cache_clear()  # evict all cached results
```

**How lru_cache works internally:** It wraps the function with a dictionary cache. The key is the tuple of arguments. It uses a doubly-linked list for LRU eviction when `maxsize` is reached. Thread-safe (uses a lock).

### partial (Partial Application)

```python
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

square(5)   # 25
cube(5)     # 125
```

---

## Practical Examples / LeetCode Patterns

### Recursion with Memoization (LeetCode 70: Climbing Stairs)

```python
from functools import lru_cache

def climbStairs(n):
    @lru_cache(maxsize=None)
    def dp(i):
        if i <= 2:
            return i
        return dp(i - 1) + dp(i - 2)
    return dp(n)

# The decorator turns O(2^n) recursion into O(n) with O(n) space.
# Each unique argument is computed once, then cached in a dict.
```

### Closures as Comparators (LeetCode 56: Merge Intervals)

```python
def merge(intervals):
    intervals.sort(key=lambda x: x[0])  # closure captures nothing special here
    merged = [intervals[0]]

    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])

    return merged
```

### Decorator Pattern (Rate Limiting)

```python
from functools import wraps
import time

def rate_limit(calls_per_second):
    min_interval = 1.0 / calls_per_second
    def decorator(func):
        last_call = [0.0]  # mutable container for closure
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_call[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            last_call[0] = time.time()
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(calls_per_second=5)
def api_request(url):
    pass  # ... make request ...
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| First-class | Functions are objects with `__code__`, `__closure__`, `__name__`, etc. |
| Argument passing | By assignment — mutable objects can be modified by callee, rebinding is local |
| Scope | LEGB: Local, Enclosing, Global, Built-in. Use `nonlocal` / `global` to modify outer |
| Closures | Captured via cell objects (extra indirection). Watch out for loop variable trap |
| Decorators | Syntactic sugar: `@dec` = `func = dec(func)`. Use `@wraps` to preserve metadata |
| Generators | `yield` suspends frame. Lazy evaluation. Memory-efficient for large sequences |
| lru_cache | Dict-based memoization. Turns O(2^n) recursion into O(n). Thread-safe |
| Type hints | No runtime effect. For mypy/pyright. Use `list[int]`, `Optional[T]`, `T | None` |
