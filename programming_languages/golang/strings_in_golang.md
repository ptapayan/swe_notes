# Strings in Go

## What Is a String?

A string in Go is an **immutable sequence of bytes** (not characters). Internally, it's a simple header:

```go
// runtime representation
type stringHeader struct {
    Data unsafe.Pointer // pointer to byte array (usually UTF-8)
    Len  int            // length in bytes (NOT runes)
}
```

```
Stack:                      Heap (or read-only data segment):
┌──────────────────┐       ┌───┬───┬───┬───┬───┐
│ ptr ─────────────┼──────→│ H │ e │ l │ l │ o │
│ len: 5           │       └───┴───┴───┴───┴───┘
└──────────────────┘
 string header (16 bytes)
```

**Key facts:**
- Strings are **immutable** — you cannot modify individual bytes
- The string header is 16 bytes (pointer + length), always passed by value (cheap)
- The backing bytes are typically in read-only memory for string literals
- Strings are **valid UTF-8 by convention**, but Go does NOT enforce this

---

## Bytes vs Runes vs Characters

This is the most important distinction to understand:

| Term | Type | Size | What it is |
|---|---|---|---|
| Byte | `byte` (`uint8`) | 1 byte | A single raw byte |
| Rune | `rune` (`int32`) | 4 bytes | A Unicode code point |
| String | `string` | 16 bytes header | A sequence of bytes |

```go
s := "Hello, 世界"

len(s)                    // 13 — bytes, NOT characters
utf8.RuneCountInString(s) // 9  — runes (Unicode code points)
```

**Why?** UTF-8 encoding uses variable-length bytes per character:
- ASCII (English): 1 byte per character
- Most European: 2 bytes
- Chinese/Japanese/Korean: 3 bytes
- Emoji: 4 bytes

```go
s := "世" // Chinese character
fmt.Println(len(s))    // 3 — takes 3 bytes in UTF-8
fmt.Println(s[0])      // 228 — first byte (NOT the character)
```

---

## Iterating Over Strings

### By Rune (Usually What You Want)

```go
s := "Hello, 世界"
for i, r := range s {
    fmt.Printf("byte offset %d: %c (U+%04X)\n", i, r, r)
}
// byte offset 0: H (U+0048)
// byte offset 7: 世 (U+4E16)  ← multi-byte, skips to byte 7
// byte offset 10: 界 (U+754C)
```

`range` on a string automatically decodes UTF-8 and yields runes.

**Related:** [for loops (range over string)](for_loops_in_golang.md)

### By Byte (Raw Access)

```go
s := "Hello"
for i := 0; i < len(s); i++ {
    fmt.Printf("byte %d: %x (%c)\n", i, s[i], s[i])
}
```

### Converting to Rune Slice (When You Need Random Access)

```go
s := "Hello, 世界"
runes := []rune(s)
fmt.Println(runes[7]) // 30028 (界) — correct rune access by index
fmt.Println(len(runes)) // 9
```

**Memory cost:** `[]rune` allocates 4 bytes per rune (instead of 1-4 bytes in UTF-8). For "Hello, 世界" that's 36 bytes vs 13 bytes. Only convert when you need random access by character index.

---

## String Immutability

```go
s := "hello"
// s[0] = 'H'  // COMPILE ERROR: cannot assign to s[0]
```

**Why immutable?**
1. Strings can share backing memory safely (no defensive copies needed)
2. String literals live in read-only memory
3. Safe for use as map keys and in concurrent code without locks

To "modify" a string, you create a new one:

```go
s := "hello"
s = "H" + s[1:] // creates new string "Hello"
```

---

## String Conversions

### String ↔ Byte Slice

```go
s := "hello"
b := []byte(s)   // copies the bytes into a new mutable slice
b[0] = 'H'
s2 := string(b)  // copies bytes back into a new immutable string
fmt.Println(s)    // "hello" — original unchanged
fmt.Println(s2)   // "Hello"
```

**Memory:** Both conversions **copy** the data. This is because strings are immutable but byte slices are mutable — sharing memory would break the immutability guarantee.

**Optimization:** The compiler avoids copies in some cases:
- `string(b)` in map lookups: `m[string(b)]` — no copy
- `[]byte(s)` in range: `for i, b := range []byte(s)` — no copy
- String comparison with byte slice conversion

### String ↔ Rune Slice

```go
s := "Hello, 世界"
runes := []rune(s) // decodes UTF-8 into rune slice (copies + decodes)
s2 := string(runes) // encodes runes back to UTF-8 (copies + encodes)
```

### Int ↔ String

```go
// Number to string
s := strconv.Itoa(42)        // "42"
s := fmt.Sprintf("%d", 42)   // "42" (slower, more flexible)

// String to number
n, err := strconv.Atoi("42") // 42, nil
n, err := strconv.Atoi("abc") // 0, error

// DON'T do this:
s := string(65) // "A" — interprets 65 as a rune (Unicode code point), NOT "65"
```

---

## String Concatenation

### The `+` Operator (Simple but Slow for Many Strings)

```go
s := "Hello" + ", " + "World" // creates intermediate strings
```

