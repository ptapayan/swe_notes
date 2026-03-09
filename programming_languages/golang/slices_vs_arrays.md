# Slices vs Arrays in Go

## Arrays

### What Is an Array?

An array is a **fixed-size**, **contiguous block** of memory holding elements of the same type. The size is part of the type.

```go
var a [5]int              // [0, 0, 0, 0, 0] — zero-valued
b := [3]string{"go", "is", "fun"}
c := [...]int{1, 2, 3, 4} // compiler counts: [4]int
```

**Critical:** `[3]int` and `[5]int` are **different types**. You cannot assign one to the other or pass `[3]int` where `[5]int` is expected.

### Memory Layout

```go
a := [4]int{10, 20, 30, 40}
```

```
Stack (or heap, depending on escape analysis):
┌────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │  32 bytes total (4 × 8 bytes for int64)
└────┴────┴────┴────┘
 a[0]  a[1] a[2] a[3]
```

- No header, no pointer, no metadata — just the raw data
- Total size is always `len × sizeof(element)`, known at compile time

### Arrays Are Values (Copied on Assignment)

```go
a := [3]int{1, 2, 3}
b := a     // COPIES all 3 elements (24 bytes)
b[0] = 99
fmt.Println(a) // [1 2 3] — unchanged
fmt.Println(b) // [99 2 3]
```

Passing an array to a function copies the entire thing:

```go
func modify(arr [1000000]int) { // copies 8MB!
    arr[0] = 42
}
```

**This is why arrays are rarely used directly.** Use slices instead.

---

## Slices

### What Is a Slice?

A slice is a **dynamic view** into an underlying array. It's a lightweight descriptor (header) that references a segment of an array.

```go
s := []int{1, 2, 3, 4, 5}
```

### Slice Header (3 Words = 24 Bytes)

```go
// runtime representation (simplified)
type slice struct {
    array unsafe.Pointer // pointer to underlying array
    len   int            // number of accessible elements
    cap   int            // total capacity of underlying array from this point
}
```

```
Stack:                          Heap:
┌──────────────────┐           ┌───┬───┬───┬───┬───┐
│ ptr ─────────────┼──────────→│ 1 │ 2 │ 3 │ 4 │ 5 │
│ len: 5           │           └───┴───┴───┴───┴───┘
│ cap: 5           │            backing array
└──────────────────┘
  slice header
```

### Creating Slices

```go
// Literal
s1 := []int{1, 2, 3}

// From make
s2 := make([]int, 5)      // len=5, cap=5, all zeros
s3 := make([]int, 0, 10)  // len=0, cap=10, empty but pre-allocated

// From array
arr := [5]int{10, 20, 30, 40, 50}
s4 := arr[1:4]  // [20, 30, 40] — shares memory with arr!

// Nil slice
var s5 []int // nil, len=0, cap=0
```

**Related:** [make function](make_function_in_golang.md) for details on `make` for slices.

---

## Slicing Operations

```go
s := []int{0, 1, 2, 3, 4, 5, 6, 7}

s[2:5]   // [2, 3, 4]     — from index 2 to 4 (exclusive)
s[:3]    // [0, 1, 2]     — from start to 2
s[5:]    // [5, 6, 7]     — from 5 to end
s[:]     // [0, 1, 2, ..] — whole slice (same backing array)
```

### Slices Share Memory (Critical to Understand)

```go
original := []int{1, 2, 3, 4, 5}
sub := original[1:3] // [2, 3]

sub[0] = 99
fmt.Println(original) // [1, 99, 3, 4, 5] — MODIFIED!
```

**What's happening:**

```
original header: ptr→[1, 2, 3, 4, 5]  len=5  cap=5
                       ↑
sub header:      ptr───┘  [2, 3]       len=2  cap=4
```

Both slice headers point into the SAME backing array. Modifying through either one affects the other.

### Full Slice Expression (Limiting Capacity)

```go
s := []int{0, 1, 2, 3, 4}
sub := s[1:3:3] // [1, 2] with cap=2 (third number limits capacity)
```

