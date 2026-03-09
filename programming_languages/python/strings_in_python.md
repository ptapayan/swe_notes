# Strings in Python

**Related:** [data types](data_types_in_python.md) | [lists & tuples](lists_and_tuples.md) | [best practices](python_best_practices.md)

---

## What Is a String?

A Python `str` is an **immutable sequence of Unicode code points**. Unlike Go (which stores UTF-8 bytes), Python strings give you direct access to characters.

```python
s = "Hello, World!"
type(s)       # <class 'str'>
len(s)        # 13 (characters, NOT bytes)
s[0]          # 'H' (a character, NOT a byte)
```

---

## Internal Representation (PEP 393)

Since Python 3.3, CPython uses **PEP 393 compact representation**. The internal encoding adapts based on the widest character in the string:

```
PEP 393 storage strategy:

If all chars are ASCII (U+0000..U+007F):
  → Latin-1 encoding: 1 byte per character

If widest char is U+0100..U+FFFF (e.g., Chinese, Japanese, most emoji):
  → UCS-2 encoding: 2 bytes per character

If widest char is U+10000+ (e.g., some emoji, rare symbols):
  → UCS-4 encoding: 4 bytes per character
```

```
PyUnicodeObject (compact form):
┌──────────────────────────┐
│ ob_refcnt (8B)           │
│ ob_type → str (8B)       │
│ length (8B)              │  <-- number of code points
│ hash (8B)                │  <-- cached hash value (-1 if not computed)
│ state flags              │  <-- encoding kind, interned flag, ascii flag
│ ... inline data ...      │  <-- actual character data (1/2/4 bytes each)
└──────────────────────────┘
```

```python
import sys

# ASCII string: 1 byte per char + overhead
sys.getsizeof("hello")          # 54 bytes (49 overhead + 5 chars * 1 byte)

# Latin-1 string: still 1 byte per char
sys.getsizeof("caf\u00e9")     # 54 bytes (49 + 4 * 1 + NUL)

# UCS-2 string: 2 bytes per char (has a Chinese char)
sys.getsizeof("hi\u4e16")      # 80 bytes (74 + 3 * 2)

# UCS-4 string: 4 bytes per char (has an emoji)
sys.getsizeof("hi\U0001F600")  # 92 bytes (80 + 3 * 4)
```

**Key insight:** One wide character upgrades the ENTIRE string to the wider encoding. `"a" + "\U0001F600"` stores both characters as 4 bytes each.

---

## Immutability

Strings cannot be modified in place. Every "modification" creates a new string object.

```python
s = "hello"
# s[0] = 'H'   # TypeError: 'str' object does not support item assignment

# "Modifying" creates new strings:
s = "H" + s[1:]      # new string "Hello" (s now points to different object)
s = s.upper()          # new string "HELLO"
s = s.replace("L", "l")  # new string "HEllO"
```

**Why immutable?**
1. **Hashable** — strings can be dict keys and set members
2. **Thread-safe** — no locks needed for concurrent access
3. **Interning** — CPython can reuse string objects
4. **Security** — strings can't be modified through aliased references

---

## String Interning

CPython automatically **interns** (caches and reuses) certain strings:
- String literals that look like identifiers (alphanumeric + underscore)
- String constants known at compile time

```python
a = "hello"
b = "hello"
a is b          # True — same object (interned)

a = "hello world"
b = "hello world"
a is b          # May be True or False (depends on context/optimization)

a = "hello!"
b = "hello!"
a is b          # Often False (punctuation prevents interning)

# Force interning:
import sys
a = sys.intern("hello world!")
b = sys.intern("hello world!")
a is b          # True — explicitly interned
```

**When interning matters:** In applications that compare many strings (parsers, symbol tables), interning turns `O(n)` string comparison into `O(1)` identity comparison (`is` instead of `==`).

---

## f-strings vs format() vs %

