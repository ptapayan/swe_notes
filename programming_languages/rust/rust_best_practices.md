# Rust Best Practices (FAANG-Level Engineering)

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics](traits_and_generics.md) | [error handling](error_handling_in_rust.md) | [collections](collections_in_rust.md) | [concurrency](concurrency_in_rust.md) | [smart pointers](smart_pointers.md) | [lifetimes](lifetimes_in_rust.md)

---

## 1. Ownership and Borrowing Patterns

### 1.1 Prefer Borrowing Over Cloning

```rust
// BAD: unnecessary clone -- allocates new heap memory
fn process(data: Vec<String>) {
    for s in &data {
        println!("{}", s);
    }
}
let data = vec![String::from("hello")];
process(data.clone());  // clones the entire Vec AND each String
process(data);           // could have just passed ownership or borrowed

// GOOD: borrow when you only need to read
fn process(data: &[String]) {  // accepts &Vec<String> too (Deref coercion)
    for s in data {
        println!("{}", s);
    }
}
let data = vec![String::from("hello")];
process(&data);  // borrows -- zero allocation
process(&data);  // can call again -- data wasn't moved
```

### 1.2 Accept the Most General Borrow Type

```rust
// BAD: overly specific -- only accepts &String
fn greet(name: &String) {
    println!("Hello, {}!", name);
}

// GOOD: accepts &str -- works with String, &str, &String, Cow<str>
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

// BAD: overly specific -- only accepts &Vec<i32>
fn sum(nums: &Vec<i32>) -> i32 { ... }

// GOOD: accepts &[i32] -- works with Vec, array, slice
fn sum(nums: &[i32]) -> i32 {
    nums.iter().sum()
}
```

**Rule:** Accept `&str` instead of `&String`, `&[T]` instead of `&Vec<T>`, `&dyn Trait` or `impl Trait` instead of concrete types. This makes your API maximally flexible.

### 1.3 When to Use Clone

Clone is NOT a code smell. Use it when:

```rust
// 1. Data is small and Copy-like (but doesn't implement Copy)
let name = user.name.clone();  // String clone is fine for a name

// 2. You need ownership and the source will be used again
let config_copy = config.clone();
thread::spawn(move || use_config(config_copy));
// config is still usable here

// 3. Breaking complex borrow checker issues where restructuring is worse
// Sometimes .clone() is the pragmatic choice over fighting the borrow checker
```

**When NOT to clone:**
- Inside hot loops (clone in a loop = O(n) allocations)
- Large data structures (Vec of Strings, complex nested types)
- When borrowing would work (most of the time)

---

## 2. Iterator Chains vs Manual Loops

### 2.1 Prefer Iterator Chains

```rust
// BAD: imperative loop with manual state
let mut result = Vec::new();
for item in &items {
    if item.active {
        let transformed = item.value * 2;
        result.push(transformed);
    }
}

// GOOD: iterator chain -- clearer intent, same performance
let result: Vec<_> = items.iter()
    .filter(|item| item.active)
    .map(|item| item.value * 2)
    .collect();
```

**Why iterators are preferred:**
1. No off-by-one errors (no manual indexing)
2. LLVM optimizes them identically to manual loops (zero-cost)
3. Easier to parallelize (change `.iter()` to `.par_iter()` with Rayon)
4. Chainable -- compose operations without intermediate collections

### 2.2 Common Iterator Patterns

```rust
// Enumerate (index + value)
for (i, item) in items.iter().enumerate() { ... }

// Zip (pair up two iterators)
let pairs: Vec<_> = keys.iter().zip(values.iter()).collect();

// Flat map (flatten nested iterators)
let all_words: Vec<&str> = sentences.iter()
    .flat_map(|s| s.split_whitespace())
    .collect();

// Fold (reduce with accumulator)
let sum = nums.iter().fold(0, |acc, &x| acc + x);

// Windows and chunks
for window in data.windows(3) { ... }  // sliding window of size 3
for chunk in data.chunks(10) { ... }   // fixed-size chunks

// Partition (split into two groups)
let (evens, odds): (Vec<_>, Vec<_>) = nums.iter().partition(|&&x| x % 2 == 0);

// any / all (short-circuit)
let has_negative = nums.iter().any(|&x| x < 0);
let all_positive = nums.iter().all(|&x| x > 0);
```

