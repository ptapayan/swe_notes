# Collections in Rust

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics (Iterator)](traits_and_generics.md) | [smart pointers](smart_pointers.md) | [error handling (collect with Result)](error_handling_in_rust.md)

## Vec<T> -- Dynamic Array

### What Is a Vec?

A `Vec<T>` is a heap-allocated, growable array. It is Rust's most commonly used collection -- the equivalent of Go's slice or C++'s `std::vector`.

```rust
let mut v: Vec<i32> = Vec::new();
v.push(1);
v.push(2);
v.push(3);

// Macro shorthand
let v = vec![1, 2, 3];
```

### Memory Layout (ptr + len + cap)

```rust
// runtime representation (simplified)
struct Vec<T> {
    ptr: *mut T,    // pointer to heap-allocated buffer
    len: usize,     // number of elements currently stored
    cap: usize,     // total capacity of allocated buffer
}
```

```
Stack:                           Heap:
v: ┌──────────────────┐        ┌───┬───┬───┬───┬───┐
   │ ptr ─────────────┼───────>│ 1 │ 2 │ 3 │ _ │ _ │
   │ len: 3           │        └───┴───┴───┴───┴───┘
   │ cap: 5           │         used ^^^  unused ^^^
   └──────────────────┘
    24 bytes on stack
```

**How to think about it:** Identical to Go's slice header (ptr + len + cap = 24 bytes). The difference is that `Vec` OWNS the heap allocation. When `Vec` is dropped, the heap buffer is freed. In Go, the GC handles this.

### Growth Strategy

When `len == cap` and you push, Vec allocates a new buffer and copies:

```
len < cap:  push is O(1), just write to buffer[len] and increment len
len == cap: allocate new buffer (2x capacity), copy, free old buffer
```

**Growth factor:** Vec doubles capacity when growing (`cap = max(1, cap * 2)`). Same strategy as Go slices for small capacities.

```rust
let mut v = Vec::new();
for i in 0..10 {
    v.push(i);
    println!("len={}, cap={}", v.len(), v.capacity());
}
// len=1, cap=1
// len=2, cap=2
// len=3, cap=4    <- doubled
// len=4, cap=4
// len=5, cap=8    <- doubled
// ...
```

### Pre-Allocation

```rust
// Avoid repeated reallocations -- allocate exact capacity upfront
let mut v = Vec::with_capacity(1000);
for i in 0..1000 {
    v.push(i);  // no reallocations -- capacity was pre-allocated
}
```

### Slicing a Vec

```rust
let v = vec![10, 20, 30, 40, 50];
let slice: &[i32] = &v[1..4];   // [20, 30, 40] -- borrows, no copy
println!("{:?}", slice);

// Mutable slice
let mut v = vec![1, 2, 3, 4, 5];
let slice: &mut [i32] = &mut v[1..4];
slice[0] = 99;
println!("{:?}", v);  // [1, 99, 3, 4, 5]
```

**Key difference from Go:** In Rust, the borrow checker prevents you from holding a reference to a Vec element while mutating the Vec. This prevents the dangling pointer bugs that Go slices can have with shared backing arrays.

---

## String vs &str

### String (Owned, Heap-Allocated)

```rust
// String is essentially Vec<u8> with a UTF-8 guarantee
struct String {
    vec: Vec<u8>,   // ptr + len + cap, guaranteed valid UTF-8
}
```

```rust
let mut s = String::from("hello");
s.push_str(", world");    // grows the internal Vec<u8>
s.push('!');              // push a single char
println!("{}", s);        // "hello, world!"
```

### &str (Borrowed String Slice)

```rust
let s: &str = "hello";   // points to read-only memory (string literal)
// s is a fat pointer: data pointer (8 bytes) + length (8 bytes) = 16 bytes
```

```
Stack:                    Read-only memory:
s: ┌────────────────┐   ┌───┬───┬───┬───┬───┐
   │ ptr ───────────┼──>│ h │ e │ l │ l │ o │
   │ len: 5         │   └───┴───┴───┴───┴───┘
   └────────────────┘
```

### String vs &str Decision

```
String: when you need to OWN the string data (store in struct, return from function, mutate)
&str:   when you just need to READ string data (function parameters, temporary views)
```

```rust
// Accept &str in function parameters -- works with both String and &str
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

let owned = String::from("Alice");
greet(&owned);     // String auto-derefs to &str
greet("Bob");      // &str passed directly
```

### String Concatenation

```rust
// + operator (takes ownership of left side)
let s1 = String::from("hello");
let s2 = String::from(" world");
let s3 = s1 + &s2;  // s1 is MOVED, s2 is borrowed
// println!("{}", s1);  // ERROR: s1 was moved

// format! macro (no ownership transfer, cleaner)
let s = format!("{} {}", "hello", "world");

// String::push_str for building in a loop
let mut result = String::with_capacity(1000);
for word in words {
    result.push_str(word);
    result.push(' ');
}
```

