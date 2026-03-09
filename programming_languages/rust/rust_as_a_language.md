# Rust as a Language

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics](traits_and_generics.md) | [error handling](error_handling_in_rust.md) | [lifetimes](lifetimes_in_rust.md) | [best practices](rust_best_practices.md)

## Overview

Rust was created by Graydon Hoare at Mozilla in 2010, with version 1.0 released in 2015. It was designed to be a systems programming language that guarantees **memory safety without a garbage collector** through its ownership system. Rust consistently ranks as the most loved language in Stack Overflow surveys. It is used at AWS (Firecracker), Microsoft (parts of Windows), Google (Android, Chromium), Cloudflare, Discord, and Dropbox.

---

## Compiler and Compilation

### Compiler: rustc with LLVM Backend

Rust uses **rustc**, an ahead-of-time (AOT) compiler that emits LLVM IR, which LLVM then optimizes and compiles to native machine code.

```bash
rustc main.rs          # produces a native binary
./main                 # runs directly on the CPU -- no VM, no runtime, no GC
```

**Why LLVM?** Rust gets world-class optimizations for free: inlining, loop unrolling, autovectorization, dead code elimination, constant propagation. The same backend that powers Clang (C/C++).

### Compilation Pipeline

```
Source (.rs) --> Lexer/Parser --> AST --> HIR --> MIR --> LLVM IR --> Machine Code
```

**Key stages:**
1. **AST (Abstract Syntax Tree):** Raw parse tree from source code. Macros are expanded here
2. **HIR (High-Level IR):** Desugared AST. Type checking and trait resolution happen here. `for` loops become `loop` + `Iterator::next()`, `?` becomes `match` on `Result`
3. **MIR (Mid-Level IR):** Control flow graph. **Borrow checking happens here.** Also: lifetime inference, optimization passes (constant propagation, dead code removal), and monomorphization of generics
4. **LLVM IR:** Handed to LLVM for backend optimization and code generation. Rust-specific concerns are done -- LLVM treats it like any other language
5. **Machine code:** Platform-specific binary via LLVM codegen

**How to think about it:** HIR is about "what does this mean?" (types, traits). MIR is about "is this safe?" (borrows, lifetimes). LLVM IR is about "make this fast" (optimizations, codegen).

### Why Rust Compiles Slowly (Compared to Go)

- Monomorphization: every generic instantiation creates a new function (code bloat)
- LLVM optimization passes are expensive (same as C++ with `-O2`)
- Borrow checker analysis is non-trivial
- Procedural macros run arbitrary code at compile time
- Cargo builds the entire dependency tree from source (no precompiled headers)

**Mitigation:** Incremental compilation, `cargo check` (type-checks without codegen), `sccache`, cranelift backend (faster debug builds).

---

## Type System

### Statically and Strongly Typed

Every expression has a type known at compile time. No implicit conversions.

```rust
let x: i32 = 42;        // explicit type
let y = "hello";         // inferred as &str at compile time
// y = 42;               // COMPILE ERROR: mismatched types

let a: i32 = 10;
let b: i64 = a as i64;  // explicit cast required
// let b: i64 = a;       // COMPILE ERROR: expected i64, found i32
```

### Algebraic Type System

Rust has true algebraic data types -- **sum types** (enums) and **product types** (structs). This is more expressive than Go or Java.

```rust
// Sum type: Option is "either Some(T) or None"
enum Option<T> {
    Some(T),
    None,
}

// Product type: struct is "all of these fields together"
struct Point {
    x: f64,
    y: f64,
}
```

**Why this matters:** The compiler can verify you handle ALL possible cases. No null pointer exceptions, no forgotten error checks. `match` forces exhaustive pattern matching.

**Related:** [structs and enums](structs_and_enums.md) for deep dive.

---

## Memory Management: No GC, Ownership Model

Rust has NO garbage collector and NO manual `malloc`/`free`. Instead, it uses the **ownership system** -- a set of rules checked at compile time.

```
Three ownership rules:
1. Each value has exactly one owner
2. When the owner goes out of scope, the value is dropped (freed)
3. There can be either one mutable reference OR any number of immutable references
```

```rust
fn main() {
    let s = String::from("hello");   // s owns the String, allocated on heap
    let t = s;                        // ownership MOVES to t. s is now invalid
    // println!("{}", s);             // COMPILE ERROR: value used after move
    println!("{}", t);                // works fine
}   // t goes out of scope, String is dropped (heap memory freed)
```

**What's happening in memory:**

```
Before move:             After move:
Stack:                   Stack:
s: [ptr|len:5|cap:5]    s: [INVALID -- moved]
     |                   t: [ptr|len:5|cap:5]
     v                        |
Heap: "hello"                 v
                         Heap: "hello"   (same allocation, NOT copied)
```

The heap data is NOT copied. Only the stack metadata (pointer + length + capacity = 24 bytes) is moved. The compiler inserts a `drop()` call at the end of `t`'s scope -- no GC needed.