### 2.3 When Manual Loops Are Better

```rust
// When you need early return with complex logic
for item in &items {
    if let Some(result) = try_process(item) {
        if result.is_critical() {
            return handle_critical(result);
        }
    }
}

// When you need mutable access to external state in complex ways
// (though iter_mut + for_each often works)
```

---

## 3. Error Handling Strategy

### 3.1 Library vs Application Error Handling

```rust
// LIBRARY: use thiserror -- callers need typed errors to match on
#[derive(Debug, thiserror::Error)]
pub enum LibError {
    #[error("connection failed: {0}")]
    Connection(String),
    #[error("timeout after {0}ms")]
    Timeout(u64),
    #[error("parse error")]
    Parse(#[from] serde_json::Error),
}

// APPLICATION: use anyhow -- just need error context for logging
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = load_config("app.toml")
        .context("failed to load application config")?;

    let conn = connect(&config.db_url)
        .context("failed to connect to database")?;

    Ok(())
}
```

### 3.2 Never Panic in Libraries

```rust
// BAD: library panics, crashing the application
pub fn parse(input: &str) -> Config {
    serde_json::from_str(input).unwrap()  // PANIC on invalid JSON
}

// GOOD: return Result, let caller decide
pub fn parse(input: &str) -> Result<Config, serde_json::Error> {
    serde_json::from_str(input)
}
```

### 3.3 Use expect() Over unwrap() When Panicking Is Acceptable

```rust
// BAD: no context in panic message
let port: u16 = env::var("PORT").unwrap().parse().unwrap();

// GOOD: panic message explains what went wrong
let port: u16 = env::var("PORT")
    .expect("PORT environment variable must be set")
    .parse()
    .expect("PORT must be a valid u16");
```

---

## 4. Performance

### 4.1 Zero-Cost Abstractions -- Trust the Compiler

```rust
// These compile to the SAME machine code:

// Iterator version
let sum: i32 = (0..1000).filter(|x| x % 2 == 0).sum();

// Manual loop version
let mut sum: i32 = 0;
for x in 0..1000 {
    if x % 2 == 0 { sum += x; }
}
```

### 4.2 Avoid Unnecessary Allocations

```rust
// BAD: allocates a new String just to compare
if input.to_lowercase() == "yes" { ... }

// GOOD: case-insensitive comparison without allocation
if input.eq_ignore_ascii_case("yes") { ... }

// BAD: collect into Vec just to iterate again
let filtered: Vec<_> = items.iter().filter(|x| x.valid).collect();
for item in &filtered { ... }

// GOOD: iterate directly -- no intermediate collection
for item in items.iter().filter(|x| x.valid) { ... }

// BAD: format! allocates a String
log::info!("{}", format!("request from {}", ip));

// GOOD: pass arguments directly
log::info!("request from {}", ip);
```

### 4.3 Pre-Allocate Collections

```rust
// BAD: starts with capacity 0, grows repeatedly
let mut results = Vec::new();
for item in &input {
    results.push(process(item));
}

// GOOD: allocate exact capacity upfront
let mut results = Vec::with_capacity(input.len());
for item in &input {
    results.push(process(item));
}

// BEST: use iterators (handles capacity automatically)
let results: Vec<_> = input.iter().map(process).collect();
```

### 4.4 Use Stack Allocation for Small Fixed-Size Data

```rust
// Heap allocation (Vec) -- unnecessary for known small sizes
let mut buf = Vec::with_capacity(256);

// Stack allocation (array) -- no heap, no GC pressure
let mut buf = [0u8; 256];

// For variable-size small data, consider smallvec or arrayvec crates
use smallvec::SmallVec;
let mut v: SmallVec<[i32; 8]> = SmallVec::new();
// Stores up to 8 elements on the stack, spills to heap only if more
```

### 4.5 String Performance

```rust
// BAD: O(n^2) string concatenation in a loop
let mut result = String::new();
for word in words {
    result = result + word + " ";  // allocates new String each iteration
}

// GOOD: O(n) with push_str
let mut result = String::with_capacity(total_len);
for word in words {
    result.push_str(word);
    result.push(' ');
}

// BEST: use join for slices
let result = words.join(" ");
```

