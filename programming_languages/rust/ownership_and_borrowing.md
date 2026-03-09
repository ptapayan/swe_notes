# Ownership and Borrowing in Rust

**Related:** [Rust as a language](rust_as_a_language.md) | [lifetimes](lifetimes_in_rust.md) | [smart pointers](smart_pointers.md) | [structs and enums](structs_and_enums.md) | [best practices](rust_best_practices.md)

## The Three Ownership Rules

This is THE foundational concept in Rust. Every other feature flows from these rules:

```
1. Each value in Rust has exactly ONE owner (a variable binding)
2. When the owner goes out of scope, the value is DROPPED (memory freed)
3. Ownership can be MOVED (transferred) or BORROWED (referenced)
```

---

## Move Semantics

### What Happens When You Assign a Heap-Allocated Value

```rust
let s1 = String::from("hello");
let s2 = s1;   // s1's ownership MOVES to s2
// println!("{}", s1);  // COMPILE ERROR: borrow of moved value: `s1`
```

**What's happening in memory:**

```
BEFORE (let s1 = String::from("hello")):
Stack:                      Heap:
s1: ┌──────────────┐       ┌───┬───┬───┬───┬───┐
    │ ptr ──────────┼──────>│ h │ e │ l │ l │ o │
    │ len: 5        │       └───┴───┴───┴───┴───┘
    │ cap: 5        │
    └──────────────┘

AFTER (let s2 = s1):
Stack:                      Heap:
s1: [INVALID/moved]         ┌───┬───┬───┬───┬───┐
s2: ┌──────────────┐   ┌───>│ h │ e │ l │ l │ o │
    │ ptr ──────────┼───┘   └───┴───┴───┴───┴───┘
    │ len: 5        │
    │ cap: 5        │
    └──────────────┘
```

**How to think about it:** A move copies the 24-byte stack metadata (ptr + len + cap) and **invalidates** the source. The heap data does NOT move. This is O(1) regardless of string length. The compiler enforces that `s1` is never used again.

### Move Into Function

```rust
fn take_ownership(s: String) {
    println!("{}", s);
}   // s is dropped here -- heap memory freed

fn main() {
    let s = String::from("hello");
    take_ownership(s);       // ownership moves into the function
    // println!("{}", s);    // COMPILE ERROR: value used after move
}
```

### Move Out of Function (Return)

```rust
fn give_ownership() -> String {
    let s = String::from("hello");
    s   // ownership moves to the caller
}       // s is NOT dropped here -- it was moved out

fn main() {
    let s = give_ownership();  // s now owns the String
    println!("{}", s);         // works fine
}   // s is dropped here
```

---

## Copy vs Clone

### Copy Trait (Stack-Only, Bitwise Copy)

Types that implement `Copy` are duplicated automatically on assignment -- no move occurs.

```rust
let x: i32 = 42;
let y = x;          // x is COPIED (not moved)
println!("{}", x);  // still valid! i32 implements Copy

let a = true;
let b = a;          // bool implements Copy too
println!("{}", a);  // works fine
```

**Types that implement Copy:** All integer types, `bool`, `f32`/`f64`, `char`, tuples of Copy types, fixed-size arrays of Copy types, references (`&T`).

**Types that do NOT implement Copy:** `String`, `Vec<T>`, `Box<T>` -- anything that owns heap data. If it has a `Drop` impl, it cannot be `Copy`.

**How to think about it:** `Copy` means "this type is just a fixed number of bytes on the stack with no heap resources. Duplicating the bytes is the same as duplicating the value." A `String` can't be `Copy` because bitwise-copying its 24 bytes would create two owners of the same heap allocation -- double free.

### Clone Trait (Explicit Deep Copy)

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // explicitly deep-copies the heap data
println!("{}", s1);    // works -- s1 was not moved
println!("{}", s2);    // works -- s2 has its own heap allocation
```

**What's happening in memory after clone:**

```
Stack:                        Heap:
s1: ┌──────────────┐        ┌───┬───┬───┬───┬───┐
    │ ptr ──────────┼───────>│ h │ e │ l │ l │ o │   (original)
    │ len: 5        │        └───┴───┴───┴───┴───┘
    │ cap: 5        │
    └──────────────┘
