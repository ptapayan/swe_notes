# Smart Pointers in Rust

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics (Deref, Drop)](traits_and_generics.md) | [concurrency (Arc, Mutex)](concurrency_in_rust.md) | [lifetimes](lifetimes_in_rust.md)

## What Is a Smart Pointer?

A smart pointer is a struct that acts like a pointer but owns the data it points to and has additional metadata/behavior. Regular references (`&T`) borrow data. Smart pointers **own** data.

Smart pointers implement `Deref` (auto-dereference, so they act like references) and `Drop` (cleanup when the pointer goes out of scope).

---

## Box<T> -- Heap Allocation

`Box<T>` is the simplest smart pointer -- it allocates data on the heap and owns it. When the `Box` goes out of scope, the heap data is freed.

```rust
let b = Box::new(5);       // allocates i32 on the heap
println!("{}", b);          // auto-derefs via Deref trait
// b goes out of scope -- heap memory freed via Drop
```

### Memory Layout

```
Stack:               Heap:
b: ┌───────────┐    ┌─────┐
   │ ptr ──────┼───>│  5  │   (i32 = 4 bytes)
   └───────────┘    └─────┘
   8 bytes           4 bytes
```

`Box<T>` is 8 bytes on the stack (one pointer). The data lives on the heap.

### When to Use Box

**1. Recursive types (must have known size):**

```rust
// COMPILE ERROR: List has infinite size
// enum List { Cons(i32, List), Nil }

// FIX: Box has known size (8 bytes)
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

```
Memory layout of Cons(1, Cons(2, Cons(3, Nil))):

Stack:
┌───────────────────┐
│ Cons              │
│  val: 1           │
│  next: Box ───────┼──┐
└───────────────────┘  │
                       v    Heap:
                    ┌───────────────────┐
                    │ Cons              │
                    │  val: 2           │
                    │  next: Box ───────┼──┐
                    └───────────────────┘  │
                                           v    Heap:
                                        ┌──────┐
                                        │ Nil  │
                                        └──────┘
```

**2. Large data you don't want on the stack:**

```rust
// 1 MB array on the stack -- might overflow
// let big = [0u8; 1_000_000];

// 1 MB on the heap -- fine
let big = Box::new([0u8; 1_000_000]);
```

**3. Trait objects:**

```rust
let animal: Box<dyn Animal> = Box::new(Dog { name: "Rex".into() });
// Box<dyn Animal> is 16 bytes: data pointer + vtable pointer
```

---

## Rc<T> -- Reference Counting (Single-Threaded)

`Rc<T>` enables multiple owners of the same data. The data is freed when the last `Rc` is dropped.

```rust
use std::rc::Rc;

let a = Rc::new(vec![1, 2, 3]);
let b = Rc::clone(&a);   // increments reference count (does NOT deep-copy)
let c = Rc::clone(&a);   // reference count is now 3

println!("{:?}", a);      // [1, 2, 3]
println!("count: {}", Rc::strong_count(&a));  // 3

drop(c);  // count drops to 2
drop(b);  // count drops to 1
drop(a);  // count drops to 0 -- data is freed
```

### Memory Layout

```
Stack:                     Heap:
a: ┌──────────┐          ┌─────────────────────┐
   │ ptr ─────┼────┐     │ strong_count: 3      │
   └──────────┘    │     │ weak_count: 0         │
b: ┌──────────┐    ├────>│ data: Vec<i32>        │
   │ ptr ─────┼────┤     │   ptr ───────────────┼──> [1, 2, 3]
   └──────────┘    │     │   len: 3              │
c: ┌──────────┐    │     │   cap: 3              │
   │ ptr ─────┼────┘     └─────────────────────┘
   └──────────┘
```

All three `Rc` pointers (8 bytes each on the stack) point to the same heap allocation. `Rc::clone` is O(1) -- it just increments the counter.

### Rc Is NOT Thread-Safe

```rust
use std::rc::Rc;
use std::thread;

let rc = Rc::new(5);
// thread::spawn(move || {
//     println!("{}", rc);
// });
// COMPILE ERROR: `Rc<i32>` cannot be sent between threads safely
// Rc does not implement Send
```

**Why:** `Rc` uses non-atomic operations to increment/decrement the count. If two threads increment simultaneously, the count could become corrupted (lost update). Use `Arc` for threads.

---

## Arc<T> -- Atomic Reference Counting (Thread-Safe)

Same as `Rc`, but uses **atomic operations** for the reference count, making it safe to share across threads.

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(vec![1, 2, 3]);
let mut handles = vec![];

for _ in 0..3 {
    let data = Arc::clone(&data);
    handles.push(thread::spawn(move || {
        println!("{:?}", data);  // immutable access across threads
    }));
}

for handle in handles {
    handle.join().unwrap();
}
```

### Rc vs Arc Cost

