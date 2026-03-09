# Iterators and Generators in Python

**Related:** [functions](functions_in_python.md) | [lists & tuples](lists_and_tuples.md) | [classes & OOP](classes_and_oop.md) | [best practices](python_best_practices.md)

---

## Iterator Protocol

An **iterator** is any object that implements two methods:
- `__iter__()` — returns the iterator itself
- `__next__()` — returns the next value, raises `StopIteration` when exhausted

An **iterable** is any object with `__iter__()` that returns an iterator. Lists, tuples, dicts, strings, files — all are iterables.

```python
# Under the hood, a for loop does this:
for x in [1, 2, 3]:
    print(x)

# Is equivalent to:
iterator = iter([1, 2, 3])    # calls list.__iter__(), returns list_iterator
while True:
    try:
        x = next(iterator)     # calls iterator.__next__()
        print(x)
    except StopIteration:
        break
```

```
Iterable vs Iterator:

list [1, 2, 3] (iterable):
┌──────────────────────┐
│ __iter__() → returns │──→ list_iterator:
│            a NEW     │    ┌──────────────────────┐
│            iterator  │    │ __iter__() → self     │
└──────────────────────┘    │ __next__() → 1, 2, 3 │
                            │ index: 0 (tracks pos) │
                            └──────────────────────┘

Key distinction:
- Iterable: you can call iter() on it multiple times (get fresh iterators)
- Iterator: you can call next() on it (it has state, it's consumed)
- All iterators are iterables (they return self from __iter__)
- Not all iterables are iterators (lists don't have __next__)
```

### Custom Iterator

```python
class CountDown:
    """Iterator that counts down from n to 1."""
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return self         # iterator IS the iterable

    def __next__(self):
        if self.n <= 0:
            raise StopIteration
        self.n -= 1
        return self.n + 1

list(CountDown(5))  # [5, 4, 3, 2, 1]
```

---

## Generator Functions

A generator function uses `yield` instead of `return`. When called, it returns a **generator object** (an iterator) without executing the body. The body executes lazily — only when `next()` is called.

```python
def count_up(n):
    i = 0
    while i < n:
        yield i        # suspends here, returns i
        i += 1         # resumes here on next next() call

gen = count_up(3)
type(gen)              # <class 'generator'>
next(gen)              # 0
next(gen)              # 1
next(gen)              # 2
next(gen)              # StopIteration
```

### Generator State Machine

When a generator function is called, CPython creates a **generator object** that wraps a suspended **frame object**:

```
Generator object:
┌────────────────────────────────┐
│ gi_frame → suspended frame     │  <-- local variables, instruction pointer
│ gi_code  → code object         │  <-- bytecode for the function
│ gi_running: False              │  <-- True only while executing
│ gi_yieldfrom: None             │  <-- for yield from delegation
└────────────────────────────────┘

State transitions:
  CREATED ──next()──→ RUNNING ──yield──→ SUSPENDED
     ↑                                      │
     └──────────next()──────────────────────┘
                                            │
  SUSPENDED ──next()──→ RUNNING ──return──→ CLOSED (StopIteration)
```

```python
import inspect

def example():
    yield 1
    yield 2

gen = example()
inspect.getgeneratorstate(gen)  # 'GEN_CREATED'
next(gen)
inspect.getgeneratorstate(gen)  # 'GEN_SUSPENDED'
next(gen)
inspect.getgeneratorstate(gen)  # 'GEN_SUSPENDED'
next(gen, None)                 # returns None instead of raising StopIteration
inspect.getgeneratorstate(gen)  # 'GEN_CLOSED'
```

### send() — Bidirectional Communication

`send(value)` resumes the generator and makes `yield` return `value` inside the generator.

```python
def accumulator():
    total = 0
    while True:
        value = yield total   # yield sends total OUT, receives value IN
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)            # 0 — priming the generator (must call next() first)
gen.send(10)         # 10 (total = 0 + 10)
gen.send(20)         # 30 (total = 10 + 20)
gen.send(5)          # 35 (total = 30 + 5)
```

