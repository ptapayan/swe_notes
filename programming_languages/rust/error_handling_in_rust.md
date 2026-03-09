# Error Handling in Rust

**Related:** [structs and enums (Result, Option)](structs_and_enums.md) | [traits and generics (From, Error trait)](traits_and_generics.md) | [ownership and borrowing](ownership_and_borrowing.md) | [best practices](rust_best_practices.md)

## Philosophy: No Exceptions, Errors Are Values

Rust has NO exceptions. Errors are represented as values using two core enums:

```rust
// For operations that might fail
enum Result<T, E> {
    Ok(T),    // success with value T
    Err(E),   // failure with error E
}

// For values that might be absent
enum Option<T> {
    Some(T),  // value present
    None,     // value absent
}
```

**How to think about it:** `Result` replaces try/catch. `Option` replaces null. The compiler FORCES you to handle both cases -- no unchecked exceptions, no null pointer dereferences.

---

## Result<T, E>

### Basic Usage

```rust
use std::fs;
use std::io;

fn read_config(path: &str) -> Result<String, io::Error> {
    let contents = fs::read_to_string(path)?;  // ? propagates error
    Ok(contents)
}

fn main() {
    match read_config("config.toml") {
        Ok(contents) => println!("Config: {}", contents),
        Err(e) => eprintln!("Failed to read config: {}", e),
    }
}
```

### Extracting Values from Result

```rust
let result: Result<i32, String> = Ok(42);

// match -- most explicit
match result {
    Ok(val) => println!("Got: {}", val),
    Err(e) => println!("Error: {}", e),
}

// unwrap -- panics on Err (use ONLY when you're certain it's Ok)
let val = result.unwrap();         // panics with generic message if Err

// expect -- panics with custom message (better than unwrap)
let val = result.expect("config must exist");  // panics: "config must exist: ..."

// unwrap_or -- provide default
let val = result.unwrap_or(0);     // returns 0 if Err

// unwrap_or_else -- provide default via closure
let val = result.unwrap_or_else(|e| {
    eprintln!("Error: {}", e);
    0
});

// map -- transform the Ok value
let doubled = result.map(|v| v * 2);  // Ok(84)

// and_then -- chain fallible operations
let parsed = result.and_then(|v| {
    if v > 0 { Ok(v) } else { Err("must be positive".to_string()) }
});
```

---

## The ? Operator (Early Return Sugar)

The `?` operator is syntactic sugar for propagating errors. It unwraps `Ok` or returns `Err` early.

```rust
fn process() -> Result<String, io::Error> {
    let contents = fs::read_to_string("input.txt")?;
    let trimmed = contents.trim().to_string();
    Ok(trimmed)
}

// The ? operator desugars to:
fn process_desugared() -> Result<String, io::Error> {
    let contents = match fs::read_to_string("input.txt") {
        Ok(val) => val,
        Err(e) => return Err(e.into()),  // note: .into() for type conversion
    };
    let trimmed = contents.trim().to_string();
    Ok(trimmed)
}
```

**Critical:** `?` calls `.into()` on the error, which means it uses the `From` trait to convert between error types. This enables automatic error type conversion.

### ? with Option<T>

```rust
fn get_first_char(s: &str) -> Option<char> {
    let first_line = s.lines().next()?;   // returns None if no lines
    first_line.chars().next()              // returns None if empty line
}
```

`?` on `Option` returns `None` early if the value is `None`.

---

## unwrap and expect: When to Use

```rust
// unwrap -- NEVER in production code (except...)
let val = some_result.unwrap();  // panics on Err

// expect -- slightly better (gives context in panic message)
let val = some_result.expect("database connection must succeed");
```

**When unwrap/expect are acceptable:**
1. **Tests** -- a panic is a test failure, which is what you want
2. **Prototyping** -- quick and dirty, will be replaced later
3. **Provably safe** -- when the logic guarantees success

```rust
// Provably safe: we just checked the condition
let s = "123";
if s.chars().all(|c| c.is_ascii_digit()) {
    let n: i32 = s.parse().unwrap();  // can't fail -- all chars are digits
}

// After validation that guarantees Ok
let config = load_config().expect("config file verified to exist at startup");
```

