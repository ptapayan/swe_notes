# Lifetimes in Rust

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics](traits_and_generics.md) | [structs and enums (structs holding references)](structs_and_enums.md) | [smart pointers](smart_pointers.md)

## What Lifetimes Actually Are

**Lifetimes are NOT about how long a value lives.** They are about how long a **reference** is valid. A lifetime is the compiler's way of ensuring that references never outlive the data they point to.

```rust
{
    let r;                  // r's lifetime starts here
    {
        let x = 5;         // x's lifetime starts here
        r = &x;            // r borrows x
    }                       // x's lifetime ENDS here -- x is dropped
    // println!("{}", r);   // COMPILE ERROR: r references dropped value
}                           // r's lifetime ends here
```

**How to think about it:** Every reference has a lifetime. The compiler tracks these lifetimes and ensures that no reference outlives its referent. Most of the time, the compiler infers lifetimes automatically. You only need annotations when the compiler can't figure it out.

---

## Lifetime Annotations

Lifetime annotations don't change how long values live -- they **describe relationships** between lifetimes of references.

```rust
// Without annotation -- compiler can't determine which input
// the output reference is tied to
fn longest(x: &str, y: &str) -> &str {  // COMPILE ERROR
    if x.len() > y.len() { x } else { y }
}

// With lifetime annotation -- tells the compiler:
// "the returned reference lives as long as the SHORTER of x and y"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**What `'a` means here:**
- `x: &'a str` -- x is a reference valid for at least lifetime `'a`
- `y: &'a str` -- y is a reference valid for at least lifetime `'a`
- `-> &'a str` -- the returned reference is valid for at least lifetime `'a`

The compiler infers `'a` to be the **intersection** (shorter) of the actual lifetimes of x and y.

```rust
fn main() {
    let string1 = String::from("long string");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
        println!("{}", result);  // OK: both string1 and string2 are alive
    }
    // println!("{}", result);  // ERROR: string2 is dropped, result may reference it
}
```

**Visual lifetime analysis:**

```
string1: |========================|  (lives until end of main)
string2:     |==========|            (lives until inner block ends)
result:      |==========|            (constrained to shorter lifetime)
                         ^
                    string2 dropped, result invalid
```

---

## Why Lifetimes Are Needed

Consider what happens WITHOUT lifetime checking:

```rust
// If Rust allowed this (it doesn't):
fn dangerous() -> &String {
    let s = String::from("hello");
    &s   // returning reference to local variable
}        // s is dropped here -- reference would be DANGLING

// In C/C++, this compiles and produces a use-after-free bug
// In Rust, it's a compile error
```

Lifetimes prevent:
1. **Dangling references** (reference to freed memory)
2. **Use-after-free** (accessing freed data through a reference)
3. **Iterator invalidation** (modifying a collection while iterating)

---

## Lifetime Elision Rules

The compiler applies three rules to infer lifetimes automatically. If the rules produce a unique, unambiguous lifetime for all output references, no annotation is needed.

```
Rule 1: Each input reference gets its own lifetime parameter
        fn foo(x: &str, y: &str)  -->  fn foo<'a, 'b>(x: &'a str, y: &'b str)

Rule 2: If there's exactly ONE input lifetime, it's assigned to all outputs
        fn foo(x: &str) -> &str   -->  fn foo<'a>(x: &'a str) -> &'a str

Rule 3: If one input is &self or &mut self, its lifetime is assigned to all outputs
        fn foo(&self, x: &str) -> &str  -->  fn foo<'a>(&'a self, x: &str) -> &'a str
```

### Examples Where Elision Works

```rust
// Rule 2: one input reference, applied to output
fn first_word(s: &str) -> &str {
    // Compiler infers: fn first_word<'a>(s: &'a str) -> &'a str
    &s[..s.find(' ').unwrap_or(s.len())]
}

// Rule 3: &self lifetime applied to output
impl MyStruct {
    fn name(&self) -> &str {
        // Compiler infers: fn name<'a>(&'a self) -> &'a str
        &self.name
    }
}
```

### Examples Where Elision Fails (Annotation Required)