### throw() and close()

```python
def safe_gen():
    try:
        while True:
            yield "value"
    except GeneratorExit:
        print("Generator closed — cleanup here")
    except ValueError as e:
        print(f"Got error: {e}")
        yield "recovered"

gen = safe_gen()
next(gen)               # "value"
gen.throw(ValueError, "bad input")  # injects exception at yield point
gen.close()             # sends GeneratorExit, generator must not yield after this
```

---

## Generator Expressions vs List Comprehensions

```python
# List comprehension — EAGER: builds entire list in memory
squares_list = [x**2 for x in range(1_000_000)]  # ~8MB in memory

# Generator expression — LAZY: computes one value at a time
squares_gen = (x**2 for x in range(1_000_000))   # ~120 bytes (just the generator object)

# Both work the same way in for loops:
for sq in squares_gen:
    process(sq)
```

**When to use which:**

| Use Generator Expression | Use List Comprehension |
|---|---|
| Single iteration over data | Need random access (`lst[i]`) |
| Very large or infinite sequences | Need `len()` |
| Piping into `sum()`, `max()`, `any()` | Need to iterate multiple times |
| Memory-constrained environments | Need to pass to API expecting a list |

```python
# Generator expressions work naturally with aggregation functions:
total = sum(x**2 for x in range(1_000_000))        # O(1) memory
largest = max(len(line) for line in open("file.txt"))  # O(1) memory
has_even = any(x % 2 == 0 for x in numbers)        # short-circuits!
```

---

## yield from — Generator Delegation

`yield from` delegates iteration to a sub-generator, passing through `send()`, `throw()`, and `close()`.

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # delegate to recursive call
        else:
            yield item

list(flatten([1, [2, [3, 4], 5], 6]))  # [1, 2, 3, 4, 5, 6]

# Without yield from (equivalent but verbose):
def flatten_manual(nested):
    for item in nested:
        if isinstance(item, list):
            for sub_item in flatten_manual(item):
                yield sub_item
        else:
            yield item
```

**Why `yield from` matters beyond syntax sugar:** It correctly forwards `send()` and `throw()` to the sub-generator. A manual `for` loop with `yield` does not.

---

## itertools — Standard Library Iteration Tools

All `itertools` functions return **lazy iterators** — O(1) memory regardless of input size.

### Infinite Iterators

```python
import itertools

# count(start, step) — infinite counter
itertools.count(10, 2)          # 10, 12, 14, 16, ...

# cycle(iterable) — infinite loop over iterable
itertools.cycle([1, 2, 3])     # 1, 2, 3, 1, 2, 3, 1, ...

# repeat(val, n) — repeat value n times (or infinitely)
itertools.repeat(0, 5)         # 0, 0, 0, 0, 0
```

### Combinatorics

```python
# product — Cartesian product
list(itertools.product([1, 2], ['a', 'b']))
# [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

# permutations
list(itertools.permutations([1, 2, 3], 2))
# [(1, 2), (1, 3), (2, 1), (2, 3), (3, 1), (3, 2)]

# combinations
list(itertools.combinations([1, 2, 3, 4], 2))
# [(1, 2), (1, 3), (1, 4), (2, 3), (2, 4), (3, 4)]

# combinations_with_replacement
list(itertools.combinations_with_replacement([1, 2, 3], 2))
# [(1, 1), (1, 2), (1, 3), (2, 2), (2, 3), (3, 3)]
```

### Selection and Grouping

```python
# islice — slice an iterator (cannot slice iterators with [])
itertools.islice(range(100), 10, 20)     # elements 10-19

# chain — concatenate iterables
itertools.chain([1, 2], [3, 4], [5])     # 1, 2, 3, 4, 5