**Rule of thumb:** If you see `.unwrap()` in a code review, it should either be in a test or have a comment explaining why it can't fail.

---

## Custom Error Types

### Manual Implementation

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    PermissionDenied,
    DatabaseError(String),
    ParseError(std::num::ParseIntError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::NotFound(resource) => write!(f, "{} not found", resource),
            AppError::PermissionDenied => write!(f, "permission denied"),
            AppError::DatabaseError(msg) => write!(f, "database error: {}", msg),
            AppError::ParseError(e) => write!(f, "parse error: {}", e),
        }
    }
}

impl std::error::Error for AppError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            AppError::ParseError(e) => Some(e),
            _ => None,
        }
    }
}

// Enable ? to convert ParseIntError into AppError automatically
impl From<std::num::ParseIntError> for AppError {
    fn from(e: std::num::ParseIntError) -> Self {
        AppError::ParseError(e)
    }
}
```

### thiserror Crate (For Libraries)

`thiserror` generates all the boilerplate above with derive macros:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error("{0} not found")]
    NotFound(String),

    #[error("permission denied")]
    PermissionDenied,

    #[error("database error: {0}")]
    DatabaseError(String),

    #[error("parse error")]
    ParseError(#[from] std::num::ParseIntError),  // auto-generates From impl
}
```

**What `#[from]` does:** Generates `impl From<ParseIntError> for AppError`, so `?` automatically converts `ParseIntError` into `AppError`.

### anyhow Crate (For Applications)

`anyhow` provides a catch-all error type for application code where you don't need typed error variants:

```rust
use anyhow::{Context, Result};

fn read_config(path: &str) -> Result<Config> {
    let contents = fs::read_to_string(path)
        .context(format!("failed to read config from {}", path))?;

    let config: Config = toml::from_str(&contents)
        .context("failed to parse config TOML")?;

    Ok(config)
}
```

**When to use which:**
- **thiserror** -- for libraries. Callers need to match on specific error variants
- **anyhow** -- for applications (binaries). You just need to report the error with context

---

## From Trait for Error Conversion

The `From` trait is how `?` knows how to convert between error types.

```rust
fn parse_and_double(s: &str) -> Result<i64, AppError> {
    let n: i64 = s.parse()?;   // ParseIntError -> AppError via From
    Ok(n * 2)
}

// The ? operator does: s.parse().map_err(AppError::from)?
```

**Error conversion chain:**

```
Function returns Result<T, AppError>
                                    ↑
Inner call returns Result<T, ParseIntError>
                                    ↑
? calls .into() which calls From<ParseIntError> for AppError
```

---

## The Error Trait

```rust
pub trait Error: Debug + Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

**The error chain:** `source()` returns the underlying cause. You can walk the chain:

```rust
fn print_error_chain(err: &dyn std::error::Error) {
    eprintln!("Error: {}", err);
    let mut source = err.source();
    while let Some(cause) = source {
        eprintln!("  Caused by: {}", cause);
        source = cause.source();
    }
}
```

---

## panic! vs Result (Unrecoverable vs Recoverable)

```
Result<T, E>  --> recoverable error. Caller decides what to do.
panic!        --> unrecoverable error. Program aborts (or unwinds stack).
```

### When to Use panic!

```rust
// 1. Programming bugs (invariant violations)
fn divide(a: f64, b: f64) -> f64 {
    assert!(b != 0.0, "division by zero is a bug, not a runtime error");
    a / b
}

// 2. Unrecoverable state
fn main() {
    let config = load_config().expect("cannot start without config");
}

// 3. Tests
#[test]
fn test_parse() {
    let result = parse("abc");
    assert!(result.is_err());
}
```

### When to Use Result

```rust
// 1. Expected failures (file not found, network error, invalid input)
fn read_file(path: &str) -> Result<String, io::Error> { ... }

// 2. Any library function -- NEVER panic in libraries
pub fn parse_config(input: &str) -> Result<Config, ConfigError> { ... }

// 3. When the caller should decide how to handle failure
fn connect(addr: &str) -> Result<Connection, ConnectionError> { ... }
```

---

## Error Handling Patterns

### Collecting Results

```rust
// Fail on first error
let results: Result<Vec<i32>, _> = vec!["1", "2", "abc", "4"]
    .into_iter()
    .map(|s| s.parse::<i32>())
    .collect();  // Err(ParseIntError) -- stops at "abc"