---

## 5. Unsafe Usage Rules

### 5.1 When Unsafe Is Acceptable

```rust
// 1. FFI -- calling C libraries
extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

// 2. Performance-critical code with proven invariants
unsafe fn get_unchecked(slice: &[i32], index: usize) -> i32 {
    *slice.get_unchecked(index)  // skips bounds check
}

// 3. Implementing low-level abstractions (like Vec, HashMap)
```

### 5.2 Unsafe Rules

```rust
// 1. Minimize unsafe scope
// BAD: entire function is unsafe
unsafe fn process(data: *mut u8, len: usize) {
    // 50 lines of code, only 2 need unsafe
}

// GOOD: surgical unsafe blocks
fn process(data: *mut u8, len: usize) {
    let slice = unsafe { std::slice::from_raw_parts(data, len) };
    // rest of function is safe
    for &byte in slice { ... }
}

// 2. Document safety invariants
/// # Safety
/// - `ptr` must point to a valid, aligned `T`
/// - `ptr` must not be null
/// - The pointed-to memory must be initialized
unsafe fn read_ptr<T>(ptr: *const T) -> T {
    ptr.read()
}

// 3. Encapsulate unsafe behind safe APIs
pub struct SafeBuffer {
    ptr: *mut u8,
    len: usize,
}

impl SafeBuffer {
    pub fn get(&self, index: usize) -> Option<u8> {
        if index < self.len {
            Some(unsafe { *self.ptr.add(index) })
        } else {
            None
        }
    }
}
```

---

## 6. Testing Patterns

### 6.1 Unit Tests (In-Module)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_addition() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "division by zero")]
    fn test_divide_by_zero() {
        divide(1, 0);
    }

    #[test]
    fn test_result() -> Result<(), String> {
        let result = parse("42")?;
        assert_eq!(result, 42);
        Ok(())
    }
}
```

### 6.2 Property-Based Testing (proptest)

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_sort_is_sorted(mut v: Vec<i32>) {
        v.sort();
        for window in v.windows(2) {
            prop_assert!(window[0] <= window[1]);
        }
    }

    #[test]
    fn test_reverse_reverse_is_identity(v: Vec<i32>) {
        let mut reversed = v.clone();
        reversed.reverse();
        reversed.reverse();
        prop_assert_eq!(v, reversed);
    }
}
```

### 6.3 Benchmark with Criterion

```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_sort(c: &mut Criterion) {
    let data: Vec<i32> = (0..10000).rev().collect();

    c.bench_function("sort 10k", |b| {
        b.iter(|| {
            let mut d = data.clone();
            d.sort();
        })
    });
}

criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

---

## 7. Common Anti-Patterns

### 7.1 Fighting the Borrow Checker (Instead of Restructuring)

```rust
// ANTI-PATTERN: over-using Rc<RefCell<T>> to avoid borrow checker
let data = Rc::new(RefCell::new(vec![1, 2, 3]));
// This is often a sign that you should restructure your data

// BETTER: restructure to avoid shared mutable state
struct Processor {
    data: Vec<i32>,
    results: Vec<i32>,
}

impl Processor {
    fn process(&mut self) {
        self.results = self.data.iter().map(|x| x * 2).collect();
    }
}
```

### 7.2 Over-Using clone() to Silence the Compiler

```rust
// ANTI-PATTERN: clone everywhere
fn process(items: &[String]) {
    let first = items[0].clone();   // unnecessary clone
    let second = items[1].clone();  // unnecessary clone
    println!("{} {}", first, second);
}

// BETTER: just borrow
fn process(items: &[String]) {
    println!("{} {}", &items[0], &items[1]);
}
```

### 7.3 Using String When &str Would Do

```rust
// ANTI-PATTERN: unnecessary owned Strings
fn is_valid(name: String) -> bool {  // takes ownership needlessly
    !name.is_empty()
}

// BETTER: borrow
fn is_valid(name: &str) -> bool {
    !name.is_empty()
}
```

### 7.4 Ignoring Clippy Warnings

```rust
// Run clippy regularly
// cargo clippy -- -W clippy::all -W clippy::pedantic