```rust
// Two input references -- compiler can't determine which one the output ties to
fn longest(x: &str, y: &str) -> &str { ... }
// Error: need explicit lifetime annotation

// Fix:
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }
```

---

## 'static Lifetime

`'static` means the reference is valid for the **entire program duration**.

```rust
// String literals are 'static -- they're baked into the binary
let s: &'static str = "hello, world";

// Owned types satisfy 'static bounds (they have no references that could dangle)
fn spawn_thread<T: Send + 'static>(val: T) {
    std::thread::spawn(move || {
        println!("got value");
    });
}
// T: 'static means T owns all its data (no borrowed references)
// String is 'static, but &String is not
```

**Common confusion:** `'static` does NOT mean "lives forever." It means "CAN live as long as needed." An owned `String` satisfies `'static` because it has no references -- but it can still be dropped.

```rust
// This is fine -- String satisfies 'static even though it gets dropped
fn example() {
    let s = String::from("hello");  // satisfies 'static (owns its data)
    drop(s);                         // dropped here, that's fine
}
```

**`T: 'static` in trait bounds:** Usually means "T contains no non-static references." This is required for `thread::spawn` because the thread might outlive the caller.

---

## Lifetimes in Structs

If a struct holds references, it needs lifetime annotations -- the struct cannot outlive the data it references.

```rust
struct Excerpt<'a> {
    text: &'a str,   // borrows a string slice
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence;
    {
        let excerpt = Excerpt {
            text: &novel[..16],  // borrows from novel
        };
        first_sentence = excerpt.text;
    }
    println!("{}", first_sentence);  // OK: novel is still alive
}
```

**Memory layout:**

```
Stack:                      Heap:
novel: ┌──────────┐       ┌──────────────────────────────────┐
       │ ptr ─────┼──────>│ C a l l   m e   I s h m a e l . │
       │ len: 34  │       └──────────────────────────────────┘
       │ cap: 34  │                ^
       └──────────┘                │
excerpt: ┌──────────┐             │
         │ text.ptr ┼─────────────┘  (points into novel's heap data)
         │ text.len │  16
         └──────────┘

Constraint: excerpt cannot outlive novel (the struct holds a reference)
```

### Struct Methods with Lifetimes

```rust
impl<'a> Excerpt<'a> {
    fn level(&self) -> i32 {
        3   // no reference returned -- no lifetime issues
    }

    fn announce_and_return(&self, announcement: &str) -> &str {
        // Rule 3: &self lifetime applied to output
        // Compiler infers the return references self's data
        println!("Attention: {}", announcement);
        self.text
    }
}
```

---

## Lifetime Bounds on Generics

```rust
// T must outlive 'a (T contains no references shorter than 'a)
fn longest_with_context<'a, T>(x: &'a str, y: &'a str, ctx: T) -> &'a str
where
    T: Display + 'a,
{
    println!("Context: {}", ctx);
    if x.len() > y.len() { x } else { y }
}
```

### Multiple Lifetimes

```rust
// Different lifetimes for different references
fn first_or_default<'a, 'b>(first: &'a str, default: &'b str) -> &'a str {
    if !first.is_empty() {
        first       // returns with lifetime 'a
    } else {
        default     // ERROR: can't return 'b as 'a unless 'b: 'a
    }
}

// Fix: constrain 'b to live at least as long as 'a
fn first_or_default<'a, 'b: 'a>(first: &'a str, default: &'b str) -> &'a str {
    if !first.is_empty() { first } else { default }
}
```

`'b: 'a` reads as "'b outlives 'a" -- the data behind `default` lives at least as long as the data behind `first`.

---

## Common Lifetime Errors and Fixes

### Error: Missing Lifetime Specifier

```rust
// BROKEN:
struct Config {
    name: &str,   // ERROR: missing lifetime specifier
}

// FIX 1: Add lifetime (struct borrows data)
struct Config<'a> {
    name: &'a str,
}

// FIX 2: Own the data (usually simpler)
struct Config {
    name: String,
}
```

**Rule of thumb:** If the struct stores data long-term, own it (`String`). If the struct is a temporary view, borrow it (`&'a str`).