// Partition successes and failures
let (oks, errs): (Vec<_>, Vec<_>) = vec!["1", "2", "abc", "4"]
    .into_iter()
    .map(|s| s.parse::<i32>())
    .partition(Result::is_ok);
```

### Combining Option and Result

```rust
fn find_user_age(name: &str) -> Result<u32, AppError> {
    let user = find_user(name).ok_or(AppError::NotFound(name.to_string()))?;
    Ok(user.age)
}
// .ok_or() converts Option<T> to Result<T, E>
// .ok_or_else() for lazy error construction
```

### Converting Between Option and Result

```rust
let opt: Option<i32> = Some(42);
let res: Result<i32, &str> = opt.ok_or("missing value");  // Option -> Result

let res: Result<i32, &str> = Ok(42);
let opt: Option<i32> = res.ok();  // Result -> Option (discards error)
```

---

## Practical Examples / LeetCode Patterns

### String to Integer (LeetCode 8) -- Chained Error Handling

```rust
fn my_atoi(s: &str) -> i32 {
    let s = s.trim();
    if s.is_empty() { return 0; }

    let (sign, digits) = match s.as_bytes()[0] {
        b'+' => (1i64, &s[1..]),
        b'-' => (-1i64, &s[1..]),
        _ => (1i64, s),
    };

    let mut result: i64 = 0;
    for &b in digits.as_bytes() {
        if !b.is_ascii_digit() { break; }
        result = result * 10 + (b - b'0') as i64;
        // Clamp to i32 range
        if sign * result > i32::MAX as i64 { return i32::MAX; }
        if sign * result < i32::MIN as i64 { return i32::MIN; }
    }
    (sign * result) as i32
}
```

**Error handling insight:** This problem doesn't use `Result` because invalid input returns 0 (not an error). But in production code, you'd return `Result<i32, ParseError>` and let the caller decide.

### File Processing Pipeline -- Idiomatic Error Propagation

```rust
use std::io;
use std::fs;

#[derive(Debug, thiserror::Error)]
enum ProcessError {
    #[error("IO error: {0}")]
    Io(#[from] io::Error),
    #[error("parse error on line {line}: {message}")]
    Parse { line: usize, message: String },
}

fn process_file(path: &str) -> Result<Vec<i32>, ProcessError> {
    let contents = fs::read_to_string(path)?;  // io::Error -> ProcessError

    contents.lines()
        .enumerate()
        .map(|(i, line)| {
            line.trim()
                .parse::<i32>()
                .map_err(|e| ProcessError::Parse {
                    line: i + 1,
                    message: e.to_string(),
                })
        })
        .collect()  // collects Results, fails on first error
}
```

**Pattern:** `?` auto-converts `io::Error` via `#[from]`. Manual `.map_err()` for custom error construction where `From` doesn't apply.

### Retry with Exponential Backoff -- Result in Loops

```rust
fn fetch_with_retry(url: &str, max_retries: u32) -> Result<String, String> {
    for attempt in 0..max_retries {
        match fetch(url) {
            Ok(body) => return Ok(body),
            Err(e) if attempt < max_retries - 1 => {
                let delay = 2u64.pow(attempt);
                std::thread::sleep(std::time::Duration::from_secs(delay));
                continue;
            }
            Err(e) => return Err(format!("failed after {} retries: {}", max_retries, e)),
        }
    }
    unreachable!()
}
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Result<T, E> | Replaces exceptions. Compiler forces you to handle errors |
| Option<T> | Replaces null. No null pointer dereferences possible |
| `?` operator | Propagates errors + calls `.into()` for type conversion |
| unwrap/expect | Only in tests, prototypes, or provably safe contexts |
| From trait | Enables automatic error conversion with `?` |
| thiserror | For libraries -- typed error variants with derive macros |
| anyhow | For applications -- catch-all error with context chaining |
| panic! | Unrecoverable errors only. NEVER panic in library code |
| Error trait | `Debug + Display + source()`. Walk the error chain |