| Operation | Rc | Arc |
|---|---|---|
| Clone (increment) | `count += 1` (non-atomic) | `count.fetch_add(1, Relaxed)` (atomic) |
| Drop (decrement) | `count -= 1` (non-atomic) | `count.fetch_sub(1, Release)` (atomic) |
| Read access | Same | Same |
| Thread-safe | No | Yes |
| Overhead | ~1 CPU cycle | ~10-30 CPU cycles per clone/drop |

**Rule:** Use `Rc` in single-threaded code, `Arc` only when you need to share across threads.

---

## RefCell<T> -- Interior Mutability (Runtime Borrow Checking)

`RefCell<T>` allows mutable borrows checked at **runtime** instead of compile time. It panics if you violate the borrowing rules.

```rust
use std::cell::RefCell;

let data = RefCell::new(vec![1, 2, 3]);

// Runtime borrow checks
{
    let r1 = data.borrow();      // immutable borrow -- OK
    let r2 = data.borrow();      // second immutable borrow -- OK
    println!("{:?} {:?}", r1, r2);
}   // r1, r2 dropped -- borrows released

{
    let mut w = data.borrow_mut();  // mutable borrow -- OK (no other borrows active)
    w.push(4);
}

// RUNTIME PANIC: mutable borrow while immutable borrow active
// {
//     let r = data.borrow();
//     let w = data.borrow_mut();  // PANIC at runtime
// }
```

### Common Pattern: Rc<RefCell<T>>

Single-threaded, multiple owners with interior mutability.

```rust
use std::rc::Rc;
use std::cell::RefCell;

let shared_data = Rc::new(RefCell::new(vec![1, 2, 3]));

let a = Rc::clone(&shared_data);
let b = Rc::clone(&shared_data);

a.borrow_mut().push(4);  // mutate through a
b.borrow_mut().push(5);  // mutate through b

println!("{:?}", shared_data.borrow());  // [1, 2, 3, 4, 5]
```

**Memory layout of Rc<RefCell<Vec<i32>>>:**

```
Stack:                  Heap (Rc allocation):           Heap (Vec buffer):
a: ┌──────┐           ┌──────────────────────────┐    ┌───┬───┬───┬───┬───┐
   │ ptr ─┼───┐       │ strong_count: 2           │    │ 1 │ 2 │ 3 │ 4 │ 5 │
   └──────┘   │       │ weak_count: 0              │    └───┴───┴───┴───┴───┘
b: ┌──────┐   ├──────>│ RefCell {                  │         ^
   │ ptr ─┼───┘       │   borrow_flag: 0           │         │
   └──────┘           │   data: Vec {              │         │
                      │     ptr ───────────────────┼─────────┘
                      │     len: 5                  │
                      │     cap: 8                  │
                      │   }                         │
                      │ }                           │
                      └──────────────────────────────┘
```

---

## Cow<T> -- Clone on Write

`Cow<T>` (Clone on Write) can hold either a borrowed reference or an owned value. It clones only when mutation is needed.

```rust
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains("bad") {
        // Need to modify -- clone into owned String
        Cow::Owned(input.replace("bad", "good"))
    } else {
        // No modification needed -- just borrow
        Cow::Borrowed(input)
    }
}

let clean = process("hello world");    // Cow::Borrowed -- no allocation
let fixed = process("bad input");      // Cow::Owned -- allocated new String
```

**When to use:** Functions that sometimes need to modify data and sometimes don't. Avoids unnecessary cloning.

```rust
// Definition:
enum Cow<'a, B: ToOwned + ?Sized> {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
// For str: Cow<'a, str> is either &'a str or String
// For [T]: Cow<'a, [T]> is either &'a [T] or Vec<T>
```

---

## Weak<T> -- Break Reference Cycles

`Weak<T>` is a non-owning reference to `Rc`/`Arc` data. It doesn't prevent the data from being dropped. Use it to break reference cycles.

```rust
use std::rc::{Rc, Weak};

let strong = Rc::new(42);
let weak: Weak<i32> = Rc::downgrade(&strong);

// Access via upgrade (returns Option -- data may be gone)
if let Some(val) = weak.upgrade() {
    println!("Still alive: {}", val);
}

drop(strong);  // data is freed (only weak references remain)

assert!(weak.upgrade().is_none());  // data is gone
```

### Breaking a Reference Cycle

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // weak reference UP (prevents cycle)
    children: RefCell<Vec<Rc<Node>>>,  // strong references DOWN
}
```

```
Without Weak (CYCLE -- memory leak):
Parent ──Rc──> Child ──Rc──> Parent   (reference count never reaches 0)

With Weak (NO cycle):
Parent ──Rc──> Child ──Weak──> Parent (Parent can be dropped, Weak becomes None)
```

---

## Deref and Drop Traits

### Deref -- Auto-Dereference

`Deref` lets a smart pointer behave like a reference. The compiler auto-inserts `*` operations.

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}

let x = MyBox(String::from("hello"));
// These are equivalent:
let s: &str = &x;        // auto-deref: &MyBox<String> -> &String -> &str
let s: &str = &(*x);     // explicit
```