---

## HashMap<K, V>

### Basics

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, i32> = HashMap::new();
scores.insert(String::from("Alice"), 95);
scores.insert(String::from("Bob"), 87);

// Lookup
let alice_score = scores.get("Alice");  // returns Option<&i32>
```

### Memory Layout and Hashing

Rust's `HashMap` uses **SipHash** by default (cryptographic-quality, DoS-resistant, slightly slower than FNV). Internally it uses **Swiss Table** (from Google's Abseil library) since Rust 1.36.

```
Swiss Table layout:
┌─────────────────────────────────┐
│ control bytes (1 per slot)      │  SIMD-friendly metadata
│ [H2|H2|H2|EMPTY|DEL|H2|...]    │  H2 = top 7 bits of hash
├─────────────────────────────────┤
│ key-value slots                 │  actual (K,V) pairs
│ [(k1,v1), (k2,v2), ...]        │
└─────────────────────────────────┘
```

**Lookup:** Hash the key, use low bits to find group of 16 control bytes, SIMD-compare H2 values (checks 16 slots in one CPU instruction), then compare full keys on matches.

### Entry API (Avoid Double Lookup)

```rust
use std::collections::HashMap;

let mut word_count: HashMap<String, i32> = HashMap::new();

// BAD: looks up key twice (once to check, once to insert)
for word in text.split_whitespace() {
    if word_count.contains_key(word) {
        *word_count.get_mut(word).unwrap() += 1;
    } else {
        word_count.insert(word.to_string(), 1);
    }
}

// GOOD: Entry API -- single lookup
for word in text.split_whitespace() {
    *word_count.entry(word.to_string()).or_insert(0) += 1;
}
```

### Ownership Rules for HashMap

```rust
let mut map = HashMap::new();

let key = String::from("hello");
let value = String::from("world");

map.insert(key, value);
// key and value are MOVED into the map
// println!("{}", key);  // COMPILE ERROR: value moved

// For Copy types (i32, bool, etc.), values are copied, not moved
let mut map: HashMap<i32, i32> = HashMap::new();
let k = 1;
map.insert(k, 42);
println!("{}", k);  // works fine -- i32 is Copy
```

---

## HashSet<T>

A `HashSet<T>` is just a `HashMap<T, ()>` -- keys with no values.

```rust
use std::collections::HashSet;

let mut seen: HashSet<i32> = HashSet::new();
seen.insert(1);
seen.insert(2);
seen.insert(1);  // duplicate, no-op
assert_eq!(seen.len(), 2);

// Set operations
let a: HashSet<_> = [1, 2, 3].into();
let b: HashSet<_> = [2, 3, 4].into();

let union: HashSet<_> = a.union(&b).collect();           // {1,2,3,4}
let intersection: HashSet<_> = a.intersection(&b).collect(); // {2,3}
let difference: HashSet<_> = a.difference(&b).collect();     // {1}
```

---

## VecDeque<T>

Double-ended queue backed by a ring buffer. O(1) push/pop from both ends.

```rust
use std::collections::VecDeque;

let mut deque = VecDeque::new();
deque.push_back(1);     // [1]
deque.push_back(2);     // [1, 2]
deque.push_front(0);    // [0, 1, 2]
deque.pop_front();       // Some(0), deque = [1, 2]
deque.pop_back();        // Some(2), deque = [1]
```

**Memory:** Ring buffer on the heap. Head and tail indices wrap around. Unlike `Vec`, push_front is O(1) (Vec would be O(n) because it shifts all elements).

---

## BTreeMap<K, V> and BTreeSet<T>

Ordered collections using a B-tree. Keys must implement `Ord`.

```rust
use std::collections::BTreeMap;

let mut map = BTreeMap::new();
map.insert(3, "c");
map.insert(1, "a");
map.insert(2, "b");

// Iteration is in sorted key order
for (k, v) in &map {
    println!("{}: {}", k, v);  // 1: a, 2: b, 3: c
}

// Range queries
for (k, v) in map.range(1..=2) {
    println!("{}: {}", k, v);  // 1: a, 2: b
}
```

**When to use BTreeMap vs HashMap:**
- **HashMap**: O(1) average lookup, unordered. Default choice
- **BTreeMap**: O(log n) lookup, sorted by key. Use when you need ordered iteration or range queries

---

## Iterators

### Lazy Evaluation

Iterators in Rust are lazy -- they do nothing until consumed. Adapter methods like `map`, `filter`, `take` build up a chain that is only executed when a consumer (`collect`, `sum`, `for_each`, etc.) is called.

```rust
let v = vec![1, 2, 3, 4, 5];

// This does NOTHING until consumed
let iter = v.iter()
    .map(|x| x * 2)
    .filter(|x| *x > 4);

// NOW it executes -- one pass through the data
let result: Vec<i32> = iter.collect();  // [6, 8, 10]
```

### Three Ways to Iterate

```rust
let v = vec![String::from("a"), String::from("b")];