**Related:** [ownership and borrowing](ownership_and_borrowing.md) for full details.

---

## Zero-Cost Abstractions

Rust's core promise: **abstractions compile down to the same code you'd write by hand.** No runtime overhead.

```rust
// Iterator chain -- looks high-level, compiles to a tight loop
let sum: i32 = (0..1000)
    .filter(|x| x % 2 == 0)
    .map(|x| x * x)
    .sum();

// Generics -- monomorphized at compile time, no vtable
fn max<T: Ord>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}
// Compiler generates max_i32, max_i64, max_String, etc. -- each fully optimized
```

**How to think about it:** In C++, `std::vector` is zero-cost because you couldn't write a faster dynamic array by hand. In Rust, the SAME principle applies to iterators, futures, trait impls, generics, and more. The compiler does the work so you don't pay at runtime.

---

## Cargo Ecosystem

Cargo is Rust's build system AND package manager (unlike Go which has separate `go build` and `go get`).

```bash
cargo new my_project     # create new project with Cargo.toml
cargo build              # compile (debug mode)
cargo build --release    # compile with optimizations (LLVM -O3)
cargo run                # compile and run
cargo test               # run tests
cargo check              # type-check without codegen (fast)
cargo clippy             # linter (catches common mistakes)
cargo fmt                # format code (like gofmt)
cargo doc --open         # generate and view documentation
cargo bench              # run benchmarks
```

**Cargo.toml** (like Go's `go.mod`):
```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

**Crates.io** is the package registry. Cargo uses **SemVer** for dependency resolution. `Cargo.lock` pins exact versions (like `go.sum`).

---

## Comparison with C/C++/Go

| Aspect | Rust | C/C++ | Go |
|---|---|---|---|
| Memory safety | Compile-time (ownership) | Manual / RAII | Runtime (GC) |
| Null | No null (uses `Option<T>`) | Null pointers everywhere | `nil` for pointers/slices/maps |
| Error handling | `Result<T,E>` + `?` operator | Exceptions (C++) / error codes (C) | `(val, error)` tuples |
| Concurrency safety | Compile-time (`Send`/`Sync`) | Manual (hope for the best) | Runtime (race detector) |
| Generics | Monomorphized (zero-cost) | Templates (complex) | GC shape stenciling + dictionaries |
| Compile speed | Slow (LLVM, monomorphization) | Slow (C++), moderate (C) | Very fast |
| Binary size | Moderate (no runtime) | Small (no runtime) | Large (includes GC + runtime) |
| Learning curve | Steep (borrow checker) | Steep (undefined behavior) | Gentle |

---

## Edition System

Rust editions (2015, 2018, 2021, 2024) allow breaking syntax changes without breaking the ecosystem. Different crates can use different editions and interoperate seamlessly.

```toml
# Cargo.toml
[package]
edition = "2021"   # which edition of Rust syntax to use
```

**Key changes by edition:**
- **2018:** `async`/`await`, module system overhaul, NLL (non-lexical lifetimes)
- **2021:** Disjoint capture in closures, `IntoIterator` for arrays
- **2024:** `gen` blocks, unsafe in `extern` blocks

**How to think about it:** Editions change the frontend (parsing, desugaring) but not the backend. All editions compile to the same MIR/LLVM IR. A 2015 crate can depend on a 2021 crate and vice versa.

---

## Unsafe Rust

Rust has an `unsafe` escape hatch for when the compiler cannot verify safety. Inside `unsafe` blocks, you can:

- Dereference raw pointers
- Call `unsafe` functions (FFI, intrinsics)
- Access mutable statics
- Implement `unsafe` traits (`Send`, `Sync`)

```rust
let x: i32 = 42;
let ptr: *const i32 = &x;

unsafe {
    println!("value: {}", *ptr);  // dereferencing raw pointer
}
```

**The contract:** `unsafe` does NOT turn off the borrow checker. It only unlocks 5 specific capabilities. You are responsible for upholding Rust's safety invariants within `unsafe` blocks. The rest of the program can still rely on those guarantees.

**Related:** [best practices](rust_best_practices.md) for when to use unsafe.

---

## Key Takeaways

| Aspect | Rust's Approach |
|---|---|
| Compiler | rustc + LLVM backend. Pipeline: AST -> HIR -> MIR -> LLVM IR -> machine code |
| Type system | Static, strong, algebraic (sum + product types). No implicit conversions |
| Memory | No GC. Ownership system enforced at compile time. Deterministic drops |
| Zero-cost | Generics monomorphized, iterators fused, traits inlined. No runtime overhead |
| Safety | Memory safety + thread safety guaranteed at compile time (unless `unsafe`) |
| Ecosystem | Cargo (build + package manager), crates.io, editions for backward compatibility |
| Trade-off | Steep learning curve and slow compilation in exchange for runtime performance and safety |
