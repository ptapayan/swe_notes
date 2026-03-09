# Traits and Generics in Rust

**Related:** [structs and enums](structs_and_enums.md) | [ownership and borrowing](ownership_and_borrowing.md) | [lifetimes](lifetimes_in_rust.md) | [smart pointers (Deref, Drop)](smart_pointers.md) | [collections (Iterator)](collections_in_rust.md)

## What Is a Trait?

A trait defines a set of methods that a type can implement. Like Go interfaces, but more powerful -- traits can have default implementations, associated types, generic parameters, and const members.

```rust
trait Summary {
    fn summarize(&self) -> String;

    // Default implementation (can be overridden)
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..20])
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, self.content)
    }
    // preview() uses the default implementation
}
```

**Key difference from Go:** In Go, interfaces are satisfied implicitly (structural typing). In Rust, you must explicitly write `impl Trait for Type`. This is intentional -- explicit implementation prevents accidental interface satisfaction and enables the orphan rule.

---

## Trait Bounds

Constrain generic types to only accept types that implement specific traits.

```rust
// Syntax 1: Trait bound in angle brackets
fn notify<T: Summary>(item: &T) {
    println!("Breaking: {}", item.summarize());
}

// Syntax 2: impl Trait (sugar for simple cases)
fn notify(item: &impl Summary) {
    println!("Breaking: {}", item.summarize());
}

// Syntax 3: where clause (cleaner for complex bounds)
fn notify<T>(item: &T)
where
    T: Summary + Display,
{
    println!("Breaking: {}", item.summarize());
}

// Multiple bounds
fn process<T: Summary + Clone + Debug>(item: T) { ... }
```

**When to use which:**
- `impl Trait` -- simple, one generic parameter, quick functions
- `T: Trait` -- when you need to refer to `T` multiple times in the signature
- `where` clause -- when bounds are complex or numerous (keeps function signature readable)

---

## Generics and Monomorphization

Generics in Rust are compiled via **monomorphization** -- the compiler generates a specialized version of the function for each concrete type used.

```rust
fn largest<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}

// When you call:
largest(5i32, 10i32);       // compiler generates largest_i32
largest(3.14f64, 2.71f64);  // compiler generates largest_f64
```

**What's happening at compile time:**

```
Source code:                  After monomorphization:

fn largest<T: Ord>            fn largest_i32(a: i32, b: i32) -> i32
                              fn largest_f64(a: f64, b: f64) -> f64
                              fn largest_String(a: String, b: String) -> String
```

**Zero-cost abstraction:** Each monomorphized function is fully optimized by LLVM -- no dynamic dispatch, no vtable lookup, no boxing. The generic version is exactly as fast as if you wrote each specialized version by hand.

**Trade-off:** Monomorphization increases binary size (code bloat). If `largest` is used with 50 different types, 50 copies exist in the binary. This is the same trade-off as C++ templates.

---

## Associated Types

An alternative to generic parameters for traits where there's only one logical implementation per type.

```rust
// With associated type (cleaner)
trait Iterator {
    type Item;   // associated type
    fn next(&mut self) -> Option<Self::Item>;
}

// Usage: no need to specify Item when calling
impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        // ...
    }
}

// Compare with generic parameter (clunkier)
trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
// Would allow impl Iterator<u32> AND impl Iterator<String> for same type
// Usually not what you want
```

**When to use associated types vs generics:**
- **Associated type:** One logical implementation per type (Iterator has one Item type)
- **Generic parameter:** Multiple implementations possible (e.g., `From<T>` can be implemented for many `T`)

---

## Trait Objects (Dynamic Dispatch)