// Common clippy catches:
// - Using .clone() when borrowing would work
// - Using manual loop when iterator method exists
// - Redundant closures: .map(|x| foo(x)) -> .map(foo)
// - Using & on already-borrowed values
// - Needless return statements
// - Single-character variable names in complex scopes
```

### 7.5 Unwrap in Production Code

```rust
// ANTI-PATTERN: unwrap everywhere
fn handle_request(req: Request) -> Response {
    let body = req.body().unwrap();              // panics on no body
    let data: Data = serde_json::from_str(&body).unwrap();  // panics on bad JSON
    let result = db.query(&data.id).unwrap();    // panics on DB error
    Response::ok(result)
}

// BETTER: proper error handling
fn handle_request(req: Request) -> Result<Response, AppError> {
    let body = req.body().ok_or(AppError::MissingBody)?;
    let data: Data = serde_json::from_str(&body)?;
    let result = db.query(&data.id)?;
    Ok(Response::ok(result))
}
```

---

## 8. Idiomatic Patterns

### 8.1 Builder Pattern

```rust
struct Server {
    host: String,
    port: u16,
    max_connections: usize,
}

struct ServerBuilder {
    host: String,
    port: u16,
    max_connections: usize,
}

impl ServerBuilder {
    fn new(host: impl Into<String>) -> Self {
        ServerBuilder {
            host: host.into(),
            port: 8080,
            max_connections: 100,
        }
    }

    fn port(mut self, port: u16) -> Self {
        self.port = port;
        self
    }

    fn max_connections(mut self, max: usize) -> Self {
        self.max_connections = max;
        self
    }

    fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            max_connections: self.max_connections,
        }
    }
}

let server = ServerBuilder::new("localhost")
    .port(9090)
    .max_connections(500)
    .build();
```

### 8.2 Type State Pattern (Compile-Time State Machines)

```rust
// States are types -- invalid transitions are compile errors
struct Locked;
struct Unlocked;

struct Door<State> {
    _state: std::marker::PhantomData<State>,
}

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> {
        Door { _state: std::marker::PhantomData }
    }
}

impl Door<Unlocked> {
    fn lock(self) -> Door<Locked> {
        Door { _state: std::marker::PhantomData }
    }

    fn open(&self) {
        println!("Door opened");
    }
}

let door = Door::<Locked> { _state: std::marker::PhantomData };
// door.open();      // COMPILE ERROR: Door<Locked> doesn't have open()
let door = door.unlock();
door.open();          // OK: Door<Unlocked> has open()
```

---

## Quick Reference: Decision Table

| Situation | Do This | Not This |
|---|---|---|
| Function parameter (read-only) | `&str`, `&[T]`, `&T` | `String`, `Vec<T>`, `T` (owned) |
| Need to modify parameter | `&mut T` | Clone and return new value |
| Return from function | Owned type (`String`, `Vec<T>`) | Reference (unless lifetime is clear) |
| Small Copy type | Pass by value | Pass by reference |
| Error in library | `thiserror` with typed variants | `anyhow` or `String` errors |
| Error in application | `anyhow` with `.context()` | Custom error types (overkill) |
| Multiple owners, single thread | `Rc<T>` | `Arc<T>` (unnecessary atomic overhead) |
| Multiple owners, multi thread | `Arc<T>` | `Rc<T>` (won't compile) |
| Interior mutability, single thread | `RefCell<T>` | `Mutex<T>` (unnecessary lock overhead) |
| Interior mutability, multi thread | `Mutex<T>` or `RwLock<T>` | `RefCell<T>` (won't compile) |
| Hot loop processing | Iterator chains | Manual indexing loops |
| String building in loop | `String::with_capacity` + `push_str` | `format!` or `+` in loop |
| Heterogeneous collection | `Vec<Box<dyn Trait>>` | Enum (if variants are known) |
| Known variants | Enum with match | `dyn Trait` (unnecessary indirection) |
| Unsafe code | Minimal scope, document invariants | Entire function marked unsafe |
| Production code | Proper error handling (Result + ?) | `.unwrap()` scattered everywhere |