s2: ┌──────────────┐        ┌───┬───┬───┬───┬───┐
    │ ptr ──────────┼───────>│ h │ e │ l │ l │ o │   (new copy)
    │ len: 5        │        └───┴───┴───┴───┴───┘
    │ cap: 5        │
    └──────────────┘
```

`Clone` is explicit because it's potentially expensive (heap allocation + memcpy).

---

## Borrowing: References

Instead of moving ownership, you can **borrow** a value via references.

### Shared References (`&T`) -- Immutable Borrow

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
}   // s goes out of scope but doesn't drop the String (it's just a reference)

fn main() {
    let s = String::from("hello");
    let len = calculate_length(&s);  // borrow s (shared, immutable)
    println!("{} has length {}", s, len);  // s is still valid
}
```

**What's happening in memory:**

```
Stack:                           Heap:
s:   ┌──────────────┐          ┌───┬───┬───┬───┬───┐
     │ ptr ──────────┼────────>│ h │ e │ l │ l │ o │
     │ len: 5        │         └───┴───┴───┴───┴───┘
     │ cap: 5        │
     └──────────────┘
          ^
&s:  ┌───┼───────────┐
     │ ptr (to s)    │   <-- reference is just a pointer (8 bytes)
     └───────────────┘
```

**Rules for shared references:**
- You can have **any number** of `&T` at the same time
- The data CANNOT be mutated through a `&T`
- The referent must outlive the reference

### Mutable References (`&mut T`) -- Exclusive Borrow

```rust
fn push_world(s: &mut String) {
    s.push_str(", world");  // can modify through &mut
}

fn main() {
    let mut s = String::from("hello");
    push_world(&mut s);
    println!("{}", s);  // "hello, world"
}
```

**The exclusivity rule (critical):**

```rust
let mut s = String::from("hello");

let r1 = &s;         // OK: shared reference
let r2 = &s;         // OK: multiple shared references
// let r3 = &mut s;  // COMPILE ERROR: cannot borrow as mutable
                      // because it is also borrowed as immutable

println!("{} {}", r1, r2);  // after this, r1 and r2 are no longer used

let r3 = &mut s;     // OK now: r1 and r2's lifetimes have ended (NLL)
r3.push_str("!");
```

**Why this rule exists:** It prevents data races at compile time. A data race requires:
1. Two or more pointers accessing the same data
2. At least one is writing
3. No synchronization

Rust's rule makes condition (1)+(2) impossible: you can have many readers OR one writer, never both.

---

## Borrow Checker Rules Summary

```
Rule 1: At any given time, you can have EITHER:
        - Any number of immutable references (&T), OR
        - Exactly ONE mutable reference (&mut T)
        ...but NOT both simultaneously

Rule 2: References must always be VALID (no dangling references)
        The referent must outlive the reference
```

---

## Common Borrow Checker Errors and Fixes

### Error: Cannot Borrow as Mutable While Immutably Borrowed

```rust
// BROKEN:
let mut v = vec![1, 2, 3];
let first = &v[0];      // immutable borrow
v.push(4);               // ERROR: mutable borrow while immutable borrow exists
println!("{}", first);

// WHY: push() might reallocate the Vec, invalidating `first`

// FIX 1: Use the immutable reference before mutating
let mut v = vec![1, 2, 3];
let first = v[0];        // COPY the value (i32 implements Copy)
v.push(4);               // now fine
println!("{}", first);

// FIX 2: Scope the borrow
let mut v = vec![1, 2, 3];
{
    let first = &v[0];
    println!("{}", first);
}   // immutable borrow ends here
v.push(4);               // now fine
```

### Error: Cannot Move Out of Borrowed Content

```rust
// BROKEN:
fn take(v: &Vec<String>) -> String {
    v[0]   // ERROR: cannot move out of borrowed content
}

// FIX 1: Clone
fn take(v: &Vec<String>) -> String {
    v[0].clone()
}

// FIX 2: Return a reference
fn take(v: &Vec<String>) -> &String {
    &v[0]
}
```

### Error: Dangling Reference

```rust
// BROKEN:
fn dangle() -> &String {
    let s = String::from("hello");
    &s   // ERROR: s is dropped at end of function, reference would dangle
}

// FIX: Return owned value
fn no_dangle() -> String {
    let s = String::from("hello");
    s    // move ownership out
}
```

---

## Non-Lexical Lifetimes (NLL)

