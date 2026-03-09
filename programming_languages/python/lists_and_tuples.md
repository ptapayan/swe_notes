# Lists and Tuples in Python

**Related:** [data types](data_types_in_python.md) | [dicts & sets](dictionaries_and_sets.md) | [iterators & generators](iterators_and_generators.md) | [best practices](python_best_practices.md)

---

## List — Dynamic Array of Pointers

A Python `list` is a **dynamic array of pointers** to PyObjects. It does NOT store the objects inline — it stores pointers to them.

### Internal Structure (PyListObject)

```
PyListObject:
┌──────────────────────────┐
│ ob_refcnt (8B)           │
│ ob_type → list (8B)      │
│ ob_size: 5 (8B)          │  <-- current length (number of elements)
│ allocated: 8 (8B)        │  <-- capacity (allocated slots)
│ ob_item → ────────────┐  │  <-- pointer to array of PyObject* pointers
└────────────────────────│──┘
                         v
Heap:   ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
        │ ptr │ ptr │ ptr │ ptr │ ptr │ --- │ --- │ --- │
        │  ↓  │  ↓  │  ↓  │  ↓  │  ↓  │     │     │     │
        └──┼──┴──┼──┴──┼──┴──┼──┴──┼──┴─────┴─────┴─────┘
           v     v     v     v     v      unused capacity
          10    20    30    40    50      (allocated=8, size=5)

Each pointer is 8 bytes on 64-bit systems.
The actual int/str/etc. objects live elsewhere on the heap.
```

**Key insight:** A list of 1000 ints doesn't store 1000 ints contiguously. It stores 1000 pointers (8KB), each pointing to a separate `PyLongObject` (28 bytes each). Total: ~36KB vs 8KB for a C array of `int64_t`.

### Over-Allocation Growth Formula

When `append()` exceeds capacity, CPython allocates a new, larger array and copies the pointers:

```python
# CPython growth formula (listobject.c):
# new_allocated = (newsize >> 3) + (3 if newsize < 9 else 6) + newsize
# This gives roughly: 0, 4, 8, 16, 24, 32, 40, 52, 64, 76, ...
```

```python
import sys

lst = []
prev_size = sys.getsizeof(lst)
for i in range(20):
    lst.append(i)
    new_size = sys.getsizeof(lst)
    if new_size != prev_size:
        print(f"len={len(lst):2d}  capacity_change  size={new_size} bytes")
        prev_size = new_size

# len= 1  capacity_change  size=88 bytes    (allocated 4 slots)
# len= 5  capacity_change  size=120 bytes   (allocated 8 slots)
# len= 9  capacity_change  size=184 bytes   (allocated 16 slots)
# len=17  capacity_change  size=248 bytes   (allocated 24 slots)
```

**How to think about it:** Growth is roughly 12.5% over-allocation, which is less aggressive than Go's doubling. This means more frequent reallocations but less wasted memory.

### List Operations — Time Complexity

| Operation | Time | What Happens |
|---|---|---|
| `lst[i]` | O(1) | Pointer arithmetic: `ob_item + i * 8` |
| `lst.append(x)` | O(1) amortized | May trigger realloc + copy |
| `lst.insert(0, x)` | O(n) | Shifts ALL pointers right by one slot |
| `lst.pop()` | O(1) | Decrements size, returns last pointer |
| `lst.pop(0)` | O(n) | Shifts ALL pointers left. Use `collections.deque` instead |
| `lst[i] = x` | O(1) | Overwrites one pointer |
| `x in lst` | O(n) | Linear scan |
| `lst.sort()` | O(n log n) | Timsort (stable, adaptive merge sort) |
| `len(lst)` | O(1) | Returns `ob_size` field directly |

---

## Tuple — Immutable Fixed-Size Array

A tuple is like a list, but **immutable** and **fixed-size**. The pointer array is embedded directly in the tuple object (no separate allocation).

### Internal Structure (PyTupleObject)

```
PyTupleObject:
┌──────────────────────────┐
│ ob_refcnt (8B)           │
│ ob_type → tuple (8B)     │
│ ob_size: 3 (8B)          │  <-- number of elements
│ ob_item[0] → ptr ────────┼──→ PyObject (value 1)
│ ob_item[1] → ptr ────────┼──→ PyObject (value 2)
│ ob_item[2] → ptr ────────┼──→ PyObject (value 3)
└──────────────────────────┘

No separate allocation for the pointer array — it's inline.
No allocated/capacity fields — size is fixed at creation.
```

### Tuple Caching (Free List)

CPython caches **tuples of length 0-19** in a free list. When a small tuple is deallocated, its memory is reused for the next tuple of the same size, avoiding a `malloc` call.

The empty tuple `()` is a **singleton** — there is exactly one in the interpreter.

```python
a = ()
b = ()
a is b  # True — same object

a = (1, 2, 3)
b = (1, 2, 3)
a is b  # False in general (but may be True for compile-time constants)
```

---

## List vs Tuple — Memory and Performance

```python
import sys

sys.getsizeof([1, 2, 3])     # 120 bytes (list header + over-allocated pointer array)
sys.getsizeof((1, 2, 3))     # 64 bytes  (tuple header + inline pointer array)
```

