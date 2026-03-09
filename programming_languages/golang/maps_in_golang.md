# Maps in Go

## What Is a Map?

A map is a built-in hash table that stores **key-value pairs** with O(1) average lookup time.

```go
m := map[string]int{
    "alice": 95,
    "bob":   87,
}
```

**Related:** [make function in Go](make_function_in_golang.md) | [for loops (range over maps)](for_loops_in_golang.md) | [structs (as map values)](structs_in_golang.md)

---

## How Maps Work Under the Hood

A map variable is a **pointer** to an `hmap` struct on the heap.

```
Stack:              Heap:
┌─────────┐        ┌──────────────────┐
│ *hmap ──┼───────→│ hmap             │
└─────────┘        │  count: 2        │  number of entries
                   │  B: 0            │  log2(buckets) — here 2^0 = 1 bucket
                   │  buckets ────────┼──→ [bmap array]
                   │  hash0           │  random hash seed (for security)
                   └──────────────────┘
```

Each **bucket** (`bmap`) holds up to **8 key-value pairs**:
```
bmap:
┌─────────────────────────────────────┐
│ tophash[8]  │  8 bytes, one per slot — top byte of each key's hash
│ keys[8]     │  8 keys packed together
│ values[8]   │  8 values packed together
│ overflow    │  pointer to next bucket (for chaining)
└─────────────────────────────────────┘
```

**Why keys and values are stored separately** (not interleaved): To avoid padding waste. If key is `int8` and value is `int64`, interleaving would add 7 bytes of padding per entry. Grouping them avoids this.

### Hash Table Operations

**Lookup:**
1. Hash the key using a randomly seeded hash function
2. Use low bits of hash to select a bucket
3. Use top 8 bits (`tophash`) for fast comparison within the bucket
4. If tophash matches, compare full key
5. If bucket is full, follow overflow pointer

**Growth:** When the load factor exceeds 6.5 (entries per bucket), the map doubles its bucket count. Growth is **incremental** — entries are migrated a few at a time on each insert/delete, not all at once.

---

## Creating Maps

```go
// Literal
m := map[string]int{
    "a": 1,
    "b": 2,
}

// make (empty but initialized)
m := make(map[string]int)

// make with size hint (avoids early rehashing)
m := make(map[string]int, 100)

// Zero value is nil — reads work, writes PANIC
var m map[string]int
_ = m["key"]    // returns 0, no panic
m["key"] = 1    // PANIC: assignment to entry in nil map
```

**Related:** [make function](make_function_in_golang.md) for details on `make` vs `new` and size hints.

---

## CRUD Operations

### Create / Update

```go
m := make(map[string]int)
m["alice"] = 95  // create
m["alice"] = 100 // update (same syntax)
```

### Read

```go
score := m["alice"] // returns 100

// Comma-ok idiom — distinguish "not found" from "zero value"
score, ok := m["bob"]
if !ok {
    fmt.Println("bob not found")
}
```

**Why the comma-ok matters:**
```go
m := map[string]int{"alice": 0}
v := m["alice"] // v = 0
v = m["bob"]    // v = 0 — same! Can't tell if key exists

v, ok := m["alice"] // v=0, ok=true (exists)
v, ok = m["bob"]    // v=0, ok=false (doesn't exist)
```

### Delete

```go
delete(m, "alice") // removes the key
delete(m, "nonexistent") // no-op, no panic
```

**Note:** `delete` does NOT shrink the map's memory. Deleted bucket slots are marked empty but the bucket array doesn't shrink. If you build up a large map and delete most entries, the memory stays allocated until the map is garbage collected.

---

## Iterating Over Maps

```go
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Keys only
for key := range m {
    fmt.Println(key)
}
```

