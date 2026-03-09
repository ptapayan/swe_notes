# Dictionaries and Sets in Python

**Related:** [data types](data_types_in_python.md) | [lists & tuples](lists_and_tuples.md) | [functions](functions_in_python.md) | [best practices](python_best_practices.md)

---

## dict — Hash Table with Insertion Order

Python `dict` is a **hash table** using open addressing with insertion-order preservation (guaranteed since Python 3.7, implementation detail since 3.6).

### Internal Structure (Compact Dict, Python 3.6+)

Before 3.6, dicts used a single sparse hash table. Since 3.6, CPython uses a **compact dict** with two arrays:

```
Compact dict architecture:

1. Hash index table (sparse — many empty slots):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ -1 │  2 │ -1 │  0 │ -1 │  1 │ -1 │ -1 │
└────┴────┴────┴────┴────┴────┴────┴────┘
  Indices into the entries array. -1 = empty slot.
  Size is always a power of 2 (here: 8).

2. Entries array (dense — packed, insertion order):
┌───────────────────────────────────┐
│ [0] hash=..., key="alice", val=95 │  ← inserted first
│ [1] hash=..., key="bob",  val=87  │  ← inserted second
│ [2] hash=..., key="carol", val=92 │  ← inserted third
└───────────────────────────────────┘
  Entries are stored in insertion order.
  No gaps — this is why iteration is ordered.
```

**Why compact dicts are better:**
- **Memory:** The index table stores small integers (1-2 bytes each for small dicts) instead of full 3-tuple entries. Saves 25-35% memory.
- **Ordering:** Entries array naturally preserves insertion order — no extra data structure needed.
- **Cache locality:** Dense entries array means iteration hits contiguous memory.

### Hash Collision Resolution (Open Addressing)

Python uses **open addressing with perturbation**. On collision, it probes the next slot using:

```
j = ((5 * j) + 1 + perturb) % table_size
perturb >>= 5
```

This is NOT simple linear probing — the perturbation mixes in higher bits of the hash, giving better distribution. As `perturb` shifts to zero, it degrades to `(5*j + 1) % size`, which visits every slot in a power-of-2 table.

### Load Factor and Resizing

The hash table resizes when it becomes **2/3 full** (load factor ~0.66). On resize, the table size doubles (or quadruples for small dicts).

```python
# Pre-sizing hint (Python 3.12+):
# No direct API, but dict comprehensions and dict.fromkeys pre-allocate.
# For known sizes, build all at once:
d = {k: v for k, v in zip(keys, values)}  # single allocation if possible
```

---

## dict Operations — Time Complexity

| Operation | Average | Worst | Notes |
|---|---|---|---|
| `d[key]` | O(1) | O(n) | Worst case: all keys collide (pathological hash) |
| `d[key] = val` | O(1) | O(n) | May trigger resize (amortized O(1)) |
| `del d[key]` | O(1) | O(n) | Marks entry as deleted (tombstone) |
| `key in d` | O(1) | O(n) | Same as lookup |
| `len(d)` | O(1) | O(1) | Stored in `ma_used` field |
| `iter(d)` | O(n) | O(n) | Iterates dense entries array |
| `d.keys()` | O(1) | O(1) | Returns a view object (no copy) |
| `d.values()` | O(1) | O(1) | Returns a view object |
| `d.items()` | O(1) | O(1) | Returns a view object |

---

## Creating Dicts

```python
# Literal (most common)
d = {"alice": 95, "bob": 87}

# dict() constructor
d = dict(alice=95, bob=87)          # keyword args (keys must be valid identifiers)
d = dict([("alice", 95), ("bob", 87)])  # from iterable of pairs

# Dict comprehension
d = {k: v for k, v in zip(names, scores)}

# fromkeys (all values same)
d = dict.fromkeys(["a", "b", "c"], 0)  # {"a": 0, "b": 0, "c": 0}
```

---

## Useful dict Methods

```python
d = {"alice": 95, "bob": 87}

# get() — never raises KeyError
d.get("charlie")       # None
d.get("charlie", 0)    # 0 (default)

# setdefault() — get or set if missing (atomic for the dict)
d.setdefault("charlie", 90)  # sets and returns 90
d.setdefault("alice", 100)   # returns 95 (already exists, NOT overwritten)

# pop() — remove and return
d.pop("bob")           # 87 (removed)
d.pop("missing", -1)   # -1 (default, no KeyError)

# update() — merge
d.update({"dave": 88, "alice": 99})  # alice overwritten to 99

# Merge operators (Python 3.9+)
merged = d1 | d2       # new dict, d2 wins on conflicts
d1 |= d2               # in-place merge
```

---

## collections Module — Specialized Dicts

### defaultdict

```python
from collections import defaultdict

# Automatically creates missing keys with a factory function
freq = defaultdict(int)        # missing keys default to 0
freq["apple"] += 1             # no KeyError — auto-creates with int() = 0

groups = defaultdict(list)     # missing keys default to []
groups["fruit"].append("apple")  # auto-creates empty list

# What's happening: defaultdict overrides __missing__(). When __getitem__
# doesn't find a key, it calls self.default_factory() and inserts the result.
```

### Counter

```python
from collections import Counter

words = ["go", "is", "go", "great", "go"]
freq = Counter(words)          # Counter({"go": 3, "is": 1, "great": 1})

freq.most_common(2)            # [("go", 3), ("is", 1)]
freq["python"]                 # 0 (missing keys return 0, not KeyError)

# Counter arithmetic:
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
c1 + c2                        # Counter(a=4, b=3)
c1 - c2                        # Counter(a=2) — drops zero/negative
c1 & c2                        # Counter(a=1, b=1) — min
c1 | c2                        # Counter(a=3, b=2) — max
```