// 1. iter() -- borrows (&T)
for s in v.iter() {         // s: &String
    println!("{}", s);
}
// v is still usable

// 2. iter_mut() -- mutable borrow (&mut T)
let mut v = vec![1, 2, 3];
for n in v.iter_mut() {     // n: &mut i32
    *n *= 2;
}
// v is now [2, 4, 6]

// 3. into_iter() -- takes ownership (T)
for s in v.into_iter() {    // s: String (moved out of v)
    println!("{}", s);
}
// v is no longer usable (moved)
```

### Zero-Cost Iteration

Iterator chains compile down to the same machine code as a hand-written loop:

```rust
// This iterator chain:
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();

// Compiles to essentially the same code as:
let mut sum: i32 = 0;
for x in 0..1000 {
    if x % 2 == 0 {
        sum += x * x;
    }
}
```

LLVM fuses the iterator adaptors into a single loop. No intermediate collections are created. No heap allocations. No function call overhead (the closures are inlined).

### collect() -- The Universal Consumer

```rust
// collect into Vec
let v: Vec<i32> = (0..10).collect();

// collect into HashMap
let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2)].into_iter().collect();

// collect into String
let s: String = vec!['h', 'e', 'l', 'l', 'o'].into_iter().collect();

// collect Result<Vec<T>, E> from iterator of Results
let results: Result<Vec<i32>, _> = vec!["1", "2", "3"]
    .iter()
    .map(|s| s.parse::<i32>())
    .collect();  // Ok([1, 2, 3])
```

`collect()` uses the `FromIterator` trait to know what type to build. The turbofish `::<Type>` or type annotation tells it which implementation to use.

---

## Practical Examples / LeetCode Patterns

### Two Sum (LeetCode 1) -- HashMap Lookup

```rust
use std::collections::HashMap;

fn two_sum(nums: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::with_capacity(nums.len());
    for (i, &num) in nums.iter().enumerate() {
        if let Some(&j) = seen.get(&(target - num)) {
            return Some((j, i));
        }
        seen.insert(num, i);
    }
    None
}
```

**What's happening:** `HashMap::with_capacity` pre-allocates to avoid rehashing. `nums.iter().enumerate()` borrows the slice and yields `(index, &value)` pairs. `i32` is `Copy`, so `seen.insert(num, i)` copies both values in.

### Group Anagrams (LeetCode 49) -- HashMap with Sorted Key

```rust
use std::collections::HashMap;

fn group_anagrams(strs: Vec<String>) -> Vec<Vec<String>> {
    let mut groups: HashMap<Vec<u8>, Vec<String>> = HashMap::new();

    for s in strs {
        let mut key: Vec<u8> = s.bytes().collect();
        key.sort_unstable();
        groups.entry(key).or_default().push(s);
    }

    groups.into_values().collect()
}
```

**Collection insight:** `entry().or_default()` avoids double lookup -- if the key doesn't exist, it inserts `Vec::new()` (via Default trait). `into_values()` takes ownership of the values, draining the HashMap.

### Sliding Window Maximum (LeetCode 239) -- VecDeque as Monotonic Deque

```rust
use std::collections::VecDeque;

fn max_sliding_window(nums: &[i32], k: usize) -> Vec<i32> {
    let mut result = Vec::with_capacity(nums.len() - k + 1);
    let mut deque: VecDeque<usize> = VecDeque::new();  // stores indices

    for i in 0..nums.len() {
        // Remove elements outside the window
        while let Some(&front) = deque.front() {
            if front + k <= i { deque.pop_front(); } else { break; }
        }
        // Remove smaller elements from back
        while let Some(&back) = deque.back() {
            if nums[back] <= nums[i] { deque.pop_back(); } else { break; }
        }
        deque.push_back(i);
        if i >= k - 1 {
            result.push(nums[*deque.front().unwrap()]);
        }
    }
    result
}
```

**Collection insight:** `VecDeque` gives O(1) push/pop at both ends. `front()` and `back()` return `Option<&T>` -- no panics on empty deque. Pre-allocating `result` avoids reallocations.

---

## Key Takeaways

| Collection | Layout | When to Use |
|---|---|---|
| `Vec<T>` | ptr + len + cap (24B). Heap array | Default growable sequence |
| `String` | `Vec<u8>` with UTF-8 guarantee | Owned string data |
| `&str` | ptr + len (16B). Borrowed view | Function params, string slices |
| `HashMap<K,V>` | Swiss Table, SipHash | O(1) key-value lookup |
| `HashSet<T>` | `HashMap<T, ()>` | Membership testing, deduplication |
| `VecDeque<T>` | Ring buffer | O(1) push/pop at both ends (BFS, sliding window) |
| `BTreeMap<K,V>` | B-tree | Sorted keys, range queries |
| Iterators | Lazy, zero-cost | Always prefer over manual loops. Compile to same code |