**Critical: Map iteration order is NOT deterministic.** The Go runtime intentionally randomizes it. If you need sorted output:

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
    fmt.Printf("%s: %d\n", k, m[k])
}
```

**Related:** [for loops](for_loops_in_golang.md) for more on range behavior.

---

## Maps Are Reference Types

A map variable is a pointer. Passing a map to a function passes the pointer — both see the same data.

```go
func addEntry(m map[string]int) {
    m["new"] = 42 // modifies the original map
}

func main() {
    m := map[string]int{"a": 1}
    addEntry(m)
    fmt.Println(m["new"]) // 42
}
```

This is different from [structs](structs_in_golang.md) and [arrays](slices_vs_arrays.md), which are copied on assignment.

---

## Valid Key Types

Map keys must be **comparable** (support `==`). Valid: all basic types, pointers, channels, structs (if all fields are comparable), arrays, interfaces.

**NOT valid:** slices, maps, functions.

```go
// OK
map[int]string{}
map[string]int{}
map[[2]int]bool{}  // array keys work
map[struct{X,Y int}]string{} // struct keys work

// NOT OK — compile error
map[[]int]string{}     // slices not comparable
map[map[int]int]string{} // maps not comparable
```

---

## Concurrency Safety

**Maps are NOT safe for concurrent use.** Reading from multiple goroutines is fine, but concurrent read+write or write+write causes a runtime panic (since Go 1.6).

```go
// UNSAFE — will panic with "concurrent map writes"
m := make(map[string]int)
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
```

**Solutions:**

### sync.Mutex

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Set(key string, val int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = val
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    v, ok := sm.m[key]
    return v, ok
}
```

### sync.Map (built-in concurrent map)

```go
var m sync.Map

m.Store("key", 42)
val, ok := m.Load("key")
m.Delete("key")
m.Range(func(key, value any) bool {
    fmt.Println(key, value)
    return true // continue iteration
})
```

`sync.Map` is optimized for two specific patterns:
1. Keys are written once and read many times
2. Multiple goroutines read/write disjoint key sets

For general-purpose concurrent maps, `sync.RWMutex` + regular map is usually better.

**Related:** [goroutines and channels](go_routines_and_channels.md) for more on concurrency.

---

## Common Patterns

### Set (using map[T]struct{})

```go
// struct{} takes 0 bytes of memory
seen := make(map[string]struct{})
seen["alice"] = struct{}{}

if _, ok := seen["alice"]; ok {
    fmt.Println("already seen")
}
```

Why `struct{}` instead of `bool`? A `struct{}` is truly zero-sized. A `bool` takes 1 byte per entry. For millions of entries, this adds up.

### Counting / Grouping

```go
// Word frequency
words := []string{"go", "is", "go", "great", "go"}
freq := make(map[string]int)
for _, w := range words {
    freq[w]++ // zero value of int is 0, so this just works
}
// freq = {"go": 3, "is": 1, "great": 1}

// Grouping
type Person struct {
    Name string
    City string
}

people := []Person{
    {"Alice", "Portland"},
    {"Bob", "Portland"},
    {"Carol", "Seattle"},
}

byCity := make(map[string][]Person)
for _, p := range people {
    byCity[p.City] = append(byCity[p.City], p)
}
```

---

---

## Practical Examples / LeetCode Patterns

### Group Anagrams (LeetCode 49) — Map with Sorted Key

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)

    for _, s := range strs {
        // Sort characters to create a canonical key
        runes := []rune(s)
        sort.Slice(runes, func(i, j int) bool { return runes[i] < runes[j] })
        key := string(runes)
        groups[key] = append(groups[key], s)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```

**What's happening:** The map key is a sorted version of each string. `append` on a nil map value works because `groups[key]` returns a nil slice, and `append(nil, s)` allocates a new slice. This is a common Go idiom — no need to initialize the slice first.

### LRU Cache (LeetCode 146) — Map + Doubly Linked List

```go
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node // most recently used
    tail     *Node // least recently used
}

type Node struct {
    key, val   int
    prev, next *Node
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node, capacity),
        head:     head,
        tail:     tail,
    }
}