```python
name = "Alice"
age = 30

# f-string (Python 3.6+) — fastest, most readable
s = f"{name} is {age} years old"

# str.format()
s = "{} is {} years old".format(name, age)
s = "{name} is {age} years old".format(name=name, age=age)

# % formatting (C-style, legacy)
s = "%s is %d years old" % (name, age)
```

**Performance comparison:**
```
f-string:     ~100ns (bytecode-level concatenation, no method call)
.format():    ~250ns (method call + parsing format spec)
% formatting: ~200ns (method call + C-level formatting)
```

**How f-strings work internally:** The compiler translates `f"{name} is {age}"` into a `FORMAT_VALUE` bytecode followed by string concatenation. No runtime parsing of the format string.

```python
# f-string expressions (Python 3.12+ can have any expression):
f"{2 ** 10}"                    # "1024"
f"{name!r}"                     # "'Alice'" (calls repr)
f"{3.14159:.2f}"                # "3.14" (format spec)
f"{1_000_000:,}"                # "1,000,000" (thousands separator)
f"{'hello':>10}"                # "     hello" (right-aligned, width 10)
f"{'hello':*^20}"               # "*******hello********" (center, fill with *)
```

---

## Encoding and Decoding (str vs bytes)

```
str (Unicode code points)       bytes (raw byte sequence)
    "Hello, 世界"          ←decode→    b'\x48\x65\x6c\x6c\x6f...'
                            ↕
                     encode('utf-8')
                     decode('utf-8')
```

```python
# str → bytes (encoding)
s = "Hello, 世界"
b = s.encode('utf-8')     # b'Hello, \xe4\xb8\x96\xe7\x95\x8c'
len(b)                     # 13 (7 ASCII bytes + 3 bytes per Chinese char)

# bytes → str (decoding)
s = b.decode('utf-8')     # "Hello, 世界"

# Common encodings:
s.encode('ascii')          # UnicodeEncodeError (Chinese chars not in ASCII)
s.encode('utf-8')          # variable length (1-4 bytes per char)
s.encode('utf-16')         # 2 bytes per char (with BOM)
s.encode('latin-1')        # 1 byte per char (only U+0000..U+00FF)

# Handling encoding errors:
s.encode('ascii', errors='ignore')    # b'Hello, '
s.encode('ascii', errors='replace')   # b'Hello, ??'
s.encode('ascii', errors='xmlcharrefreplace')  # b'Hello, &#19990;&#30028;'
```

**Rule of thumb:** Use `str` for text processing. Use `bytes` for I/O (files, network, APIs). Encode at the boundary (when writing), decode at the boundary (when reading).

---

## Common String Operations

```python
s = "Hello, World!"

# Searching:
s.find("World")          # 7 (index, or -1 if not found)
s.index("World")         # 7 (same, but raises ValueError if not found)
s.count("l")             # 3
"World" in s             # True (preferred for existence check)
s.startswith("Hello")    # True
s.endswith("!")          # True

# Transforming (all return NEW strings):
s.upper()               # "HELLO, WORLD!"
s.lower()               # "hello, world!"
s.title()               # "Hello, World!"
s.swapcase()            # "hELLO, wORLD!"
s.strip()               # removes whitespace from both ends
s.lstrip()              # left strip
s.rstrip()              # right strip
s.replace("World", "Python")  # "Hello, Python!"

# Splitting and joining:
"a,b,c".split(",")      # ['a', 'b', 'c']
"  hello  world  ".split()  # ['hello', 'world'] (splits on any whitespace)
",".join(["a", "b", "c"])   # "a,b,c"

# Checking:
"abc123".isalnum()       # True
"abc".isalpha()          # True
"123".isdigit()          # True
"   ".isspace()          # True
```

---

## Regular Expressions (Basics)