This prevents `sub` from accidentally appending into `s`'s territory. The three-index form `[low:high:max]` sets `cap = max - low`.

---

## Append

```go
s := []int{1, 2, 3}
s = append(s, 4)          // add one element
s = append(s, 5, 6, 7)    // add multiple
s = append(s, other...)   // append another slice
```

### How Append Works Internally

```
If len < cap:
  → Write to s[len], increment len. No allocation.

If len == cap:
  → Allocate new, larger backing array
  → Copy existing elements to new array
  → Add new element
  → Return new slice header pointing to new array
  → Old array becomes garbage
```

**Growth strategy (Go 1.18+):**
- For cap < 256: double the capacity
- For cap >= 256: grow by ~25% + 192 (formula: `newcap = oldcap + (oldcap + 3*256) / 4`)

```go
s := make([]int, 0)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// len=1  cap=1
// len=2  cap=2
// len=3  cap=4     ← doubled
// len=4  cap=4
// len=5  cap=8     ← doubled
// len=6  cap=8
// ...
```

**ALWAYS reassign:** `s = append(s, val)`. The old `s` may point to a now-stale backing array if a reallocation happened.

---

## Copy

```go
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)

n := copy(dst, src) // copies min(len(dst), len(src)) elements
fmt.Println(dst)    // [1, 2, 3]
fmt.Println(n)      // 3

// copy creates independent memory — no shared backing array
dst[0] = 99
fmt.Println(src[0]) // 1 — unaffected
```

Use `copy` when you need to **detach** a slice from its backing array.

---

## Nil Slice vs Empty Slice

```go
var nilSlice []int          // nil, len=0, cap=0
emptySlice := []int{}       // NOT nil, len=0, cap=0
madeSlice := make([]int, 0) // NOT nil, len=0, cap=0
```

**In practice they behave the same:**

```go
len(nilSlice)                // 0
cap(nilSlice)                // 0
nilSlice = append(nilSlice, 1) // works fine
for range nilSlice { }       // zero iterations

// Only difference: nil check
nilSlice == nil  // true
emptySlice == nil // false
```

**Convention:** Return `nil` to indicate "no data" and `[]T{}` to indicate "empty collection." JSON marshaling differs: `nil` → `null`, `[]T{}` → `[]`.

---

## Common Patterns

### Removing an Element (Order Preserved)

```go
s := []int{1, 2, 3, 4, 5}
i := 2 // remove index 2

s = append(s[:i], s[i+1:]...) // [1, 2, 4, 5]
```

**Memory:** This shifts elements left. O(n) operation.

### Removing an Element (Order NOT Preserved, O(1))

```go
s := []int{1, 2, 3, 4, 5}
i := 2

s[i] = s[len(s)-1]   // replace with last element
s = s[:len(s)-1]      // shrink by one
// [1, 2, 5, 4]
```

### Filtering Without Allocation

```go
// Reuses the same backing array
func filter(s []int, keep func(int) bool) []int {
    result := s[:0] // len=0 but shares backing array
    for _, v := range s {
        if keep(v) {
            result = append(result, v)
        }
    }
    return result
}

nums := []int{1, 2, 3, 4, 5, 6}
evens := filter(nums, func(n int) bool { return n%2 == 0 })
// evens = [2, 4, 6], no new allocation
```

---

## Passing Slices to Functions

```go
func modify(s []int) {
    s[0] = 99 // modifies the original backing array
}

func appendToSlice(s []int) []int {
    return append(s, 100) // MAY create a new backing array
}

func main() {
    s := []int{1, 2, 3}
    modify(s)
    fmt.Println(s) // [99, 2, 3] — modified!

    s2 := appendToSlice(s)
    fmt.Println(s)  // [99, 2, 3] — unchanged if append caused reallocation
    fmt.Println(s2) // [99, 2, 3, 100]
}
```

**How to think about it:** The slice header (24 bytes: pointer + len + cap) is copied. But the pointer still points to the same backing array. So direct index writes are visible to the caller, but `append` may or may not be (depends on whether it reallocated).

**Related:** [functions (pass-by-value)](function_in_golang.md) | [structs (value semantics)](structs_in_golang.md)