func (c *LRUCache) Get(key int) int {
    if node, ok := c.cache[key]; ok {
        c.moveToFront(node)
        return node.val
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    if node, ok := c.cache[key]; ok {
        node.val = value
        c.moveToFront(node)
        return
    }
    node := &Node{key: key, val: value}
    c.cache[key] = node
    c.addToFront(node)
    if len(c.cache) > c.capacity {
        lru := c.tail.prev
        c.remove(lru)
        delete(c.cache, lru.key)
    }
}

func (c *LRUCache) addToFront(node *Node) {
    node.next = c.head.next
    node.prev = c.head
    c.head.next.prev = node
    c.head.next = node
}

func (c *LRUCache) remove(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) moveToFront(node *Node) {
    c.remove(node)
    c.addToFront(node)
}
```

**Map role:** The map provides O(1) lookup by key. The linked list provides O(1) insertion/removal order tracking. Together they give O(1) for all operations.

### Frequency Counter Pattern (LeetCode 242: Valid Anagram)

```go
func isAnagram(s string, t string) bool {
    if len(s) != len(t) {
        return false
    }
    count := make(map[rune]int)
    for _, r := range s {
        count[r]++
    }
    for _, r := range t {
        count[r]--
        if count[r] < 0 {
            return false
        }
    }
    return true
}
```

**Optimization:** For ASCII-only input, use `[26]int` instead of a map — array access is ~10x faster than map lookup:

```go
func isAnagram(s string, t string) bool {
    if len(s) != len(t) {
        return false
    }
    var count [26]int // fixed-size array on the stack — zero allocation
    for i := 0; i < len(s); i++ {
        count[s[i]-'a']++
        count[t[i]-'a']--
    }
    for _, c := range count {
        if c != 0 {
            return false
        }
    }
    return true
}
```

---

---

## Practical Examples (LeetCode Patterns)

### Group Anagrams (LeetCode #49) — Map with Sorted Key

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)
    for _, s := range strs {
        // Sort characters to create a canonical key
        runes := []rune(s)
        sort.Slice(runes, func(i, j int) bool { return runes[i] < runes[j] })
        key := string(runes)
        groups[key] = append(groups[key], s)
    }

    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```

**Map pattern:** Using a derived key (sorted string) to group related items. The `append` on `groups[key]` works even if the key doesn't exist yet (zero value of `[]string` is nil, and `append(nil, x)` works).

### LRU Cache (LeetCode #146) — Map + Doubly Linked List

```go
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node // most recent
    tail     *Node // least recent
}

type Node struct {
    key, val   int
    prev, next *Node
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node, capacity),
        head:     head,
        tail:     tail,
    }
}
```

**Why map + linked list:** Map gives O(1) lookup. Linked list gives O(1) insertion/removal for LRU ordering. The map stores pointers to nodes so we can jump from key → node → position in list instantly.

### Valid Anagram (LeetCode #242) — Map as Frequency Counter

```go
func isAnagram(s string, t string) bool {
    if len(s) != len(t) {
        return false
    }
    freq := make(map[rune]int)
    for _, r := range s {
        freq[r]++
    }
    for _, r := range t {
        freq[r]--
        if freq[r] < 0 {
            return false
        }
    }
    return true
}
```

**Pattern:** Increment counts for one string, decrement for the other. Map's zero value (0) means you don't need to check if a key exists before incrementing.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Internal | Pointer to hash table (`hmap`). Buckets hold 8 entries each |
| Nil map | Reads return zero value. Writes PANIC |
| Iteration order | Randomized by runtime. Never depend on it |
| Reference type | Passing a map shares the underlying data |
| Concurrency | NOT safe. Use `sync.RWMutex` or `sync.Map` |
| Key types | Must be comparable. No slices, maps, or functions |
| Set pattern | Use `map[T]struct{}` for zero-cost sets |
