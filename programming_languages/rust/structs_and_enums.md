# Structs and Enums in Rust

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics](traits_and_generics.md) | [error handling (Result, Option)](error_handling_in_rust.md) | [lifetimes (structs holding references)](lifetimes_in_rust.md)

## Structs

### Named Structs

```rust
struct User {
    username: String,
    email: String,
    active: bool,
    sign_in_count: u64,
}

let user = User {
    username: String::from("alice"),
    email: String::from("alice@example.com"),
    active: true,
    sign_in_count: 1,
};
```

### Tuple Structs

Named types with unnamed fields. Useful for newtypes and strong typing.

```rust
struct Color(u8, u8, u8);
struct Meters(f64);

let red = Color(255, 0, 0);
let distance = Meters(42.0);
// let d: f64 = distance;  // COMPILE ERROR: Meters is not f64
let d: f64 = distance.0;   // access by index
```

### Unit Structs

Zero-sized types. Useful as marker types or for implementing traits on "nothing."

```rust
struct AlwaysEqual;

let _subject = AlwaysEqual;
// sizeof::<AlwaysEqual>() == 0 -- takes no memory at runtime
```

---

## Memory Layout of Structs

Structs are stored as contiguous blocks of memory with fields laid out by the compiler (NOT necessarily in declaration order).

```rust
struct Example {
    a: bool,    // 1 byte
    b: i64,     // 8 bytes
    c: bool,    // 1 byte
}
```

**Default layout (`Repr(Rust)`):** The compiler may reorder fields to minimize padding.

```
Compiler-optimized layout (may reorder):
Offset  Field   Size
0       b       8 bytes  (moved first for alignment)
8       a       1 byte
9       c       1 byte
10-15          6 bytes padding (struct aligned to 8)
Total: 16 bytes (compiler saved 8 bytes by reordering!)
```

**Compare with C layout (`#[repr(C)]`):** Fields stay in declaration order.

```
#[repr(C)] layout (declaration order preserved):
Offset  Field   Size   Padding
0       a       1      7 bytes padding (b needs 8-byte alignment)
8       b       8
16      c       1      7 bytes padding (struct aligned to largest = 8)
Total: 24 bytes
```

**How to think about it:** Rust's default layout is better for performance. Use `#[repr(C)]` only when you need FFI compatibility with C or exact control over layout.

```rust
use std::mem;
println!("Size: {}", mem::size_of::<Example>());      // 16 (Rust default)

#[repr(C)]
struct ExampleC { a: bool, b: i64, c: bool }
println!("Size: {}", mem::size_of::<ExampleC>());      // 24 (C layout)
```

---

## Struct Update Syntax

```rust
let user1 = User {
    username: String::from("alice"),
    email: String::from("alice@example.com"),
    active: true,
    sign_in_count: 1,
};

let user2 = User {
    email: String::from("bob@example.com"),
    ..user1   // remaining fields from user1
};
// CAUTION: user1.username was MOVED into user2 (String doesn't implement Copy)
// user1.active is still valid (bool implements Copy)
// println!("{}", user1.username);  // COMPILE ERROR: partially moved
```

---

## Impl Blocks (Methods and Associated Functions)

```rust
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // Associated function (like a static method / constructor)
    fn new(width: f64, height: f64) -> Self {
        Rectangle { width, height }
    }

    // Method with immutable borrow of self
    fn area(&self) -> f64 {
        self.width * self.height
    }

    // Method with mutable borrow of self
    fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }

    // Method that consumes self (takes ownership)
    fn into_square(self) -> Rectangle {
        let side = self.width.max(self.height);
        Rectangle { width: side, height: side }
    }
}

let mut r = Rectangle::new(10.0, 5.0);
println!("Area: {}", r.area());     // 50.0
r.scale(2.0);                        // mutates in place
let sq = r.into_square();            // r is moved, no longer usable
```

**`self` parameter determines ownership:**
- `&self` -- borrows immutably (most common)
- `&mut self` -- borrows mutably (for mutation)
- `self` -- takes ownership (consumes the value, used for transformations)

---

