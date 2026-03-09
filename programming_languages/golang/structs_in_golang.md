# Structs in Go

**Related:** [Go as a language](golang_as_a_language.md) | [interfaces (method sets)](golang_interface.md) | [functions (methods with receivers)](function_in_golang.md) | [maps (struct as values)](maps_in_golang.md)

## What Is a Struct?

A struct is a composite type that groups together fields of different types. It's Go's primary way to create custom data types (Go has no classes).

```go
type Person struct {
    Name string
    Age  int
    Email string
}
```

---

## Memory Layout

Structs are stored as **contiguous blocks of memory** with fields laid out in declaration order.

```go
type Example struct {
    A bool    // 1 byte
    B int64   // 8 bytes
    C bool    // 1 byte
}
```

**But it's NOT 10 bytes.** Due to **alignment**, the compiler inserts **padding**:

```
Offset  Field   Size   Padding
0       A       1      7 bytes padding (B needs 8-byte alignment)
8       B       8
16      C       1      7 bytes padding (struct must be multiple of largest alignment)
Total: 24 bytes
```

### Optimize by Ordering Fields

```go
// Bad: 24 bytes (lots of padding)
type Bad struct {
    A bool    // 1 + 7 padding
    B int64   // 8
    C bool    // 1 + 7 padding
}

// Good: 16 bytes (minimal padding)
type Good struct {
    B int64   // 8
    A bool    // 1
    C bool    // 1 + 6 padding
}
```

**Rule of thumb:** Order fields from largest to smallest alignment to minimize padding. For hot structs in performance-critical code, this can matter (cache lines are 64 bytes).

You can verify with:
```go
fmt.Println(unsafe.Sizeof(Bad{}))  // 24
fmt.Println(unsafe.Sizeof(Good{})) // 16
```

---

## Creating Structs

```go
// Named fields (preferred — order doesn't matter, self-documenting)
p1 := Person{
    Name:  "Alice",
    Age:   30,
    Email: "alice@example.com",
}

// Positional (fragile — breaks if you add/reorder fields)
p2 := Person{"Bob", 25, "bob@example.com"}

// Zero value — all fields get their zero values
var p3 Person // Name="", Age=0, Email=""

// Using new (returns a pointer)
p4 := new(Person) // *Person, all fields zeroed
```

---

## Value Semantics (Pass-by-Value)

Structs are values. Assigning or passing a struct **copies all fields**.

```go
func birthday(p Person) {
    p.Age++ // modifies the COPY
}

func main() {
    alice := Person{Name: "Alice", Age: 30}
    birthday(alice)
    fmt.Println(alice.Age) // still 30 — the copy was modified, not alice
}
```

**What's happening in memory:**
1. `alice` is a struct on the stack (let's say 40 bytes for Name header + int + Email header)
2. Calling `birthday(alice)` copies all 40 bytes into a new stack frame
3. `p.Age++` modifies the copy's Age field
4. When `birthday` returns, the copy is discarded

### Use Pointers to Modify

```go
func birthday(p *Person) {
    p.Age++ // modifies the original through the pointer
}

func main() {
    alice := Person{Name: "Alice", Age: 30}
    birthday(&alice)
    fmt.Println(alice.Age) // 31
}
```

**When to use pointer vs value:**
- **Value:** Small structs (< ~64 bytes), immutable data, when you want a copy
- **Pointer:** Large structs, when you need to modify the original, when the struct has methods with pointer receivers

---

## Methods on Structs

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Value receiver — works on a copy
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver — can modify the original
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Println(rect.Area()) // 50

    rect.Scale(2) // Go auto-takes address: (&rect).Scale(2)
    fmt.Println(rect.Area()) // 200
}
```

---

## Struct Embedding (Composition Over Inheritance)

Go has no inheritance. Instead, you embed structs to compose behavior.

```go
type Address struct {
    Street string
    City   string
    State  string
}

type Employee struct {
    Person  // embedded — fields and methods are "promoted"
    Address // embedded
    Salary  float64
}

func main() {
    e := Employee{
        Person:  Person{Name: "Alice", Age: 30, Email: "alice@co.com"},
        Address: Address{Street: "123 Main", City: "Portland", State: "OR"},
        Salary:  100000,
    }

    // Promoted fields — accessed directly
    fmt.Println(e.Name)   // "Alice" (from Person)
    fmt.Println(e.City)   // "Portland" (from Address)
    fmt.Println(e.Salary) // 100000

    // Can also access explicitly
    fmt.Println(e.Person.Name) // "Alice"
}
```

**What's happening in memory:** Embedding is NOT a pointer. The embedded struct is laid out inline:

```
Employee memory layout:
┌─────────────────────┐
│ Person.Name (string) │  ← Person fields are inline here
│ Person.Age (int)     │
│ Person.Email (string)│
│ Address.Street       │  ← Address fields inline here
│ Address.City         │
│ Address.State        │
│ Salary (float64)     │
└─────────────────────┘
```

### Method Promotion

If `Person` has methods, `Employee` gets them too:

```go
func (p Person) Greet() string {
    return "Hi, I'm " + p.Name
}