When you need runtime polymorphism (don't know the concrete type at compile time), use trait objects with `dyn Trait`.

```rust
fn print_all(items: &[&dyn Summary]) {
    for item in items {
        println!("{}", item.summarize());
    }
}
```

### vtable and Fat Pointers

A `&dyn Trait` is a **fat pointer** -- two words (16 bytes):

```
&dyn Summary:
┌───────────────────┐
│ data pointer (8B) │ ──→ actual concrete value
├───────────────────┤
│ vtable ptr   (8B) │ ──→ vtable for this type's Summary impl
└───────────────────┘

vtable:
┌──────────────────────────┐
│ drop function pointer    │
│ size of concrete type    │
│ alignment                │
│ summarize() fn pointer   │  ← dynamic dispatch through this
│ preview() fn pointer     │
└──────────────────────────┘
```

**How to think about it:** Static dispatch (generics) = compiler knows the type, inlines the call. Dynamic dispatch (`dyn Trait`) = runtime vtable lookup, cannot inline, but enables heterogeneous collections.

### Static vs Dynamic Dispatch Comparison

```rust
// STATIC dispatch (monomorphization) -- no runtime cost
fn process_static(item: &impl Summary) {
    println!("{}", item.summarize());
}
// Compiler generates: process_static_Article, process_static_Tweet, etc.

// DYNAMIC dispatch (vtable) -- runtime cost but flexible
fn process_dynamic(item: &dyn Summary) {
    println!("{}", item.summarize());
}
// One function, dispatches through vtable at runtime
```

| Aspect | Static (`impl Trait` / generics) | Dynamic (`dyn Trait`) |
|---|---|---|
| Dispatch | Compile-time (inlined) | Runtime (vtable lookup) |
| Binary size | Larger (one copy per type) | Smaller (one function) |
| Heterogeneous collections | No | Yes (`Vec<Box<dyn Trait>>`) |
| Performance | Faster (no indirection) | Slightly slower (~1-2ns vtable lookup) |

---

## Object Safety

Not all traits can be used as `dyn Trait`. A trait is **object-safe** if:

1. All methods take `&self`, `&mut self`, or `self: Box<Self>` (not `Self` by value)
2. No methods return `Self` (the concrete type is erased behind `dyn`)
3. No methods have generic type parameters
4. No associated functions without `self`

```rust
// Object-safe
trait Draw {
    fn draw(&self);
}

// NOT object-safe (returns Self)
trait Clone {
    fn clone(&self) -> Self;  // can't know the size of Self behind dyn
}
```

---

## Orphan Rule

You can implement a trait for a type ONLY if either the trait or the type is defined in your crate. This prevents conflicting implementations across crates.

```rust
// OK: your trait, external type
impl MyTrait for Vec<i32> { ... }

// OK: external trait, your type
impl Display for MyStruct { ... }

// COMPILE ERROR: external trait, external type
impl Display for Vec<i32> { ... }  // neither Display nor Vec is yours
```

**Workaround:** Use the newtype pattern to wrap the external type.

```rust
struct MyVec(Vec<i32>);
impl Display for MyVec { ... }  // OK: MyVec is your type
```

---

## Common Standard Library Traits

### Display and Debug

```rust
use std::fmt;

// Debug -- for programmers (usually derived)
#[derive(Debug)]
struct Point { x: i32, y: i32 }
println!("{:?}", Point { x: 1, y: 2 });  // "Point { x: 1, y: 2 }"

// Display -- for users (must implement manually)
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
println!("{}", Point { x: 1, y: 2 });    // "(1, 2)"
```

### From and Into

```rust
// From -- convert from another type
impl From<(i32, i32)> for Point {
    fn from(tuple: (i32, i32)) -> Self {
        Point { x: tuple.0, y: tuple.1 }
    }
}

let p: Point = Point::from((1, 2));
let p: Point = (1, 2).into();   // Into is auto-implemented when From exists
```

### Iterator

```rust
struct Counter {
    count: u32,
    max: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

// Once you implement Iterator, you get 70+ methods for free:
let sum: u32 = Counter { count: 0, max: 5 }
    .filter(|&x| x % 2 == 0)
    .map(|x| x * x)
    .sum();
```

### Clone, Copy, PartialEq, Eq, Hash

```rust
#[derive(Clone, Copy, PartialEq, Eq, Hash)]
struct Coord {
    x: i32,
    y: i32,
}
// Copy: bitwise copy on assignment (only if all fields are Copy)
// Clone: explicit .clone() (required to derive Copy)
// PartialEq: enables == and !=
// Eq: marker that equality is reflexive (x == x is always true)
// Hash: enables use as HashMap key (requires Eq)
```

### PartialOrd and Ord

```rust
// PartialOrd: enables <, >, <=, >= (may have incomparable values, like f64::NAN)
// Ord: total ordering (every pair is comparable, required for BTreeMap keys)

#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Score(u32);
```

---

## Supertraits

A trait can require that implementors also implement another trait.

```rust
trait Animal: Display + Debug {
    fn name(&self) -> &str;
}

// Any type implementing Animal MUST also implement Display and Debug
```

This is NOT inheritance. It's a constraint: "to be an `Animal`, you must also be `Display` and `Debug`."

---

## Practical Examples / LeetCode Patterns

### Custom Sort with Ord (LeetCode 56: Merge Intervals)

```rust
fn merge(mut intervals: Vec<Vec<i32>>) -> Vec<Vec<i32>> {
    intervals.sort_by_key(|iv| iv[0]);  // sort by start (uses Ord on i32)
    let mut result: Vec<Vec<i32>> = Vec::new();

    for interval in intervals {
        if let Some(last) = result.last_mut() {
            if last[1] >= interval[0] {
                last[1] = last[1].max(interval[1]);
                continue;
            }
        }
        result.push(interval);
    }
    result
}
```

**Trait insight:** `sort_by_key` requires the key type to implement `Ord`. `i32` implements `Ord`, so this works. `f64` does NOT implement `Ord` (because `NaN != NaN`), so you'd need `sort_by` with a custom comparator.

### Iterator Chains (LeetCode 347: Top K Frequent Elements)

```rust
use std::collections::HashMap;

fn top_k_frequent(nums: Vec<i32>, k: i32) -> Vec<i32> {
    let mut freq: HashMap<i32, usize> = HashMap::new();
    for &n in &nums {
        *freq.entry(n).or_insert(0) += 1;
    }

    let mut pairs: Vec<_> = freq.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1));  // sort by frequency descending

    pairs.into_iter()
        .take(k as usize)
        .map(|(num, _)| num)
        .collect()
}
```

**Trait insight:** `into_iter()` returns an iterator that takes ownership. `take()`, `map()`, `collect()` are all zero-cost iterator adapters that fuse into a single loop at compile time. `collect()` uses the `FromIterator` trait to know what type to build.

### Trait Objects for Heterogeneous Collections (Strategy Pattern)

```rust
trait Scorer {
    fn score(&self, text: &str) -> f64;
}

struct LengthScorer;
impl Scorer for LengthScorer {
    fn score(&self, text: &str) -> f64 { text.len() as f64 }
}

struct KeywordScorer { keyword: String }
impl Scorer for KeywordScorer {
    fn score(&self, text: &str) -> f64 {
        text.matches(&self.keyword).count() as f64
    }
}

fn rank(text: &str, scorers: &[Box<dyn Scorer>]) -> f64 {
    scorers.iter().map(|s| s.score(text)).sum()
}
```

**Trait object insight:** `Box<dyn Scorer>` is a fat pointer (data ptr + vtable ptr). The `scorers` slice can hold different concrete types. Each `.score()` call goes through the vtable. This is the Rust equivalent of Go's interface-based polymorphism.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Traits | Like interfaces but with default impls, associated types, and const members |
| Generics | Monomorphized at compile time. Zero runtime cost. Increases binary size |
| `impl Trait` | Sugar for simple trait bounds. Static dispatch |
| `dyn Trait` | Fat pointer (data + vtable). Dynamic dispatch. Enables heterogeneous collections |
| Associated types | One implementation per type. Cleaner than generic parameters for Iterator-like traits |
| Orphan rule | Either the trait or the type must be in your crate |
| Object safety | No `Self` returns, no generics in methods, no associated functions without `self` |
| Common traits | `Debug`, `Display`, `Clone`, `Copy`, `From`, `Iterator`, `PartialEq`, `Eq`, `Hash`, `Ord` |
