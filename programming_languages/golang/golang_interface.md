# Interfaces in Go

**Related:** [Go as a language](golang_as_a_language.md) | [structs (methods, embedding)](structs_in_golang.md) | [functions (methods with receivers)](function_in_golang.md)

## What Is an Interface?

An interface defines a **set of method signatures**. Any type that implements all those methods **implicitly** satisfies the interface — no `implements` keyword needed.

```go
type Speaker interface {
    Speak() string
}
```

Any type with a `Speak() string` method is a `Speaker`. Period. The type doesn't need to know the interface exists.

---

## How Interfaces Work Under the Hood

An interface value is internally a **two-word struct** (16 bytes on 64-bit):

```
┌─────────────┐
│  type pointer │ ──→ type metadata (itab): method table, type info
├─────────────┤
│  data pointer │ ──→ actual concrete value (or pointer to it)
└─────────────┘
```

This is called an **iface** (for non-empty interfaces) or **eface** (for `interface{}`/`any`).

**iface struct (simplified):**
```go
type iface struct {
    tab  *itab          // points to interface table (type + method pointers)
    data unsafe.Pointer // points to the actual value
}
```

The `itab` contains:
- The concrete type's metadata
- A table of function pointers for the interface methods → enables **dynamic dispatch**

---

## Basic Interface Usage

```go
type Speaker interface {
    Speak() string
}

type Dog struct {
    Name string
}

func (d Dog) Speak() string {
    return d.Name + " says woof!"
}

type Cat struct {
    Name string
}

func (c Cat) Speak() string {
    return c.Name + " says meow!"
}

func greet(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    greet(Dog{Name: "Rex"})   // "Rex says woof!"
    greet(Cat{Name: "Milo"})  // "Milo says meow!"
}
```

**What's happening in memory when you call `greet(Dog{Name: "Rex"})`:**
1. A `Dog` struct is created (on stack or heap depending on escape analysis)
2. An interface value is constructed: `{itab: *Dog_Speaker_itab, data: *Dog_value}`
3. Inside `greet`, calling `s.Speak()` does a **dynamic dispatch**: looks up the `Speak` function pointer in the itab and calls it

---

## Implicit Satisfaction (Structural Typing)

This is Go's killer feature for interfaces. Compare with other languages:

```go
// Java/C#: explicit
// class Dog implements Speaker { ... }

// Go: implicit — just implement the methods
type Dog struct{}
func (d Dog) Speak() string { return "woof" }
// Dog now satisfies Speaker automatically
```

**Why this matters:**
- You can define interfaces AFTER the types exist
- You can define interfaces in a DIFFERENT package than the implementing type
- Enables powerful decoupling — the implementer doesn't depend on the interface package

---

## The Empty Interface: `interface{}` / `any`

Every type satisfies the empty interface because it has zero methods.

```go
func printAnything(val any) { // `any` is an alias for `interface{}`
    fmt.Println(val)
}

func main() {
    printAnything(42)
    printAnything("hello")
    printAnything([]int{1, 2, 3})
}
```

**Memory:** An empty interface (`eface`) is simpler than `iface`:
```go
type eface struct {
    _type *_type         // type metadata only (no method table)
    data  unsafe.Pointer // pointer to value
}
```

**Think about it:** Using `any` loses all compile-time type safety. Avoid it unless you genuinely need to handle arbitrary types (like `fmt.Println` does).

---

## Type Assertions

Extract the concrete type from an interface.

```go
func describe(s Speaker) {
    // Unsafe — panics if s is not a Dog
    dog := s.(Dog)
    fmt.Println("Dog name:", dog.Name)

    // Safe — returns ok=false instead of panicking
    cat, ok := s.(Cat)
    if ok {
        fmt.Println("Cat name:", cat.Name)
    } else {
        fmt.Println("not a cat")
    }
}
```

**What happens internally:**
- The runtime checks if the `itab.type` in the interface matches the asserted type
- If yes, returns the `data` pointer cast to the concrete type
- If no, panics (single return) or returns zero value + false (comma-ok form)

---

## Type Switches

Cleaner way to handle multiple possible concrete types:

```go
func identify(s Speaker) {
    switch v := s.(type) {
    case Dog:
        fmt.Printf("Dog: %s\n", v.Name) // v is typed as Dog here
    case Cat:
        fmt.Printf("Cat: %s\n", v.Name) // v is typed as Cat here
    default:
        fmt.Printf("Unknown speaker: %T\n", v)
    }
}
```

---

## Interface Composition (Embedding)