**Deref coercion chain:** `&Box<String>` -> `&String` -> `&str`. The compiler follows the `Deref` chain automatically.

### Drop -- Custom Cleanup

`Drop` runs when a value goes out of scope. Used for releasing resources.

```rust
struct FileHandle {
    name: String,
}

impl Drop for FileHandle {
    fn drop(&mut self) {
        println!("Closing file: {}", self.name);
    }
}

{
    let f = FileHandle { name: "data.txt".into() };
    // use f...
}   // "Closing file: data.txt" -- drop() called automatically

// Early drop
let f = FileHandle { name: "data.txt".into() };
drop(f);  // explicitly drop early
// f is no longer usable
```

**Drop order:** Variables are dropped in reverse declaration order. Struct fields are dropped in declaration order.

---

## When to Use Which Smart Pointer

| Smart Pointer | Use Case | Thread-Safe | Overhead |
|---|---|---|---|
| `Box<T>` | Single owner, heap allocation | Yes (if T: Send) | 0 (just a pointer) |
| `Rc<T>` | Multiple owners, single-threaded | **No** | Reference count increment/decrement |
| `Arc<T>` | Multiple owners, multi-threaded | Yes | Atomic ref count (costlier than Rc) |
| `RefCell<T>` | Interior mutability, single-threaded | **No** | Runtime borrow flag check |
| `Mutex<T>` | Interior mutability, multi-threaded | Yes | Lock acquisition |
| `Cow<T>` | Clone only when needed | Depends on T | None until clone |
| `Weak<T>` | Non-owning reference (break cycles) | Matches Rc/Arc | Upgrade check |

### Common Combinations

```
Box<T>           -- single owner, heap data
Rc<T>            -- multiple owners, immutable, single-thread
Rc<RefCell<T>>   -- multiple owners, mutable, single-thread
Arc<T>           -- multiple owners, immutable, multi-thread
Arc<Mutex<T>>    -- multiple owners, mutable, multi-thread
Arc<RwLock<T>>   -- multiple owners, reader-writer lock, multi-thread
```

---

## Practical Examples / LeetCode Patterns

### LRU Cache (LeetCode 146) -- Rc<RefCell<Node>> for Doubly-Linked List

```rust
use std::collections::HashMap;
use std::rc::Rc;
use std::cell::RefCell;

type Link = Option<Rc<RefCell<Node>>>;

struct Node {
    key: i32,
    val: i32,
    prev: Link,
    next: Link,
}

struct LRUCache {
    capacity: usize,
    map: HashMap<i32, Rc<RefCell<Node>>>,
    head: Link,  // sentinel
    tail: Link,  // sentinel
}
```

**Smart pointer insight:** Each node is `Rc<RefCell<Node>>` because: multiple pointers reference the same node (HashMap + prev/next pointers), and we need to mutate nodes (RefCell). In single-threaded LeetCode, Rc is sufficient -- no need for Arc.

### Tree with Parent Pointers -- Weak for Back-References

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct TreeNode {
    value: i32,
    parent: RefCell<Weak<TreeNode>>,
    children: RefCell<Vec<Rc<TreeNode>>>,
}

impl TreeNode {
    fn new(value: i32) -> Rc<TreeNode> {
        Rc::new(TreeNode {
            value,
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(vec![]),
        })
    }

    fn add_child(parent: &Rc<TreeNode>, child: &Rc<TreeNode>) {
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        parent.children.borrow_mut().push(Rc::clone(child));
    }
}
```

**Smart pointer insight:** Children use `Rc` (strong reference -- children keep parent alive). Parent pointer uses `Weak` (non-owning -- doesn't create a cycle). When all strong references are dropped, the node is freed, and all Weak pointers become `None`.

### Graph Representation -- Arc<Mutex<T>> for Thread-Safe Shared State

```rust
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

type Graph = Arc<Mutex<HashMap<i32, Vec<i32>>>>;

fn add_edge(graph: &Graph, from: i32, to: i32) {
    let mut g = graph.lock().unwrap();
    g.entry(from).or_default().push(to);
    g.entry(to).or_default().push(from);
}
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| `Box<T>` | Single owner, heap allocation. 8 bytes on stack. Zero overhead |
| `Rc<T>` | Multiple owners, non-atomic ref count. NOT thread-safe |
| `Arc<T>` | Multiple owners, atomic ref count. Thread-safe. 10-30x costlier clone than Rc |
| `RefCell<T>` | Runtime borrow checking. Panics on violation. NOT thread-safe |
| `Cow<T>` | Borrow or own. Clones only when mutation needed |
| `Weak<T>` | Non-owning. Prevents reference cycles. `upgrade()` returns `Option` |
| `Deref` | Auto-dereference chain. `Box<String>` acts like `&String` acts like `&str` |
| `Drop` | Cleanup on scope exit. Variables dropped in reverse order |
| Combination | `Rc<RefCell<T>>` single-thread mutable. `Arc<Mutex<T>>` multi-thread mutable |