Each `+` allocates a new string. For n concatenations, this is O(n^2).

### strings.Builder (Efficient for Many Strings)

```go
var b strings.Builder
for i := 0; i < 1000; i++ {
    b.WriteString("hello")
    b.WriteByte(' ')
}
result := b.String()
```

**How Builder works internally:** It maintains a `[]byte` buffer that grows as needed (like `append`). `String()` does a single conversion at the end. Amortized O(n).

### strings.Join (When You Have a Slice)

```go
parts := []string{"Hello", "World", "Go"}
result := strings.Join(parts, ", ") // "Hello, World, Go"
```

Internally, `Join` pre-calculates the total length, allocates once, and copies — most efficient for joining known slices.

### fmt.Sprintf (Flexible but Slowest)

```go
name := "Alice"
age := 30
s := fmt.Sprintf("%s is %d years old", name, age)
```

Uses reflection internally, so it's slower. Use for formatting, not bulk concatenation.

---

## Common String Operations

```go
import "strings"

strings.Contains("hello world", "world")  // true
strings.HasPrefix("hello", "hel")          // true
strings.HasSuffix("hello", "llo")          // true
strings.Index("hello", "ll")               // 2
strings.ToUpper("hello")                   // "HELLO"
strings.ToLower("HELLO")                   // "hello"
strings.TrimSpace("  hello  ")             // "hello"
strings.Trim("!!hello!!", "!")             // "hello"
strings.Replace("aaa", "a", "b", 2)       // "bba" (replace first 2)
strings.ReplaceAll("aaa", "a", "b")        // "bbb"
strings.Split("a,b,c", ",")               // ["a", "b", "c"]
strings.Fields("  hello   world  ")        // ["hello", "world"] (split by whitespace)
strings.Repeat("ha", 3)                    // "hahaha"
strings.Count("hello", "l")               // 2
strings.EqualFold("Hello", "hello")        // true (case-insensitive compare)
```

---

## String Comparison

```go
// Direct comparison (lexicographic, byte-by-byte)
"abc" == "abc"  // true
"abc" < "abd"   // true
"abc" < "abcd"  // true

// Case-insensitive comparison
strings.EqualFold("Go", "go") // true — handles Unicode correctly
// DON'T use strings.ToLower(a) == strings.ToLower(b) — allocates two new strings
```

---

## Multiline Strings (Raw String Literals)

```go
// Interpreted string (processes escape sequences)
s1 := "Hello\nWorld\t!"

// Raw string literal (no escape processing)
s2 := `Hello\nWorld\t!`  // literally contains backslash-n, not a newline
fmt.Println(s2) // Hello\nWorld\t!

// Multiline
query := `
    SELECT id, name
    FROM users
    WHERE age > 18
    ORDER BY name
`
```

Raw strings cannot contain backticks. If you need a backtick in a string, use interpreted strings with concatenation.

---

---

## Practical Examples (LeetCode Patterns)

### Longest Substring Without Repeating Characters (LeetCode #3) — Sliding Window

```go
func lengthOfLongestSubstring(s string) int {
    lastSeen := make(map[byte]int) // char → index
    maxLen := 0
    left := 0

    for right := 0; right < len(s); right++ {
        if idx, ok := lastSeen[s[right]]; ok && idx >= left {
            left = idx + 1
        }
        lastSeen[s[right]] = right
        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}
```

**String insight:** We index by byte `s[right]` — fine for ASCII. For Unicode, you'd need to iterate by rune with `range` or use `[]rune` conversion.

### Valid Palindrome (LeetCode #125) — Rune Handling

```go
func isPalindrome(s string) bool {
    s = strings.ToLower(s)
    runes := []rune{}
    for _, r := range s {
        if unicode.IsLetter(r) || unicode.IsDigit(r) {
            runes = append(runes, r)
        }
    }
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        if runes[i] != runes[j] {
            return false
        }
    }
    return true
}
```

**Why `[]rune`?** We need to compare characters, not bytes. Multi-byte UTF-8 characters would break byte-level comparison.

### Reverse Words in a String (LeetCode #151) — strings.Fields

```go
func reverseWords(s string) string {
    words := strings.Fields(s) // splits by any whitespace, trims automatically
    for i, j := 0, len(words)-1; i < j; i, j = i+1, j-1 {
        words[i], words[j] = words[j], words[i]
    }
    return strings.Join(words, " ")
}
```

**Using the stdlib:** `strings.Fields` handles multiple spaces, leading/trailing spaces — all in one call. `strings.Join` does a single allocation for the result.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Internal | 16-byte header (pointer + length). Immutable byte sequence |
| UTF-8 | `len()` returns bytes, not characters. Use `utf8.RuneCountInString()` for rune count |
| Rune vs byte | `range` iterates by rune. Index `s[i]` accesses a byte |
| Immutability | Can't modify in place. Conversions to `[]byte` copy the data |
| Concatenation | Use `strings.Builder` for loops, `strings.Join` for slices, `+` for simple cases |
| Conversion | `string(65)` is "A" (rune), not "65". Use `strconv.Itoa` for numbers |