func main() {
    e := Employee{Person: Person{Name: "Alice"}}
    fmt.Println(e.Greet()) // "Hi, I'm Alice" — promoted method
}
```

If `Employee` defines its own `Greet()`, it **shadows** the embedded one (not overriding — Go has no polymorphism through embedding).

---

## Struct Tags

Metadata attached to fields, commonly used for serialization.

```go
type User struct {
    ID        int    `json:"id" db:"user_id"`
    FirstName string `json:"first_name" db:"first_name"`
    Password  string `json:"-"`                          // excluded from JSON
    Email     string `json:"email,omitempty"`             // omitted if empty
}

func main() {
    u := User{ID: 1, FirstName: "Alice", Email: "alice@example.com"}
    data, _ := json.Marshal(u)
    fmt.Println(string(data))
    // {"id":1,"first_name":"Alice","email":"alice@example.com"}
}
```

**How tags work internally:** Tags are stored as string literals in the type metadata at compile time. Libraries like `encoding/json` use the `reflect` package to read them at runtime:

```go
t := reflect.TypeOf(User{})
field, _ := t.FieldByName("FirstName")
fmt.Println(field.Tag.Get("json")) // "first_name"
```

Reflection is slow (involves runtime type inspection), so hot paths should avoid it. Libraries typically cache the tag lookups.

---

## Comparing Structs

Structs are comparable if ALL their fields are comparable:

```go
type Point struct {
    X, Y int
}

p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true — field-by-field comparison
```

**NOT comparable if any field is a slice, map, or function:**

```go
type Data struct {
    Values []int // slices are NOT comparable
}

// d1 == d2  // COMPILE ERROR
// Use reflect.DeepEqual(d1, d2) instead (slow but works)
```

---

## Anonymous Structs

Useful for one-off structures, especially in tests and JSON unmarshalling:

```go
// Inline struct
point := struct {
    X, Y int
}{X: 10, Y: 20}

// Common in tests
tests := []struct {
    name     string
    input    int
    expected int
}{
    {"positive", 5, 25},
    {"zero", 0, 0},
    {"negative", -3, 9},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        if got := square(tt.input); got != tt.expected {
            t.Errorf("square(%d) = %d, want %d", tt.input, got, tt.expected)
        }
    })
}
```

---

## Constructor Pattern (No Constructors in Go)

Go has no constructors. The convention is a `New` function:

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

func NewServer(host string, port int) *Server {
    return &Server{
        host:    host,
        port:    port,
        timeout: 30 * time.Second, // sensible default
    }
}
```

**Memory note:** `&Server{...}` does NOT always allocate on the heap. Escape analysis determines this. If the pointer doesn't escape the caller, it stays on the stack.

---

---

## Practical Examples (LeetCode Patterns)

### Min Stack (LeetCode #155) — Struct with Multiple Fields

```go
type MinStack struct {
    stack []entry
}

type entry struct {
    val int
    min int // tracks the minimum at this point in the stack
}

func Constructor() MinStack {
    return MinStack{}
}

func (s *MinStack) Push(val int) {
    min := val
    if len(s.stack) > 0 && s.stack[len(s.stack)-1].min < val {
        min = s.stack[len(s.stack)-1].min
    }
    s.stack = append(s.stack, entry{val: val, min: min})
}

func (s *MinStack) Pop() {
    s.stack = s.stack[:len(s.stack)-1]
}

func (s *MinStack) GetMin() int {
    return s.stack[len(s.stack)-1].min
}
```

**Struct pattern:** Each stack entry bundles the value WITH the minimum at that point. This is O(1) for all operations because the struct carries the state.

### Binary Tree Traversal — Struct as Tree Node

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Inorder traversal (Left → Node → Right)
func inorder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    var result []int
    result = append(result, inorder(root.Left)...)
    result = append(result, root.Val)
    result = append(result, inorder(root.Right)...)
    return result
}
```

**Memory insight:** `*TreeNode` pointers are 8 bytes. The actual nodes live on the heap (escape analysis: returned pointers prevent stack allocation). The tree structure is just structs pointing to each other.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Memory layout | Contiguous, with alignment padding. Order fields large-to-small |
| Pass-by-value | Assigning/passing copies all fields. Use pointers for large structs or mutation |
| Embedding | Composition, not inheritance. Fields and methods are promoted |
| Tags | String metadata for serialization. Read via reflect (cached by libraries) |
| Comparability | Only if all fields are comparable. No slices, maps, or functions |
| Constructors | Convention: `NewTypeName()` function returning `*Type` |
