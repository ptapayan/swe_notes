# If/Else Statements in Go

**Related:** [functions (error handling)](function_in_golang.md) | [interfaces (error interface)](golang_interface.md) | [maps (comma-ok idiom)](maps_in_golang.md)

## Basic Syntax

```go
if x > 10 {
    fmt.Println("greater")
} else if x == 10 {
    fmt.Println("equal")
} else {
    fmt.Println("less")
}
```

**Important syntax rules:**
- Opening brace `{` MUST be on the same line as `if` (Go's formatter enforces this)
- No parentheses around the condition (unlike C/Java)
- Braces are always required, even for single-line bodies

---

## If with Short Statement (Init Statement)

Go allows a short statement before the condition. This is idiomatic and used everywhere.

```go
if err := doSomething(); err != nil {
    fmt.Println("error:", err)
    return
}
// err is NOT accessible here — scoped to the if/else block
```

**What's happening:** `err` is declared and assigned in the same line. Its scope is limited to the `if`/`else if`/`else` chain. This keeps the namespace clean.

```go
if val, ok := cache["key"]; ok {
    fmt.Println("cached:", val)
} else {
    fmt.Println("not in cache")
    // val and ok are accessible here too (entire if/else chain)
}
// val and ok are NOT accessible here
```

**Related:** [maps (comma-ok idiom)](maps_in_golang.md) | [functions (error handling)](function_in_golang.md)

---

## The Error Handling Pattern

This is the most common use of `if` in Go. You'll see it hundreds of times in any Go project:

```go
result, err := someFunction()
if err != nil {
    return fmt.Errorf("someFunction failed: %w", err)
}
// use result (happy path continues at same indentation)
```

**How to think about this:**
- Go has no try/catch. Errors are values returned from functions
- The convention is: handle the error immediately, then continue with the happy path
- This keeps the "golden path" (success case) at the leftmost indentation level
- Use `%w` to wrap errors for `errors.Is()` and `errors.As()` chains

**Related:** [interfaces (error interface)](golang_interface.md) | [functions (multiple return values)](function_in_golang.md)

```go
// BAD: nesting makes the happy path hard to follow
func process() error {
    data, err := fetchData()
    if err == nil {
        result, err := transform(data)
        if err == nil {
            return save(result)
        } else {
            return err
        }
    } else {
        return err
    }
}

// GOOD: early returns, flat structure
func process() error {
    data, err := fetchData()
    if err != nil {
        return fmt.Errorf("fetch: %w", err)
    }

    result, err := transform(data)
    if err != nil {
        return fmt.Errorf("transform: %w", err)
    }

    return save(result)
}
```

---

## Boolean Expressions

Go has strict boolean typing. No truthy/falsy values like JavaScript or Python.

```go
x := 5

// These work
if x > 0 { }
if x != 0 { }
if true { }

// These are COMPILE ERRORS
// if x { }        // int is not bool
// if "hello" { }  // string is not bool
// if nil { }      // nil is not bool
```

### Logical Operators

```go
if a > 0 && b > 0 {  // AND — short-circuit: if a<=0, b>0 is NOT evaluated
    fmt.Println("both positive")
}

if a > 0 || b > 0 {  // OR — short-circuit: if a>0, b>0 is NOT evaluated
    fmt.Println("at least one positive")
}

if !done {  // NOT
    fmt.Println("not done")
}
```

**Short-circuit evaluation** is important for safety:

```go
// Safe: if p is nil, p.Value is never accessed
if p != nil && p.Value > 0 {
    fmt.Println(p.Value)
}
```

---

## No Ternary Operator

Go intentionally has no ternary (`?:`). Use an if/else:

```go
// Not valid Go:
// result := x > 0 ? "positive" : "non-positive"

// Do this instead:
var result string
if x > 0 {
    result = "positive"
} else {
    result = "non-positive"
}
```

For simple cases, some Go developers use a helper:

```go
func ternary[T any](cond bool, a, b T) T {
    if cond {
        return a
    }
    return b
}

result := ternary(x > 0, "positive", "non-positive")
```

**Note:** Both `a` and `b` are always evaluated (unlike a real ternary). This is fine for simple values but not for expensive function calls.

---

## Comparing with Switch

When you have many conditions on the same variable, `switch` is cleaner:

```go
// Verbose if/else chain
if status == "active" {
    // ...
} else if status == "pending" {
    // ...
} else if status == "inactive" {
    // ...
}

// Cleaner switch
switch status {
case "active":
    // ...
case "pending":
    // ...
case "inactive":
    // ...
}
```

`switch` with no expression acts like a clean if/else chain:

```go
switch {
case score >= 90:
    grade = "A"
case score >= 80:
    grade = "B"
case score >= 70:
    grade = "C"
default:
    grade = "F"
}
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Init statement | `if err := fn(); err != nil { }` — scoped to the if/else block |
| Error pattern | Check error, return early. Keep happy path at left margin |
| No truthy/falsy | Conditions must be `bool`. No implicit conversions |
| Short-circuit | `&&` and `\|\|` stop evaluating as soon as result is known |
| No ternary | Use if/else or a generic helper function |
