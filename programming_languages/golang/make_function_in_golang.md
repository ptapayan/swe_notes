# The `make` Function in Go

**Related:** [slices vs arrays](slices_vs_arrays.md) | [maps](maps_in_golang.md) | [goroutines and channels](go_routines_and_channels.md)

## What Is `make`?

`make` is a **built-in function** that allocates and initializes slices, maps, and channels. These are the only three types it works with — they all require internal runtime initialization beyond just zeroing memory.

```go
slice := make([]int, 5)       // slice
m := make(map[string]int)     // map
ch := make(chan int, 10)       // channel
```

---

## `make` vs `new` — What's the Difference?

```go
// new: allocates zeroed memory, returns a POINTER
p := new(int)       // *int pointing to 0
s := new([]int)     // *[]int pointing to nil slice header

// make: allocates AND initializes, returns the VALUE (not a pointer)
sl := make([]int, 5) // []int with len=5, cap=5, backing array allocated
```

**Why the distinction exists:**
- `new` just zeros memory. A zeroed slice/map/channel is `nil` and unusable
- `make` does the internal setup: allocates the backing array (slice), creates the hash table (map), or sets up the buffer and queues (channel)

```go
// This panics:
var m map[string]int
m["key"] = 1 // panic: assignment to entry in nil map

// This works:
m := make(map[string]int)
m["key"] = 1 // fine
```

---

## `make` for Slices

### Syntax

```go
make([]T, length)           // len = length, cap = length
make([]T, length, capacity) // len = length, cap = capacity
```

### What Happens in Memory

```go
s := make([]int, 3, 5)
```

This creates:

```
Stack:                    Heap:
┌──────────────┐         ┌───┬───┬───┬───┬───┐
│ ptr ──────────┼────────→│ 0 │ 0 │ 0 │ ? │ ? │  backing array (5 ints)
│ len: 3       │         └───┴───┴───┴───┴───┘
│ cap: 5       │
└──────────────┘
 (slice header)
```

- The **slice header** (24 bytes: pointer + len + cap) is on the stack
- The **backing array** (5 * 8 = 40 bytes for int64) is on the heap
- Elements 0-2 are initialized to zero value (`0` for int)
- Elements 3-4 exist in memory but are not accessible via the slice until you `append`

### Common Usage Patterns

```go
// When you know the exact size
scores := make([]int, 10)
for i := range scores {
    scores[i] = i * 10
}

// When you'll append but know the approximate size
// len=0 so it's empty, but cap=100 avoids reallocations
results := make([]string, 0, 100)
for _, item := range items {
    results = append(results, process(item))
}
```

### Why Pre-allocate Capacity?

```go
// Without pre-allocation: multiple reallocations
s := []int{}
for i := 0; i < 1000; i++ {
    s = append(s, i)
    // Go doubles capacity when full: 0→1→2→4→8→16→...→1024
    // That's ~10 allocations + copies
}

// With pre-allocation: zero reallocations
s := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    s = append(s, i) // never reallocates
}
```

Each reallocation means:
1. Allocate a new, larger backing array on the heap
2. Copy all existing elements to the new array
3. Old array becomes garbage (GC has to collect it)

Pre-allocating with `make` avoids all of this.

---

## `make` for Maps

### Syntax

```go
make(map[KeyType]ValueType)           // default initial capacity
make(map[KeyType]ValueType, sizeHint) // hint for expected number of entries
```

### What Happens in Memory

```go
m := make(map[string]int, 10)
```

A map in Go is a **hash table** implemented as an `hmap` struct:

```
┌─────────────────┐
│ hmap            │
│  count: 0       │  number of entries
│  B: 1           │  log2 of number of buckets
│  buckets ───────┼──→ [array of bmap buckets]
│  oldbuckets     │     each bucket holds 8 key-value pairs
│  ...            │
└─────────────────┘
```

Each **bucket** holds up to 8 key-value pairs. The runtime uses the hash of the key to pick a bucket, then does a linear scan within that bucket.

### Size Hint

```go
// No hint: runtime picks a small default
m := make(map[string]int)

// With hint: pre-allocates enough buckets for ~10 entries
m := make(map[string]int, 10)
```

The size hint avoids **rehashing**. When a map gets too full (load factor > 6.5 entries per bucket), Go allocates a new, larger bucket array and gradually migrates entries — this is expensive.

```go
// Good practice when you know the size
userAges := make(map[string]int, len(users))
for _, u := range users {
    userAges[u.Name] = u.Age
}
```

---

## `make` for Channels

### Syntax

```go
make(chan Type)          // unbuffered channel (capacity 0)
make(chan Type, capacity) // buffered channel
```

### What Happens in Memory

```go
ch := make(chan int, 5)
```

Creates an `hchan` struct on the heap:

```
┌─────────────────┐
│ hchan           │
│  qcount: 0      │  current number of items in buffer
│  dataqsiz: 5    │  capacity
│  buf ───────────┼──→ [circular buffer: 5 int slots]
│  sendx: 0       │  send index into circular buffer
│  recvx: 0       │  receive index into circular buffer
│  sendq          │  queue of waiting senders (linked list)
│  recvq          │  queue of waiting receivers (linked list)
│  lock           │  mutex
└─────────────────┘
```

**Unbuffered (`make(chan int)`):** No circular buffer. Sends and receives directly hand off data between goroutines.

**Buffered (`make(chan int, 5)`):** A circular ring buffer. Sends write to `buf[sendx]` and advance sendx. Receives read from `buf[recvx]` and advance recvx.

---

## Nil vs Empty (Zero Value vs Made)

| Type | Zero Value | After `make` |
|---|---|---|
| `[]int` | `nil` (no backing array) | Empty but initialized (has backing array) |
| `map[K]V` | `nil` (reads return zero, writes panic) | Empty but usable |
| `chan T` | `nil` (send/receive block forever) | Usable channel |

```go
// Nil slice — safe to append, safe to range (0 iterations)
var s []int
s = append(s, 1) // works! creates backing array
len(s)           // 1

// Nil map — safe to read, PANICS on write
var m map[string]int
_ = m["key"]     // returns 0 (zero value), no panic
m["key"] = 1     // PANIC: assignment to entry in nil map

// Nil channel — blocks forever
var ch chan int
ch <- 1   // blocks forever (deadlock if no other goroutine)
<-ch      // blocks forever
```

---

## When to Use `make` vs Literal Initialization

```go
// These are equivalent for slices:
s1 := make([]int, 0)
s2 := []int{}

// But make is needed when you want to set capacity:
s3 := make([]int, 0, 100) // no literal equivalent

// These are equivalent for maps:
m1 := make(map[string]int)
m2 := map[string]int{}

// Use literal when you have initial values:
m3 := map[string]int{"a": 1, "b": 2}

// Use make when you need a size hint:
m4 := make(map[string]int, 1000) // no literal equivalent
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| `make` | Only for slices, maps, channels. Returns initialized value, not pointer |
| `new` | Returns pointer to zeroed memory. Rarely used in practice |
| Slice capacity | Pre-allocate with `make([]T, 0, n)` to avoid reallocations |
| Map size hint | `make(map[K]V, n)` avoids rehashing |
| Nil vs empty | Nil map panics on write. Nil channel blocks forever. Nil slice is safe to append |