```python
import re

# Search for pattern:
match = re.search(r'\d+', 'abc123def456')
match.group()           # '123' (first match)
match.span()            # (3, 6) (start, end indices)

# Find all matches:
re.findall(r'\d+', 'abc123def456')  # ['123', '456']

# Substitution:
re.sub(r'\d+', 'NUM', 'abc123def456')  # 'abcNUMdefNUM'

# Compile for reuse (faster in loops):
pattern = re.compile(r'^(\w+)\s+(\d+)$')
match = pattern.match("Alice 95")
match.groups()          # ('Alice', '95')

# Common patterns:
# \d  digit        \D  non-digit
# \w  word char    \W  non-word char
# \s  whitespace   \S  non-whitespace
# .   any char     ^   start    $   end
# *   0 or more    +   1 or more    ?   0 or 1
# ()  capture      (?:) non-capturing group
```

**Performance note:** Python's `re` module releases the GIL during matching, so regex operations are thread-friendly. For heavy regex workloads, consider the `regex` third-party module (supports Unicode properties, fuzzy matching).

---

## Practical Examples / LeetCode Patterns

### Valid Palindrome (LeetCode 125)

```python
def isPalindrome(s):
    # Filter and lowercase in one pass:
    cleaned = [c.lower() for c in s if c.isalnum()]
    return cleaned == cleaned[::-1]

# More memory-efficient (two pointers, no extra list):
def isPalindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True

# What's happening: s[i] is O(1) character access (not byte access like Go).
# No need for rune conversion — Python strings are already Unicode.
```

### Longest Palindromic Substring (LeetCode 5)

```python
def longestPalindrome(s):
    if not s:
        return ""

    start, max_len = 0, 1

    def expand(left, right):
        nonlocal start, max_len
        while left >= 0 and right < len(s) and s[left] == s[right]:
            if right - left + 1 > max_len:
                start = left
                max_len = right - left + 1
            left -= 1
            right += 1

    for i in range(len(s)):
        expand(i, i)       # odd-length palindromes
        expand(i, i + 1)   # even-length palindromes

    return s[start:start + max_len]

# Slicing s[start:start+max_len] creates a new string (copies characters).
# O(n^2) time, O(1) space (not counting the output).
```

### Group Anagrams String Pattern (LeetCode 49)

```python
from collections import defaultdict

def groupAnagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        # Count characters — O(k) vs O(k log k) for sorting
        count = [0] * 26
        for c in s:
            count[ord(c) - ord('a')] += 1
        groups[tuple(count)].append(s)
    return list(groups.values())

# ord(c) returns the Unicode code point as an int.
# chr(n) converts back: chr(97) == 'a'.
# ord('a') == 97, so ord(c) - ord('a') gives 0-25 for lowercase letters.
```

---

## String Building Patterns

```python
# For interview coding: use list + join
def build_string(words):
    parts = []
    for word in words:
        parts.append(word.upper())
    return " ".join(parts)

# One-liner with generator:
result = " ".join(word.upper() for word in words)

# io.StringIO — for complex string building (rare in interviews)
import io
buffer = io.StringIO()
buffer.write("Hello")
buffer.write(", ")
buffer.write("World")
result = buffer.getvalue()  # "Hello, World"
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Internal | PEP 393: 1/2/4 bytes per char depending on widest character in string |
| Immutability | Every "modification" creates a new string. Enables hashing and interning |
| Interning | CPython caches identifier-like strings. `sys.intern()` for manual interning |
| len() | Returns character count (code points), NOT byte count. Unlike Go |
| Indexing | `s[i]` returns a character (str of length 1), NOT a byte. O(1) |
| f-strings | Fastest formatting. Compiled to bytecode. Use for all new code |
| Encoding | `str.encode()` → bytes, `bytes.decode()` → str. Always specify encoding |
| Concatenation | Use `"".join(list)` in loops. `+=` is O(n^2) |
| Comparison | `==` for equality, `<` for lexicographic. Case-insensitive: `.lower()` or `.casefold()` |