# groupby — group consecutive elements (requires sorted input!)
data = [("a", 1), ("a", 2), ("b", 3), ("b", 4)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# a [('a', 1), ('a', 2)]
# b [('b', 3), ('b', 4)]

# accumulate — running totals
list(itertools.accumulate([1, 2, 3, 4]))  # [1, 3, 6, 10]

# starmap — map with unpacking
list(itertools.starmap(pow, [(2, 3), (3, 2), (10, 3)]))  # [8, 9, 1000]
```

---

## When Generators Save Memory

```python
# Example: processing a 10GB log file

# BAD: loads entire file (~10GB memory)
lines = open("huge.log").readlines()
errors = [line for line in lines if "ERROR" in line]

# GOOD: generator pipeline (~0 memory)
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip()

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line:
            yield line

def extract_timestamps(lines):
    for line in lines:
        yield line[:23]  # first 23 chars = timestamp

# Pipeline: each stage is lazy, data flows one line at a time
pipeline = extract_timestamps(filter_errors(read_lines("huge.log")))
for timestamp in pipeline:
    print(timestamp)

# At any point, only ONE line is in memory.
```

---

## Async Generators (Python 3.6+)

Async generators combine `async` with `yield` — they produce values lazily in an async context.

```python
import asyncio

async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.01)  # simulate async work
        yield i

async def main():
    async for value in async_range(5):
        print(value)

asyncio.run(main())

# Async generator expressions:
async def main():
    values = [v async for v in async_range(10)]  # async list comprehension
    total = sum([v async for v in async_range(10)])
```

---

## Practical Examples / LeetCode Patterns

### Flatten Nested List Iterator (LeetCode 341)

```python
class NestedIterator:
    def __init__(self, nestedList):
        self._gen = self._flatten(nestedList)
        self._next = None
        self._has_next = False
        self._advance()

    def _flatten(self, nested):
        for item in nested:
            if item.isInteger():
                yield item.getInteger()
            else:
                yield from self._flatten(item.getList())

    def _advance(self):
        try:
            self._next = next(self._gen)
            self._has_next = True
        except StopIteration:
            self._has_next = False

    def next(self):
        val = self._next
        self._advance()
        return val

    def hasNext(self):
        return self._has_next

# yield from does the heavy lifting — recursive flattening with O(1)
# memory per level of nesting.
```

### Subsets Using itertools (LeetCode 78)

```python
from itertools import combinations

def subsets(nums):
    result = []
    for size in range(len(nums) + 1):
        result.extend(combinations(nums, size))
    return [list(combo) for combo in result]

# Or with bit manipulation (no itertools):
def subsets(nums):
    result = []
    for mask in range(1 << len(nums)):
        subset = [nums[i] for i in range(len(nums)) if mask & (1 << i)]
        result.append(subset)
    return result
```

### Generator-Based BFS (Memory Efficient Tree Traversal)

```python
def bfs_generator(root):
    """BFS traversal as a generator — yields one node at a time."""
    if not root:
        return
    from collections import deque
    queue = deque([root])
    while queue:
        node = queue.popleft()
        yield node.val
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

# Usage: lazy traversal, can stop early
for val in bfs_generator(root):
    if val == target:
        print("Found!")
        break  # generator is cleaned up, no wasted work
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Iterator protocol | `__iter__()` + `__next__()`. Raise `StopIteration` when done |
| Generator function | `yield` suspends frame. Returns generator object. Lazy evaluation |
| Generator expression | `(x for x in ...)` — lazy, O(1) memory. No brackets = generator |
| send() | Bidirectional — `value = yield result`. Must prime with `next()` first |
| yield from | Delegates to sub-generator. Forwards send/throw/close correctly |
| itertools | Lazy combinatorics: `product`, `permutations`, `combinations`. All O(1) memory |
| Memory savings | Generator pipeline: chain generators, data flows one item at a time |
| Async generators | `async def` + `yield`. Use `async for` to consume |
| Rule of thumb | If iterating once: generator. If need len/index/multiple passes: list |