---

---

## Practical Examples / LeetCode Patterns

### Two Sum (LeetCode 1) — Slices + Maps

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums)) // pre-allocate map
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```

**What's happening:** `nums` is a slice (passed by header copy, backing array shared). The `seen` map pre-allocates for `len(nums)` entries to avoid rehashing.

### Sliding Window (LeetCode 3: Longest Substring Without Repeating)

```go
func lengthOfLongestSubstring(s string) int {
    lastSeen := make(map[byte]int)
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        if idx, ok := lastSeen[s[right]]; ok && idx >= left {
            left = idx + 1
        }
        lastSeen[s[right]] = right
        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}
```

**What's happening:** We iterate by byte index (fine for ASCII). The slice `s` is a string (immutable byte sequence). `s[right]` is a byte access — O(1). The two pointers `left` and `right` define a window into the string's backing bytes.

### In-Place Array Manipulation (LeetCode 26: Remove Duplicates)

```go
func removeDuplicates(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    slow := 0
    for fast := 1; fast < len(nums); fast++ {
        if nums[fast] != nums[slow] {
            slow++
            nums[slow] = nums[fast] // modifying backing array in-place
        }
    }
    return slow + 1
}
```

**Memory insight:** This modifies the slice's backing array in-place. No new allocation. The caller sees the changes because the slice header points to the same backing array. We return the new "logical length" since we can't shrink the allocation.

### Rotate Array (LeetCode 189) — Slice Tricks

```go
func rotate(nums []int, k int) {
    k %= len(nums)
    reverse(nums)           // reverse entire array
    reverse(nums[:k])       // reverse first k elements
    reverse(nums[k:])       // reverse remaining elements
}

func reverse(s []int) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i] // swap via multi-assignment
    }
}
```

**What's happening:** `nums[:k]` and `nums[k:]` are sub-slices sharing the same backing array. `reverse` modifies in-place through the shared memory. No allocations at all — pure O(1) space.

---

---

## Practical Examples (LeetCode Patterns)

### Two Sum (LeetCode #1) — Slices + Maps Together

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int, len(nums)) // val → index
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```

**What's happening:** We iterate the slice once (O(n)). The map gives O(1) lookups. Pre-allocating with `len(nums)` avoids rehashing.

### Sliding Window Maximum (LeetCode #239) — Slice as Deque

```go
func maxSlidingWindow(nums []int, k int) []int {
    result := make([]int, 0, len(nums)-k+1)
    deque := []int{} // stores indices, acts as a monotonic deque

    for i, n := range nums {
        // Remove elements outside the window
        for len(deque) > 0 && deque[0] < i-k+1 {
            deque = deque[1:] // pop front
        }
        // Remove smaller elements from back (they'll never be the max)
        for len(deque) > 0 && nums[deque[len(deque)-1]] < n {
            deque = deque[:len(deque)-1] // pop back
        }
        deque = append(deque, i)
        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}
```

**Key slice operations at play:** `deque[1:]` (pop front via re-slicing), `deque[:len-1]` (pop back), `append` (push back). All are cheap because they just adjust the slice header.

### Rotate Array (LeetCode #189) — In-Place Slice Reversal

```go
func rotate(nums []int, k int) {
    k %= len(nums)
    reverse(nums)
    reverse(nums[:k])
    reverse(nums[k:])
}

func reverse(s []int) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}
```

**Why this works:** Sub-slices share the same backing array, so reversing a sub-slice modifies the original in place. Zero allocations.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Array | Fixed size, value type, size is part of the type. Rarely used directly |
| Slice header | 24 bytes: pointer + len + cap. Passed by value (header copied) |
| Shared backing | Sub-slices share the same array. Mutation is visible across slices |
| Append | Always reassign `s = append(s, v)`. May reallocate |
| Copy | Creates independent memory. Use to detach from shared backing |
| Nil vs empty | Both work the same. Nil → JSON `null`, empty → JSON `[]` |
| Pre-allocate | `make([]T, 0, n)` avoids repeated reallocation |
