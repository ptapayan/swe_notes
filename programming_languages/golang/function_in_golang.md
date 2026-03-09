# Functions in Go

**Related:** [Go as a language](golang_as_a_language.md) | [structs (methods)](structs_in_golang.md) | [interfaces (method sets)](golang_interface.md) | [goroutines (go keyword)](go_routines_and_channels.md)

## What Is a Function at a Low Level?

A function in Go is a block of machine instructions stored in the **text segment** of the binary. When you call a function, the runtime:

1. Pushes a new **stack frame** onto the goroutine's stack
2. Copies arguments into that frame (Go is **pass-by-value**, always)
3. Jumps to the function's instruction address
4. On return, copies return values back and pops the frame

Go does NOT use the system thread stack (which is typically 1-8MB). Each goroutine gets its own stack starting at **2KB-8KB** that grows dynamically (by copying the whole stack to a bigger allocation when needed).

---

## Basic Function Declaration

```go
func add(a int, b int) int {
    return a + b
}
```

**What's happening in memory:**
- `a` and `b` are copied into the new stack frame (pass-by-value)
- The return value `int` is also pre-allocated space in the caller's stack frame
- After `return`, the result is written to that pre-allocated slot

---

## Multiple Return Values

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10.0, 3.0)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(result) // 3.3333333333333335
}
```

**Key insight:** Multiple return values are NOT packed into a tuple or struct. The compiler allocates space for each return value separately in the caller's stack frame. This is a zero-cost abstraction — no heap allocation.

---

## Named Return Values (Naked Returns)

```go
func rectangleArea(width, height float64) (area float64) {
    area = width * height
    return // "naked return" — returns the named variable
}
```

**How to think about this:** Named return values are just local variables that are pre-initialized to their zero value. The `return` statement without arguments returns whatever those variables currently hold. Use sparingly — they hurt readability in longer functions.

---

## Variadic Functions

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))       // 6
    fmt.Println(sum(1, 2, 3, 4, 5)) // 15

    // Passing a slice with the spread operator
    numbers := []int{10, 20, 30}
    fmt.Println(sum(numbers...))     // 60
}
```

**What's happening under the hood:** `nums ...int` is syntactic sugar. The compiler creates a `[]int` slice from the arguments. The slice header (pointer, length, capacity) lives on the stack, but if there are many arguments, the backing array may escape to the heap.

When you use `numbers...` to spread an existing slice, **no new slice is created** — the original slice is passed directly.

---

## Functions as First-Class Citizens

Functions in Go are values. They can be assigned to variables, passed as arguments, and returned from other functions.

```go
func applyOperation(a, b int, op func(int, int) int) int {
    return op(a, b)
}

func main() {
    multiply := func(x, y int) int {
        return x * y
    }

    result := applyOperation(3, 4, multiply)
    fmt.Println(result) // 12

    // Inline anonymous function
    result2 := applyOperation(10, 5, func(x, y int) int {
        return x - y
    })
    fmt.Println(result2) // 5
}
```

**Memory detail:** A function value is internally a pointer to a `funcval` struct that contains the function's code pointer. When there are no captured variables, this is a simple static pointer (no allocation).

---

## Closures

A closure is a function that captures variables from its enclosing scope.

```go
func makeCounter() func() int {
    count := 0 // this variable is captured
    return func() int {
        count++
        return count
    }
}

func main() {
    counter := makeCounter()
    fmt.Println(counter()) // 1
    fmt.Println(counter()) // 2
    fmt.Println(counter()) // 3

    // Each call to makeCounter creates a NEW closure with its own `count`
    counter2 := makeCounter()
    fmt.Println(counter2()) // 1 (independent)
}
```

**What's happening in memory:**
- `count` would normally live on `makeCounter`'s stack frame, which gets destroyed on return
- Because the closure references `count`, the compiler performs **escape analysis** and moves `count` to the **heap**
- The closure struct holds a **pointer** to the heap-allocated `count`
- This is why `counter()` can keep incrementing — it's modifying a heap variable through a pointer
- The garbage collector will free `count` when no closures reference it anymore

**Common closure trap:**

```go
// BUG: all goroutines share the same `i` variable
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // likely prints "5" five times
    }()
}

// FIX: pass i as a parameter (creates a copy)
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n) // prints 0, 1, 2, 3, 4 (in some order)
    }(i)
}
```

**Note:** As of Go 1.22+, the loop variable is per-iteration by default, so the bug above is fixed in modern Go. But understanding why it happened is still important.

---

## Defer

`defer` schedules a function call to run when the enclosing function returns.

```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // guaranteed to run when readFile returns

    // ... read from f ...
    return nil
}
```

**How defer works internally:**
- Deferred calls are pushed onto a **LIFO stack** (last in, first out)
- Arguments to deferred functions are **evaluated immediately** (not when the defer executes)
- Since Go 1.14, most defers are "open-coded" — the compiler inlines them, making defer nearly zero-cost

```go
func deferOrder() {
    defer fmt.Println("first")  // runs third
    defer fmt.Println("second") // runs second
    defer fmt.Println("third")  // runs first
    fmt.Println("main body")
}
// Output:
// main body
// third
// second
// first
```

**Argument evaluation gotcha:**

```go
func main() {
    x := 10
    defer fmt.Println(x) // captures x=10 NOW
    x = 20
    fmt.Println(x) // prints 20
}
// Output:
// 20
// 10   <-- defer used the value at the time of the defer statement
```

If you need the final value, use a closure:

