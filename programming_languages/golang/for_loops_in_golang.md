# For Loops in Go

**Related:** [slices vs arrays](slices_vs_arrays.md) | [maps (iteration order)](maps_in_golang.md) | [strings (rune iteration)](strings_in_golang.md) | [goroutines (range over channel)](go_routines_and_channels.md)

## The Only Loop Keyword

Go has **one** looping construct: `for`. It replaces `while`, `do-while`, and `for` from other languages.

---

## Classic Three-Component For Loop

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

**What's happening:**
- `i` is scoped to the loop block (not visible outside)
- `i := 0` runs once before the first iteration
- `i < 10` is checked before each iteration
- `i++` runs after each iteration body

**Memory:** `i` lives on the stack. The compiler may even keep it in a CPU register for the entire loop since it's a simple integer counter — never touches RAM.

---

## While-Style Loop

```go
count := 0
for count < 5 {
    fmt.Println(count)
    count++
}
```

Just a `for` with only the condition. This is Go's `while`.

---

## Infinite Loop

```go
for {
    // runs forever until break, return, or os.Exit
    fmt.Println("forever")
}
```

Common pattern with a `select` for event loops:

```go
for {
    select {
    case msg := <-ch:
        handle(msg)
    case <-done:
        return
    }
}
```

---

## Range Loop

`range` iterates over slices, arrays, maps, strings, and channels.

### Range Over Slice

```go
fruits := []string{"apple", "banana", "cherry"}

// Both index and value
for i, fruit := range fruits {
    fmt.Printf("%d: %s\n", i, fruit)
}

// Index only
for i := range fruits {
    fmt.Println(i) // 0, 1, 2
}

// Value only (discard index)
for _, fruit := range fruits {
    fmt.Println(fruit)
}
```

**What's happening in memory:**
- `range` evaluates the slice expression **once** before the loop starts
- It copies the **slice header** (pointer, length, capacity) — NOT the underlying array
- On each iteration, `fruit` is a **copy** of the element (pass-by-value)
- Modifying `fruit` does NOT modify the slice

```go
nums := []int{1, 2, 3}
for _, n := range nums {
    n *= 2 // modifies the copy only — nums is unchanged
}
fmt.Println(nums) // [1 2 3]

// To modify: use the index
for i := range nums {
    nums[i] *= 2
}
fmt.Println(nums) // [2 4 6]
```

### Range Over Map

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}
```

**Critical:** Map iteration order is **randomized** by the Go runtime. Never depend on map ordering. The runtime deliberately randomizes it to prevent code from accidentally depending on a specific order.

### Range Over String

```go
s := "Hello, 世界"
for i, r := range s {
    fmt.Printf("byte offset %d: %c (U+%04X)\n", i, r, r)
}
// byte offset 0: H (U+0048)
// byte offset 1: e (U+0065)
// ...
// byte offset 7: 世 (U+4E16)  ← 3 bytes in UTF-8
// byte offset 10: 界 (U+754C)
```

**Key insight:** `range` over a string iterates by **rune** (Unicode code point), not by byte. The index `i` is the **byte offset**, and `r` is the decoded `rune` (type `int32`). Multi-byte UTF-8 characters are decoded automatically.

If you want to iterate by byte:

```go
for i := 0; i < len(s); i++ {
    fmt.Printf("byte %d: %x\n", i, s[i])
}
```

### Range Over Channel

```go
ch := make(chan int)

go func() {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // must close, or range will block forever
}()

for val := range ch {
    fmt.Println(val) // 0, 1, 2, 3, 4
}
// Loop exits when channel is closed and drained
```

---

## Loop Variable Behavior (Go 1.22+ Change)

### Before Go 1.22 (Old Behavior)

```go
// BUG in Go < 1.22
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // all goroutines see i=3
    }()
}
```

The loop variable was **one variable reused across all iterations**. Closures captured the variable, not the value.

### Go 1.22+ (New Behavior)

Each iteration creates a **new** variable. The closure bug is gone:

```go
// Works correctly in Go 1.22+
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // each goroutine gets its own i (0, 1, 2)
    }()
}
```

**How to think about it:** In Go 1.22+, `for i := ...` behaves as if there's an implicit `i := i` at the start of each iteration.

---

## Break, Continue, Labels

### Break and Continue

```go
for i := 0; i < 10; i++ {
    if i == 3 {
        continue // skip the rest of this iteration
    }
    if i == 7 {
        break // exit the loop entirely
    }
    fmt.Println(i) // 0, 1, 2, 4, 5, 6
}
```

### Labels (for Nested Loops)

```go
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if i == 1 && j == 1 {
                break outer // breaks the OUTER loop
            }
            fmt.Printf("i=%d j=%d\n", i, j)
        }
    }