**Why tuples are smaller:**
1. No over-allocation (exact size)
2. Pointer array is inline (no extra pointer + allocation)
3. No `allocated` field (saves 8 bytes)

**Why tuples are faster:**
1. Creation: tuples can come from the free list (no malloc)
2. Access: identical O(1) pointer arithmetic
3. Iteration: slightly faster due to better cache locality (inline array)
4. Hashing: tuples are hashable (can be dict keys), lists are not

```python
# Benchmark creation:
# Tuple: ~50ns
# List:  ~80ns (needs separate pointer array allocation)

# Use tuple when:
# - Data is fixed/immutable (coordinates, RGB, config)
# - Dict keys or set members
# - Returning multiple values from a function
# - Storing records (named tuples)

# Use list when:
# - Data changes (append, remove, sort)
# - Building up results iteratively
# - Stack/queue behavior
```

---

## Slicing (Creates Copies)

Slicing a list creates a **new list** with a **new pointer array**, but the pointers reference the **same objects**.

```python
original = [1, 2, 3, 4, 5]
sliced = original[1:4]       # [2, 3, 4] — shallow copy

sliced[0] = 99
print(original)  # [1, 2, 3, 4, 5] — unaffected (different pointer array)
```

```
original.ob_item →  [ptr→1, ptr→2, ptr→3, ptr→4, ptr→5]
sliced.ob_item   →  [ptr→2, ptr→3, ptr→4]  ← NEW array, same objects

Modifying sliced[0] just changes where the pointer points.
The original's pointer array is separate.
```

**Contrast with Go:** Go slices share the backing array. Python slices always copy the pointer array. This is safer but uses more memory.

```python
# Slice tricks:
lst[::-1]      # reverse copy
lst[::2]       # every other element
lst[:]         # shallow copy (same as lst.copy() or list(lst))
```

---

## List Comprehensions

List comprehensions compile to specialized bytecode that is faster than equivalent `for` loops.

```python
# Comprehension (preferred):
squares = [x**2 for x in range(10)]

# Equivalent loop (slower):
squares = []
for x in range(10):
    squares.append(x)  # each append is a method lookup + call
```

### Bytecode Comparison

```python
import dis

# Comprehension compiles to LIST_APPEND opcode (fast C-level append)
dis.dis("[x**2 for x in range(10)]")
# ... LIST_APPEND ...

# For loop uses LOAD_ATTR + CALL_FUNCTION for .append() (slower)
```

**How to think about it:** List comprehensions avoid the overhead of Python-level method dispatch for each append. The `LIST_APPEND` bytecode directly calls the C function.

### Nested Comprehensions

```python
# Flatten a 2D matrix:
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Read order: outer loop first, then inner loop
# Equivalent to:
flat = []
for row in matrix:
    for x in row:
        flat.append(x)
```

---

## Practical Examples / LeetCode Patterns

### Two Pointers (LeetCode 167: Two Sum II)

```python
def twoSum(numbers, target):
    left, right = 0, len(numbers) - 1
    while left < right:
        total = numbers[left] + numbers[right]
        if total == target:
            return [left + 1, right + 1]
        elif total < target:
            left += 1
        else:
            right -= 1
    return []

# What's happening: numbers is a list (dynamic array of pointers).
# numbers[left] is O(1) pointer lookup — no linked list traversal.
# Two pointers converge from both ends in O(n) time.
```

### Sliding Window (LeetCode 209: Minimum Size Subarray Sum)

```python
def minSubArrayLen(target, nums):
    left = 0
    current_sum = 0
    min_len = float('inf')

    for right in range(len(nums)):
        current_sum += nums[right]
        while current_sum >= target:
            min_len = min(min_len, right - left + 1)
            current_sum -= nums[left]
            left += 1

    return min_len if min_len != float('inf') else 0

# Pattern: expand right boundary, shrink left boundary when condition met.
# List random access O(1) makes this efficient.
```

### Stack Using List (LeetCode 20: Valid Parentheses)

```python
def isValid(s):
    stack = []  # list as stack — append() = push, pop() = pop
    matching = {')': '(', '}': '{', ']': '['}

    for char in s:
        if char in matching:
            if not stack or stack[-1] != matching[char]:
                return False
            stack.pop()   # O(1) — removes last element
        else:
            stack.append(char)  # O(1) amortized

    return len(stack) == 0

# Why list works as a stack:
# append() and pop() are both O(1) amortized.
# DON'T use pop(0) for queue — that's O(n). Use collections.deque instead.
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| List internal | Dynamic array of PyObject pointers (NOT inline values). 8 bytes per slot |
| Growth | ~12.5% over-allocation. Less aggressive than Go's doubling |
| Tuple internal | Immutable, fixed-size, inline pointer array. Cached for sizes 0-19 |
| Memory | Tuple ~40% smaller than equivalent list (no over-allocation, inline array) |
| Slicing | Always creates a NEW pointer array (shallow copy). Unlike Go's shared backing |
| Comprehensions | Faster than for+append — compiles to C-level LIST_APPEND opcode |
| As stack | `append()` + `pop()` are O(1). Never use `pop(0)` — it's O(n) |
| As queue | Use `collections.deque` — O(1) on both ends |