## Derive Macros

Automatically implement common traits:

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Point {
    x: i32,
    y: i32,
}

let p = Point { x: 1, y: 2 };
println!("{:?}", p);            // Debug: "Point { x: 1, y: 2 }"
let p2 = p.clone();             // Clone
assert_eq!(p, p2);              // PartialEq
```

Common derives: `Debug`, `Clone`, `Copy` (only if all fields are Copy), `PartialEq`, `Eq`, `Hash`, `Default`, `PartialOrd`, `Ord`.

---

## Enums (Algebraic Data Types)

Rust enums are **sum types** -- a value is one of several variants. Each variant can hold different data. This is far more powerful than C/Go enums (which are just named integers).

### Basic Enums

```rust
enum Direction {
    North,
    South,
    East,
    West,
}

let d = Direction::North;
```

### Enums with Data

```rust
enum Message {
    Quit,                          // no data (unit variant)
    Move { x: i32, y: i32 },      // named fields (struct variant)
    Write(String),                 // single value (tuple variant)
    ChangeColor(u8, u8, u8),       // multiple values (tuple variant)
}

let msg = Message::Write(String::from("hello"));
```

### Memory Layout of Enums

An enum is stored as: **discriminant** (tag) + **largest variant's data**.

```rust
enum Example {
    A(u8),          // 1 byte data
    B(i64),         // 8 bytes data
    C(u8, u8, u8),  // 3 bytes data
}
```

```
Memory layout:
┌──────────────┬──────────────────────┐
│ discriminant │ variant data         │
│ (1-8 bytes)  │ (size of largest)    │
└──────────────┴──────────────────────┘

For Example:
discriminant: identifies which variant (A=0, B=1, C=2)
data: 8 bytes (size of largest variant, which is B's i64)
Total: discriminant + padding + 8 = typically 16 bytes

When storing variant A(42):
┌─────┬─────────────────────────┐
│  0  │ 42 | padding...         │   only 1 byte used, rest is padding
└─────┴─────────────────────────┘

When storing variant B(1000):
┌─────┬─────────────────────────┐
│  1  │ 1000 (8 bytes)          │   full 8 bytes used
└─────┴─────────────────────────┘
```

**Niche optimization:** For `Option<&T>`, `Option<Box<T>>`, etc., the compiler uses the null pointer as the `None` discriminant. `Option<&T>` is the SAME size as `&T` (8 bytes) -- zero overhead!

```rust
use std::mem;
assert_eq!(mem::size_of::<&i32>(), 8);
assert_eq!(mem::size_of::<Option<&i32>>(), 8);  // niche optimization!
```

---

## Option<T> and Result<T, E>

The two most important enums in Rust. They replace null and exceptions.

### Option<T> -- Nullable Values

```rust
enum Option<T> {
    Some(T),
    None,
}

fn find_user(id: u32) -> Option<String> {
    if id == 1 {
        Some(String::from("Alice"))
    } else {
        None
    }
}

// You MUST handle the None case -- compiler enforces it
match find_user(1) {
    Some(name) => println!("Found: {}", name),
    None => println!("Not found"),
}
```

### Result<T, E> -- Recoverable Errors

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

**Related:** [error handling](error_handling_in_rust.md) for deep dive.

---

## Pattern Matching

### match (Exhaustive)

```rust
fn describe(msg: &Message) -> &str {
    match msg {
        Message::Quit => "quit",
        Message::Move { x, y } => "moving",
        Message::Write(text) => "writing",
        Message::ChangeColor(r, g, b) => "changing color",
    }
    // Every variant MUST be handled. The compiler enforces exhaustiveness.
}
```

### if let (Single Pattern)

```rust
let opt: Option<i32> = Some(42);

// Instead of a full match:
if let Some(value) = opt {
    println!("Got: {}", value);
}
// Equivalent to: match opt { Some(value) => ..., _ => () }
```

### while let (Loop Until Pattern Fails)

```rust
let mut stack = vec![1, 2, 3];

while let Some(top) = stack.pop() {
    println!("{}", top);   // 3, 2, 1
}
```

### Pattern Matching Guards and Bindings

```rust
let num = Some(42);

match num {
    Some(n) if n < 0 => println!("negative"),    // guard
    Some(n @ 0..=100) => println!("0-100: {}", n), // binding with range
    Some(n) => println!("large: {}", n),
    None => println!("nothing"),
}
```

---

## Newtype Pattern

Wrap an existing type to create a new type with different semantics. Zero runtime cost.

```rust
struct Meters(f64);
struct Seconds(f64);

fn speed(distance: Meters, time: Seconds) -> f64 {
    distance.0 / time.0
}

let d = Meters(100.0);
let t = Seconds(9.58);
// speed(t, d);  // COMPILE ERROR: wrong order -- types prevent bugs
speed(d, t);     // correct
```

**How to think about it:** The newtype is a zero-cost wrapper. `Meters(f64)` has exactly the same memory layout as `f64`. The type distinction exists only at compile time.

---

## When to Use Struct vs Enum

| Use | When |
|---|---|
| **Struct** | You have a fixed set of fields that are ALL present (product type) |
| **Enum** | A value can be ONE OF several variants (sum type) |
| **Struct with Option fields** | Some fields are optional but the overall shape is fixed |
| **Enum with struct variants** | Each variant has different structured data |

```rust
// Struct: a user always has all these fields
struct User {
    name: String,
    email: String,
    age: u32,
}

// Enum: a shape is exactly one of these
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}
```

---

## Practical Examples / LeetCode Patterns

### Binary Tree (LeetCode 104: Maximum Depth) -- Enum as Recursive Type

```rust
use std::cell::RefCell;
use std::rc::Rc;