Build complex interfaces from simpler ones:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader // embedded
    Writer // embedded
}
// ReadWriter requires both Read() and Write()
```

This is how the standard library builds `io.ReadWriter`, `io.ReadWriteCloser`, etc.

```go
type Closer interface {
    Close() error
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

---

## Pointer Receivers and Interfaces

This is a common gotcha:

```go
type Mover interface {
    Move()
}

type Car struct {
    Position int
}

// Pointer receiver
func (c *Car) Move() {
    c.Position++
}

func main() {
    var m Mover

    car := Car{}
    m = &car  // OK: *Car implements Mover
    // m = car // COMPILE ERROR: Car does not implement Mover (Move has pointer receiver)

    m.Move()
}
```

**Why?** A value of type `Car` doesn't necessarily have a stable address (it could be in a temporary). You can always get a pointer from an addressable value, but not always the reverse. So Go enforces: if the method set requires a pointer receiver, you must pass a pointer.

**Rules:**
- Type `T` has the method set of all value-receiver methods
- Type `*T` has the method set of ALL methods (both value and pointer receivers)

---

## Common Standard Library Interfaces

### Stringer (like toString)

```go
type Stringer interface {
    String() string
}

type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}

func main() {
    p := Person{"Alice", 30}
    fmt.Println(p) // "Alice (age 30)" — fmt calls String() automatically
}
```

### error Interface

```go
type error interface {
    Error() string
}

// Custom error
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

func validate(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    return nil
}
```

### io.Reader and io.Writer

```go
// The most important interfaces in Go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Implemented by: files, network connections, buffers, HTTP bodies, compressors, encryptors, etc. This is what makes Go's I/O so composable.

---

## Nil Interface vs Nil Concrete Value (Critical Gotcha)

```go
type MyError struct{}

func (e *MyError) Error() string { return "error" }

func getError() error {
    var err *MyError = nil
    return err // returns interface{type: *MyError, data: nil} — NOT nil!
}

func main() {
    err := getError()
    fmt.Println(err == nil) // false!!! The interface is NOT nil
}
```

**Why?** An interface is nil ONLY when BOTH the type pointer AND data pointer are nil. Here, the type pointer is `*MyError` (not nil), even though the data is nil.

**Fix:**

```go
func getError() error {
    var err *MyError = nil
    if err == nil {
        return nil // return an untyped nil
    }
    return err
}
```

---

## Interface Design Principles

### Keep Interfaces Small

```go
// Good — small and focused
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Bad — too many methods, hard to implement
type DoEverything interface {
    Read() []byte
    Write([]byte)
    Close()
    Flush()
    Seek(int)
    // ...
}
```

> "The bigger the interface, the weaker the abstraction." — Rob Pike

### Accept Interfaces, Return Structs

```go
// Good: function accepts an interface (flexible)
func Process(r io.Reader) error { ... }

// Good: function returns a concrete type (specific)
func NewBuffer() *bytes.Buffer { ... }
```

### Define Interfaces Where They're Used, Not Where They're Implemented

```go
// In the consumer package (where you USE the behavior):
type Storage interface {
    Save(data []byte) error
}

// In the provider package (concrete type):
type S3Client struct{ ... }
func (s *S3Client) Save(data []byte) error { ... }
// S3Client satisfies Storage without importing the consumer package
```

---

---

## Practical Examples (LeetCode / Real-World Patterns)

### Iterator Pattern with Interface

```go
type Iterator interface {
    HasNext() bool
    Next() int
}

type SliceIterator struct {
    data  []int
    index int
}

func (it *SliceIterator) HasNext() bool {
    return it.index < len(it.data)
}

func (it *SliceIterator) Next() int {
    val := it.data[it.index]
    it.index++
    return val
}

// Works with ANY iterator implementation
func sum(it Iterator) int {
    total := 0
    for it.HasNext() {
        total += it.Next()
    }
    return total
}
```

### Strategy Pattern — Sort with Custom Comparator

```go
// sort.Interface from the standard library
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

// Custom sort for intervals (LeetCode #56 - Merge Intervals)
type ByStart [][]int

func (a ByStart) Len() int           { return len(a) }
func (a ByStart) Less(i, j int) bool { return a[i][0] < a[j][0] }
func (a ByStart) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func merge(intervals [][]int) [][]int {
    sort.Sort(ByStart(intervals))
    // ... merge logic
}
```

**Interface insight:** `sort.Sort` accepts any type that satisfies `sort.Interface`. Your custom `ByStart` type wraps `[][]int` and provides the three methods. The interface enables polymorphism — one sort algorithm works with any data type.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Interface value | 16 bytes: type pointer + data pointer |
| Satisfaction | Implicit — just implement the methods |
| Empty interface | `any` — every type satisfies it, but you lose type safety |
| Pointer receivers | `*T` satisfies interfaces that `T` alone cannot |
| Nil gotcha | Interface with nil data is NOT a nil interface |
| Design | Small interfaces, accept interfaces, return structs |