// Without the label, break would only exit the inner loop
```

---

## Performance Considerations

### Pre-calculate Length

```go
// The compiler is smart enough to optimize len(slice) in for loops
// but for function calls, be explicit:

// Inefficient — calls expensiveLen() every iteration
for i := 0; i < expensiveLen(); i++ { ... }

// Better
n := expensiveLen()
for i := 0; i < n; i++ { ... }
```

### Avoid Allocations in Loops

```go
// Bad: creates a new slice each iteration
for i := 0; i < n; i++ {
    buf := make([]byte, 1024)
    process(buf)
}

// Good: reuse the buffer
buf := make([]byte, 1024)
for i := 0; i < n; i++ {
    process(buf)
}
```

### Range with Large Structs

```go
type BigStruct struct {
    Data [1024]byte
}

items := []BigStruct{ ... }

// Copies BigStruct on each iteration (1024 bytes per copy!)
for _, item := range items {
    _ = item
}

// Better: use index to avoid the copy
for i := range items {
    _ = items[i]
}
```

---

---

## Practical Examples / LeetCode Patterns

### Two Pointers (LeetCode 11: Container With Most Water)

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0

    for left < right { // classic while-style loop
        h := min(height[left], height[right])
        water := h * (right - left)
        if water > maxWater {
            maxWater = water
        }
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}
```

### BFS with Queue (LeetCode 102: Binary Tree Level Order Traversal)

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 { // while-style: loop until queue is empty
        levelSize := len(queue)
        level := make([]int, 0, levelSize)

        for i := 0; i < levelSize; i++ { // classic for: process current level
            node := queue[0]
            queue = queue[1:] // dequeue (note: doesn't free memory, see slices)
            level = append(level, node.Val)

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
```

**Loop patterns used:** Outer `for len(queue) > 0` is a while-style loop. Inner `for i := 0; i < levelSize; i++` is classic three-component. This nested pattern is typical for BFS.

### Nested Loops with Break Label (LeetCode 36: Valid Sudoku)

```go
func isValidSudoku(board [][]byte) bool {
    var rows, cols [9][9]bool
    var boxes [3][3][9]bool

    for i := 0; i < 9; i++ {
        for j := 0; j < 9; j++ {
            if board[i][j] == '.' {
                continue // skip empty cells
            }
            num := board[i][j] - '1'
            if rows[i][num] || cols[j][num] || boxes[i/3][j/3][num] {
                return false
            }
            rows[i][num] = true
            cols[j][num] = true
            boxes[i/3][j/3][num] = true
        }
    }
    return true
}
```

**Memory insight:** All tracking arrays are fixed-size and live on the stack. Zero heap allocation. The `continue` skips the rest of the inner loop body — a clean way to handle the "skip" case.

---

---

## Practical Examples (LeetCode Patterns)

### Two Pointers (LeetCode #11: Container With Most Water)

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0

    for left < right { // while-style loop
        h := min(height[left], height[right])
        water := h * (right - left)
        if water > maxWater {
            maxWater = water
        }
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}
```

### BFS Level Order Traversal (LeetCode #102)

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 { // outer while-style: until queue empty
        levelSize := len(queue)
        level := make([]int, 0, levelSize)

        for i := 0; i < levelSize; i++ { // inner classic for: process level
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
```

**Loop patterns:** Outer `for len(queue) > 0` is while-style. Inner `for i := 0; i < levelSize; i++` is classic. This nested pattern is the standard Go approach for BFS.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Only `for` | Go has one loop keyword. `for` does everything |
| `range` copy | The loop variable is a copy of the element, not a reference |
| Map order | Randomized. Never depend on iteration order |
| String range | Iterates by rune (Unicode), index is byte offset |
| Go 1.22+ | Loop variables are per-iteration now. Closure bug is fixed |
| Performance | Reuse buffers, avoid copies of large structs, use index for mutation |