### OrderedDict (Pre-3.7 Legacy)

Since Python 3.7, regular `dict` preserves insertion order. `OrderedDict` still has niche uses:
- `move_to_end(key, last=True/False)` — O(1) reordering
- Equality comparison considers order (`OrderedDict` vs `OrderedDict`)
- LRU cache implementation

---

## set — Hash Table Without Values

A `set` is internally a hash table where each entry stores only a key (no value). Same open-addressing collision resolution as dict.

```python
s = {1, 2, 3, 4, 5}
type(s)       # <class 'set'>

# Empty set — {} creates a dict, not a set!
s = set()     # correct
d = {}        # this is a dict

# frozenset — immutable set (hashable, can be dict key or set member)
fs = frozenset([1, 2, 3])
```

### Set Operations

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b       # {1, 2, 3, 4, 5, 6}  — union
a & b       # {3, 4}              — intersection
a - b       # {1, 2}              — difference
a ^ b       # {1, 2, 5, 6}        — symmetric difference

a.issubset(b)      # False
a.issuperset(b)    # False
a.isdisjoint(b)    # False (they share elements)

# All set operations: O(min(len(a), len(b))) for &, O(len(a) + len(b)) for |
```

### Set Time Complexity

| Operation | Average | Notes |
|---|---|---|
| `x in s` | O(1) | Hash lookup |
| `s.add(x)` | O(1) | |
| `s.remove(x)` | O(1) | Raises KeyError if missing |
| `s.discard(x)` | O(1) | No error if missing |
| `a & b` | O(min(n,m)) | Iterates smaller set |
| `a \| b` | O(n + m) | |

---

## Dict Comprehensions and Set Comprehensions

```python
# Dict comprehension
squares = {x: x**2 for x in range(10)}
# {0: 0, 1: 1, 2: 4, 3: 9, ...}

# Invert a dict
inverted = {v: k for k, v in original.items()}

# Set comprehension
unique_lengths = {len(word) for word in words}

# Conditional comprehension
passing = {name: score for name, score in grades.items() if score >= 70}
```

---

## Performance Characteristics

### dict vs list for Lookup

```python
# List: O(n) lookup
names = ["alice", "bob", "charlie", ..., "zara"]
"zara" in names  # scans ALL elements

# Dict: O(1) lookup
name_set = {"alice", "bob", "charlie", ..., "zara"}
"zara" in name_set  # single hash + probe

# At n=10,000: dict lookup is ~500x faster than list scan
```

### Hashability Requirements

Dict keys and set members must be **hashable** — they must have a `__hash__()` method that returns a consistent integer, and `__eq__()` for collision resolution.

```python
# Hashable (immutable types): int, float, str, tuple (if contents hashable), frozenset
# NOT hashable: list, dict, set

# Custom hashable class:
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        return hash((self.x, self.y))

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
```

---

## Common Patterns

### Frequency Counting

```python
# Using Counter (cleanest):
from collections import Counter
freq = Counter(items)

# Using dict.get():
freq = {}
for item in items:
    freq[item] = freq.get(item, 0) + 1

# Using defaultdict:
from collections import defaultdict
freq = defaultdict(int)
for item in items:
    freq[item] += 1
```

### Grouping

```python
from collections import defaultdict

# Group words by first letter
words = ["apple", "ant", "banana", "bat", "cherry"]
groups = defaultdict(list)
for word in words:
    groups[word[0]].append(word)
# {'a': ['apple', 'ant'], 'b': ['banana', 'bat'], 'c': ['cherry']}
```

### Memoization / Caching

```python
# Manual memoization with dict:
cache = {}
def fib(n):
    if n in cache:
        return cache[n]
    if n <= 1:
        return n
    cache[n] = fib(n - 1) + fib(n - 2)
    return cache[n]

# Better: use functools.lru_cache (decorator)
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)
```

---

## Practical Examples / LeetCode Patterns

### Two Sum (LeetCode 1)

```python
def twoSum(nums, target):
    seen = {}  # val -> index
    for i, n in enumerate(nums):
        complement = target - n
        if complement in seen:
            return [seen[complement], i]
        seen[n] = i
    return []

# What's happening: dict lookup `complement in seen` is O(1).
# Single pass: O(n) total, O(n) space.
# Pre-allocation not needed — Python dict handles growth automatically.
```

### Group Anagrams (LeetCode 49)

```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # tuple is hashable, list is not
        groups[key].append(s)
    return list(groups.values())

# Alternative with counting key (O(k) vs O(k log k)):
def groupAnagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        groups[tuple(count)].append(s)
    return list(groups.values())

# Why tuple(count): lists are unhashable, but tuples of ints are.
# The count-based key avoids O(k log k) sorting per string.
```

### LRU Cache (LeetCode 146)

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # O(1) — marks as most recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # O(1) — removes least recently used

# OrderedDict.move_to_end() and popitem() are O(1) because it maintains
# a doubly-linked list internally alongside the hash table.
# This is one of the rare cases where OrderedDict is still useful post-3.7.
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| dict internal | Compact hash table: sparse index array + dense entries array. Ordered since 3.7 |
| Collision resolution | Open addressing with perturbation (not linear probing) |
| Load factor | Resizes at 2/3 full. Table size always power of 2 |
| set internal | Same hash table as dict, but no values |
| Hashability | Keys must be hashable (immutable). Use `tuple` not `list` for composite keys |
| defaultdict | Avoids KeyError — calls factory function for missing keys |
| Counter | Frequency counting in one line. Supports arithmetic |
| Lookup | O(1) average. For membership testing, always prefer set/dict over list |
| Views | `d.keys()`, `d.values()`, `d.items()` return views (no copy). Dynamic |