Since Rust 2018, the borrow checker tracks when a reference is **last used**, not when it goes out of scope.

```rust
let mut s = String::from("hello");
let r1 = &s;
println!("{}", r1);      // r1's lifetime ENDS here (last use)

let r2 = &mut s;         // OK! r1 is no longer alive
r2.push_str(" world");
```

**Before NLL (Rust 2015):** Lifetimes extended to the end of the enclosing block, causing unnecessary borrow conflicts. NLL made the borrow checker much more ergonomic.

---

## Interior Mutability: Cell and RefCell

Sometimes you need to mutate data behind a shared reference (`&T`). This is called **interior mutability** -- it moves the borrow checking from compile time to runtime.

### Cell<T> (Copy Types Only, No Runtime Cost)

```rust
use std::cell::Cell;

let c = Cell::new(5);    // no mut needed on the binding
c.set(10);               // mutate through shared reference
println!("{}", c.get()); // 10
```

`Cell` works by copying the value in and out. No runtime borrow checks. Only works for `Copy` types.

### RefCell<T> (Runtime Borrow Checking)

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// Borrow immutably at runtime
let borrowed = data.borrow();
println!("{:?}", *borrowed);
drop(borrowed);  // must drop before mutable borrow

// Borrow mutably at runtime
let mut mut_borrowed = data.borrow_mut();
mut_borrowed.push(4);
```

**Runtime panics:** If you violate borrow rules at runtime (e.g., `borrow_mut()` while a `borrow()` is active), RefCell **panics**. The rules are the same -- one writer XOR many readers -- but enforced at runtime instead of compile time.

**When to use:** When you need mutation through `&self` (common in trait impls), or in complex data structures where the compiler cannot prove safety.

**Related:** [smart pointers](smart_pointers.md) | [lifetimes](lifetimes_in_rust.md)

---

## Practical Examples / LeetCode Patterns

### Two Sum (LeetCode 1) -- HashMap with Borrowing

```rust
use std::collections::HashMap;

fn two_sum(nums: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::with_capacity(nums.len());
    for (i, &num) in nums.iter().enumerate() {
        let complement = target - num;
        if let Some(&j) = seen.get(&complement) {
            return Some((j, i));
        }
        seen.insert(num, i);
    }
    None
}
```

**Ownership insight:** `nums: &[i32]` borrows the slice -- no ownership transfer. `i32` implements `Copy`, so `num` is copied out of the slice. The HashMap owns its keys and values (both `i32` and `usize`, which are `Copy` types, so insert copies them in).

### Reverse String In-Place (LeetCode 344) -- Mutable Borrow

```rust
fn reverse_string(s: &mut Vec<char>) {
    let (mut left, mut right) = (0, s.len() - 1);
    while left < right {
        s.swap(left, right);  // swap uses &mut self
        left += 1;
        right -= 1;
    }
}
```

**Ownership insight:** `&mut Vec<char>` gives exclusive mutable access. `s.swap()` is safe because we have the only mutable reference. No cloning needed -- pure in-place mutation.

### Valid Parentheses (LeetCode 20) -- Owned Stack

```rust
fn is_valid(s: &str) -> bool {
    let mut stack: Vec<char> = Vec::new();
    for c in s.chars() {
        match c {
            '(' => stack.push(')'),
            '{' => stack.push('}'),
            '[' => stack.push(']'),
            _ => {
                if stack.pop() != Some(c) {
                    return false;
                }
            }
        }
    }
    stack.is_empty()
}
```

**Ownership insight:** `s: &str` borrows the string. `stack` is an owned `Vec<char>` on the stack. `chars()` iterates by value (char is Copy). When the function returns, `stack` is dropped automatically.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Ownership | Each value has one owner. Value dropped when owner goes out of scope |
| Move | Assignment of heap types transfers ownership. Source becomes invalid |
| Copy | Stack-only types (integers, bool, references) are bitwise copied, not moved |
| Clone | Explicit deep copy for heap types. Potentially expensive |
| `&T` | Shared reference. Many allowed. Cannot mutate. Must not outlive referent |
| `&mut T` | Exclusive reference. Only one allowed. Can mutate. No other references allowed |
| NLL | Borrow lifetimes end at last use, not at end of scope |
| Interior mutability | `Cell` (Copy types), `RefCell` (runtime borrow checks). Escape hatch for `&T` mutation |