// The standard LeetCode tree definition
type TreeNode = Option<Rc<RefCell<TreeNodeInner>>>;

struct TreeNodeInner {
    val: i32,
    left: TreeNode,
    right: TreeNode,
}

fn max_depth(root: &TreeNode) -> i32 {
    match root {
        None => 0,
        Some(node) => {
            let node = node.borrow();
            1 + max_depth(&node.left).max(max_depth(&node.right))
        }
    }
}
```

**Enum insight:** `Option<Rc<RefCell<TreeNodeInner>>>` is the idiomatic "nullable tree node." `None` represents a null child. The niche optimization means `Option<Rc<...>>` is the same size as `Rc<...>`.

### Linked List -- Enum as Recursive Sum Type

```rust
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}

use List::{Cons, Nil};

fn sum(list: &List<i32>) -> i32 {
    match list {
        Nil => 0,
        Cons(val, rest) => val + sum(rest),
    }
}

let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
assert_eq!(sum(&list), 6);
```

**Why `Box` is needed:** Without `Box`, `List` would have infinite size (it contains itself). `Box<List<T>>` is a heap pointer (8 bytes), making the size finite.

### State Machine -- Enum with Exhaustive Matching

```rust
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: String },
    Error { message: String },
}

fn handle_state(state: &ConnectionState) -> &str {
    match state {
        ConnectionState::Disconnected => "idle",
        ConnectionState::Connecting { attempt } if *attempt > 3 => "giving up",
        ConnectionState::Connecting { .. } => "retrying",
        ConnectionState::Connected { .. } => "active",
        ConnectionState::Error { message } => message,
    }
}
```

**Enum insight:** The compiler guarantees every state is handled. If you add a new variant, every `match` in the codebase that doesn't handle it becomes a compile error. This makes state machines much safer than integer flags or string states.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Struct layout | Compiler may reorder fields to minimize padding. Use `#[repr(C)]` for FFI |
| Enum layout | Discriminant + largest variant. `Option<&T>` is niche-optimized to pointer size |
| Pattern matching | `match` is exhaustive -- compiler forces you to handle all cases |
| Option/Result | Replace null and exceptions. Compiler-enforced error handling |
| Newtype | Zero-cost wrapper for type safety. `Meters(f64)` prevents unit confusion |
| impl blocks | `&self` borrows, `&mut self` mutates, `self` consumes |
| if let / while let | Syntactic sugar for single-pattern matching |