```go
func main() {
    x := 10
    defer func() {
        fmt.Println(x) // closure captures the variable, not the value
    }()
    x = 20
}
// Output:
// 20   <-- closure sees the final value of x
```

---

## Methods (Functions with Receivers)

Methods are functions attached to a type. There are two kinds of receivers.

### Value Receiver

```go
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}
```

- `c` is a **copy** of the Circle
- Modifying `c.Radius` inside this method does NOT affect the original
- Safe to call on both values and pointers (Go auto-dereferences)

### Pointer Receiver

```go
func (c *Circle) Scale(factor float64) {
    c.Radius *= factor // modifies the original Circle
}
```

- `c` is a **pointer** to the original Circle (8 bytes on 64-bit systems)
- Can modify the original struct
- Avoids copying large structs

**Rule of thumb:** If ANY method needs a pointer receiver, make ALL methods use pointer receivers for consistency. This also matters for interface satisfaction.

```go
func main() {
    c := Circle{Radius: 5.0}
    fmt.Println(c.Area())  // 78.539...
    c.Scale(2)
    fmt.Println(c.Area())  // 314.159... (radius is now 10)
}
```

---

## init() Functions

Special functions that run automatically before `main()`.

```go
var config map[string]string

func init() {
    // Runs once when the package is loaded
    config = make(map[string]string)
    config["env"] = "production"
}
```

**Execution order:**
1. Package-level variable declarations (in dependency order)
2. `init()` functions (in the order they appear in the file, and files in alphabetical order)
3. `main()` function

A single file can have **multiple** `init()` functions. They all run. Avoid heavy logic in `init()` — it makes testing harder and creates hidden dependencies.

---

---

## Practical Examples / LeetCode Patterns

### Recursion with Memoization (LeetCode 70: Climbing Stairs)

```go
func climbStairs(n int) int {
    memo := make(map[int]int)

    var climb func(n int) int
    climb = func(n int) int {
        if n <= 2 {
            return n
        }
        if val, ok := memo[n]; ok {
            return val
        }
        memo[n] = climb(n-1) + climb(n-2)
        return memo[n]
    }

    return climb(n)
}
```

**What's happening:** `climb` is a closure (anonymous recursive function) that captures `memo` from the outer scope. `memo` escapes to the heap because it's shared across recursive calls. The `var climb func(n int) int` declaration is needed so the closure can reference itself.

### Multiple Return Values for Error Handling (LeetCode 8: String to Integer)

```go
func myAtoi(s string) int {
    s = strings.TrimSpace(s)
    if len(s) == 0 {
        return 0
    }

    sign := 1
    i := 0
    if s[0] == '+' || s[0] == '-' {
        if s[0] == '-' {
            sign = -1
        }
        i++
    }

    result := 0
    for ; i < len(s) && s[i] >= '0' && s[i] <= '9'; i++ {
        digit := int(s[i] - '0')
        if result > (math.MaxInt32-digit)/10 {
            if sign == 1 {
                return math.MaxInt32
            }
            return math.MinInt32
        }
        result = result*10 + digit
    }
    return sign * result
}
```

### Closures as Comparators (Custom Sorting)

```go
// LeetCode 56: Merge Intervals
func merge(intervals [][]int) [][]int {
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })
    // The anonymous function captures `intervals` from outer scope
    // sort.Slice uses it as a less-than comparator

    merged := [][]int{intervals[0]}
    for _, interval := range intervals[1:] {
        last := merged[len(merged)-1]
        if interval[0] <= last[1] {
            last[1] = max(last[1], interval[1])
        } else {
            merged = append(merged, interval)
        }
    }
    return merged
}
```

---

---

## Practical Examples (LeetCode Patterns)

### Generate Parentheses (LeetCode #22) — Recursion + Closures

```go
func generateParenthesis(n int) []string {
    result := []string{}

    var backtrack func(current string, open, close int)
    backtrack = func(current string, open, close int) {
        if len(current) == 2*n {
            result = append(result, current)
            return
        }
        if open < n {
            backtrack(current+"(", open+1, close)
        }
        if close < open {
            backtrack(current+")", open, close+1)
        }
    }

    backtrack("", 0, 0)
    return result
}
```

**Function pattern:** `backtrack` is a closure that captures `result` and `n`. It's recursive — each call adds a new stack frame. The closure lets us avoid passing `result` and `n` as parameters through every recursive call.

### Memoization with Closures — Fibonacci

```go
func fib() func(int) int {
    cache := map[int]int{}

    var f func(int) int
    f = func(n int) int {
        if n <= 1 {
            return n
        }
        if v, ok := cache[n]; ok {
            return v
        }
        cache[n] = f(n-1) + f(n-2)
        return cache[n]
    }
    return f
}

func main() {
    fibonacci := fib()
    fmt.Println(fibonacci(50)) // instant — O(n) instead of O(2^n)
}
```

**Closure + memory:** The `cache` map lives on the heap (captured by the closure). Each call to `fib()` creates a new independent cache. The map grows as needed and is garbage collected when `fibonacci` goes out of scope.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Pass-by-value | Everything is copied. Slices, maps, channels copy the header (which contains a pointer), not the underlying data |
| Stack vs Heap | Escape analysis decides. If a variable outlives the function, it goes to the heap |
| Closures | Capture variables by reference (pointer to heap). Be careful in loops |
| Defer | LIFO order, args evaluated immediately, nearly zero-cost since Go 1.14 |
| Receivers | Value = copy, Pointer = modify original. Be consistent per type |