### Error: Lifetime May Not Live Long Enough

```rust
// BROKEN:
fn parse_first_word(input: &str) -> Excerpt {
    let word = input.split_whitespace().next().unwrap();
    Excerpt { text: word }
    // ERROR: compiler can't infer lifetime relationship
}

// FIX: make the relationship explicit
fn parse_first_word<'a>(input: &'a str) -> Excerpt<'a> {
    let word = input.split_whitespace().next().unwrap();
    Excerpt { text: word }
}
```

### Error: Cannot Return Reference to Local Variable

```rust
// BROKEN:
fn create_and_ref() -> &str {
    let s = String::from("hello");
    &s   // ERROR: s is dropped at end of function
}

// FIX: return owned data
fn create_and_own() -> String {
    String::from("hello")
}
```

---

## Relationship to Borrowing

Lifetimes and borrowing are two sides of the same coin:

```
Borrowing rules:
  - One mutable reference OR any number of immutable references
  - References must be valid

Lifetimes:
  - The mechanism that PROVES references are valid
  - Track the region of code where a reference is valid
  - Prevent dangling references
```

**NLL (Non-Lexical Lifetimes)** made this more precise: a reference's lifetime ends at its last use, not at the end of its scope.

```rust
let mut data = vec![1, 2, 3];
let first = &data[0];       // immutable borrow starts
println!("{}", first);       // immutable borrow ENDS (last use, via NLL)
data.push(4);                // mutable borrow OK -- no conflict
```

---

## Practical Examples / LeetCode Patterns

### String Processing with Lifetimes -- Split and Reference

```rust
fn longest_word<'a>(sentence: &'a str) -> &'a str {
    sentence.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

let text = String::from("the quick brown fox");
let word = longest_word(&text);
println!("Longest: {}", word);  // "quick"
// word borrows from text -- text must outlive word
```

### Iterator Returning References (LeetCode Pattern: Sliding Window)

```rust
fn windows_sum(nums: &[i32], k: usize) -> Vec<i32> {
    // .windows(k) returns an iterator of &[i32] slices
    // Each slice borrows from the original nums
    nums.windows(k)
        .map(|window| window.iter().sum())
        .collect()
}

let nums = vec![1, 2, 3, 4, 5];
let sums = windows_sum(&nums, 3);  // [6, 9, 12]
```

**Lifetime insight:** `windows()` returns `&'a [i32]` slices that borrow from `nums`. The iterator can't outlive `nums`. The compiler verifies this without any explicit annotations (elision rule 2).

### Struct with Borrowed Data -- Cache Pattern

```rust
struct Parser<'input> {
    input: &'input str,
    position: usize,
}

impl<'input> Parser<'input> {
    fn new(input: &'input str) -> Self {
        Parser { input, position: 0 }
    }

    fn next_token(&mut self) -> Option<&'input str> {
        // Returns a slice of the original input
        // The returned reference has lifetime 'input, not 'self
        let remaining = &self.input[self.position..];
        let end = remaining.find(' ').unwrap_or(remaining.len());
        if end == 0 { return None; }
        let token = &self.input[self.position..self.position + end];
        self.position += end + 1;
        Some(token)
    }
}
```

**Lifetime insight:** The parser borrows the input string and returns references into it. The `'input` lifetime ensures the input outlives both the parser and any tokens it returns. The returned `&'input str` ties to the input, not to `&self` -- so tokens remain valid even after the parser is dropped.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| What lifetimes are | How long a REFERENCE is valid, not how long a VALUE lives |
| Annotations `'a` | Describe relationships between lifetimes. Don't change behavior |
| Elision rules | 3 rules that let the compiler infer lifetimes. Manual annotation only when ambiguous |
| `'static` | Reference valid for entire program. Owned types satisfy `'static` bounds |
| Structs with `&` | Must have lifetime parameters. Struct can't outlive the borrowed data |
| `'b: 'a` | "'b outlives 'a" -- data behind 'b lives at least as long as data behind 'a |
| NLL | Lifetimes end at last use, not end of scope. Much more ergonomic |
| Rule of thumb | Own data (String) for long-lived structs. Borrow (&str) for temporary views |
