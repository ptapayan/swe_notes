# Compiling in Go — Deep Dive

**Related:** [Go as a language](golang_as_a_language.md) | [functions (stack frames)](function_in_golang.md) | [structs (memory layout)](structs_in_golang.md) | [best practices (escape analysis, performance)](golang_best_practices.md)

---

## What Kind of Compiler Is Go?

Go uses an **ahead-of-time (AOT)**, **single-pass**, **self-hosting** compiler that produces **statically linked native binaries**.

| Property | Go | Java | Python | C++ |
|---|---|---|---|---|
| Compilation | AOT → machine code | AOT → bytecode → JIT | Interpreted (+ bytecode) | AOT → machine code |
| Runtime needed | Embedded in binary | JVM required | CPython interpreter | None (libc) |
| Binary output | Single static binary | .class / .jar files | .pyc bytecode | ELF / Mach-O binary |
| Startup time | Instant (native code) | Slow (JVM boot + JIT warmup) | Moderate | Instant |

**Key distinction:** Go embeds its entire runtime (scheduler, GC, networking) INTO the binary. There's no external VM. The binary runs directly on the OS kernel.

---

## The Compilation Pipeline

```
┌──────────┐    ┌────────┐    ┌─────┐    ┌──────────────┐    ┌─────┐
│ Source    │───→│ Lexer  │───→│ AST │───→│ Type Checker │───→│ SSA │
│ (.go)    │    │ Parser │    │     │    │              │    │     │
└──────────┘    └────────┘    └─────┘    └──────────────┘    └─────┘
                                                                │
                                                                ▼
┌──────────┐    ┌────────┐    ┌─────────────────┐    ┌──────────────────┐
│ Binary   │◄───│ Linker │◄───│ Machine Code    │◄───│ SSA Optimization │
│ (ELF/    │    │        │    │ Generation      │    │ + Escape Analysis│
│  Mach-O) │    └────────┘    └─────────────────┘    └──────────────────┘
└──────────┘
```

Let's walk through each stage and what it's actually doing.

---

## Stage 1: Lexing and Parsing

### Lexer (Tokenization)

The lexer reads raw source bytes and produces **tokens** — the smallest meaningful units.

```go
// Source code:
x := 42 + y

// Tokens produced:
// IDENT("x")  DEFINE(":=")  INT("42")  ADD("+")  IDENT("y")  SEMICOLON
```

**Go's semicolon insertion rule:** You never write semicolons in Go. The lexer automatically inserts them after any line that ends with:
- An identifier (`x`, `myFunc`)
- A literal (`42`, `"hello"`)
- One of: `break`, `continue`, `fallthrough`, `return`
- One of: `++`, `--`, `)`, `]`, `}`

This is why you MUST put `{` on the same line as `if`/`for`/`func`:

```go
// This breaks because the lexer inserts a semicolon after "true"
if true    // ← semicolon inserted here
{          // ← this becomes a separate block statement = compile error
}

// This works:
if true {
}
```

### Parser (AST Construction)

The parser takes tokens and builds an **Abstract Syntax Tree (AST)** — a tree representation of the program's structure.

```go
// Source:
func add(a, b int) int {
    return a + b
}
```

```
AST:
FuncDecl
├── Name: "add"
├── Params: [a int, b int]
├── Results: [int]
└── Body:
    └── ReturnStmt
        └── BinaryExpr
            ├── X: Ident("a")
            ├── Op: ADD
            └── Y: Ident("b")
```

**Why Go parses so fast:**
- **No header files** — each file is self-contained. C++ must parse transitive includes (sometimes millions of lines)
- **No circular imports** — dependency graph is a DAG, packages compile in topological order
- **Simple grammar** — no angle-bracket generics ambiguity (pre-1.18 used `[T any]`), no multiple inheritance resolution, no preprocessor macros
- **Unused imports = error** — the compiler never processes dead imports

You can inspect the AST yourself:

```go
import "go/parser"
import "go/ast"
import "go/token"

fset := token.NewFileSet()
node, _ := parser.ParseFile(fset, "example.go", src, 0)
ast.Print(fset, node) // prints the full AST
```

---

## Stage 2: Type Checking

The type checker walks the AST and:
1. **Resolves all identifiers** — matches every name to its declaration
2. **Infers types** — figures out that `x := 42` means `x` is `int`
3. **Checks type correctness** — ensures `int + string` is an error
4. **Evaluates constant expressions** — `const x = 3 * 4` becomes `12` at compile time

```go
var x int = 42
var y float64 = x  // TYPE ERROR: cannot use x (type int) as type float64
var z float64 = float64(x) // OK: explicit conversion

// Constant evaluation happens here:
const size = 1 << 20  // computed at compile time = 1048576
```

**Untyped constants** are a Go specialty. They have much higher precision than their typed counterparts:

```go
const pi = 3.14159265358979323846264338327950288419716939937510
// This constant has arbitrary precision at compile time.
// Only when assigned to a variable does it get truncated to float64.
var f float64 = pi // truncated to float64 precision
```

---

## Stage 3: SSA (Static Single Assignment) Generation

The type-checked AST is converted to **SSA form** — an intermediate representation where every variable is assigned exactly once. This is where the real optimization magic happens.

```go
// Source:
func example(x int) int {
    y := x + 1
    y = y * 2
    return y
}

// SSA form (simplified):
v1 = Parameter x
v2 = v1 + 1
v3 = v2 * 2
Return v3
```

**Why SSA?** When each variable is assigned once, optimization passes become much simpler:
- **Dead code elimination** — if a value is never used, remove it
- **Constant propagation** — if `v1 = 5`, replace all uses of `v1` with `5`
- **Copy propagation** — if `v2 = v1`, replace all uses of `v2` with `v1`
- **Common subexpression elimination** — if `a + b` is computed twice, reuse the result

You can view the SSA for any function:

```bash
GOSSAFUNC=example go build main.go
# Produces ssa.html — open in browser to see every SSA optimization pass
```

This generates an interactive HTML file showing how your function transforms through ~30 optimization passes. Extremely useful for understanding what the compiler does.

---

## Stage 4: Optimization Passes

### Escape Analysis

The compiler's most performance-critical decision: **does this variable live on the stack or the heap?**

```go
func noEscape() int {
    x := 42      // x stays on stack (doesn't escape)
    return x     // copied to caller's frame, x is freed when function returns
}

func escapes() *int {
    x := 42      // x ESCAPES to heap (returned as pointer)
    return &x    // caller needs x to outlive this function
}

func alsoEscapes() any {
    x := 42      // x ESCAPES (boxed into interface)
    return x     // interface{} needs a heap-allocated copy
}
```

**What triggers escape:**
| Pattern | Escapes? | Why |
|---|---|---|
| Return pointer to local | Yes | Outlives function |
| Store in global/package var | Yes | Outlives function |
| Send to channel | Yes | Another goroutine needs it |
| Store in interface (`any`) | Yes | Boxing requires indirection |
| Closure captures variable | Yes | Closure outlives function |
| Slice too large for stack | Yes | Stack has size limits |
| `make([]T, n)` where n is variable | Yes | Size unknown at compile time |
| Small struct returned by value | No | Copied to caller's stack |
| Local variable never shared | No | Lives and dies with function |

```bash
# View escape analysis decisions:
go build -gcflags="-m" ./...
# -m=2 for more detail

# Example output:
# ./main.go:5:2: moved to heap: x
# ./main.go:10:6: s does not escape
# ./main.go:15:12: leaking param: p
```

**Stack is free. Heap has a cost.** Stack allocation is just moving a pointer. Heap allocation requires the GC to track, scan, and eventually free the memory. In a hot path, the difference is measured in nanoseconds per allocation, which at 100k req/sec adds up fast.

### Function Inlining

The compiler replaces small function calls with the function body directly. This eliminates call overhead (push stack frame, jump, return) and enables further optimizations.

```go
func add(a, b int) int { return a + b } // small enough to inline

func main() {
    result := add(3, 4) // compiler replaces this with: result := 3 + 4
    // Further optimization: result := 7 (constant folded)
}
```

**Inlining budget:** Go uses a cost model. Each AST node has a cost. If a function's total cost is below the threshold (~80), it gets inlined. Things that prevent inlining:
- `go` and `defer` statements
- Loops (in older Go versions)
- Calls to non-inlinable functions (but mid-stack inlining helps since Go 1.12)

```bash
# See what gets inlined:
go build -gcflags="-m" ./...
# ./main.go:3:6: can inline add
# ./main.go:7:14: inlining call to add
```

**Practical tip:** If a function is on a very hot path and isn't being inlined, you can sometimes restructure it to reduce its "cost" (e.g., extract the cold error path into a separate function).

### Bounds Check Elimination (BCE)

Go slices have runtime bounds checking. But the compiler can often prove an access is safe and skip the check:

```go
func sum(s []int) int {
    total := 0
    for i := 0; i < len(s); i++ {
        total += s[i] // compiler KNOWS i < len(s), skips bounds check
    }
    return total
}

// range loops also get BCE automatically
for _, v := range s {
    total += v // no bounds check needed
}
```

```go
// Hint to the compiler for manual BCE:
func process(s []int) {
    _ = s[3] // early bounds check — panics here if len < 4
    // Compiler now knows len(s) >= 4, skips checks below:
    a := s[0] // no check
    b := s[1] // no check
    c := s[2] // no check
    d := s[3] // no check
}
```

```bash
# See bounds checks:
go build -gcflags="-d=ssa/check_bce/debug=1" ./...
```

### Dead Code Elimination

```go
const debug = false

func process() {
    if debug {
        log.Println("debug info") // entire block removed at compile time
    }
    // ... real code ...
}
```

Because `debug` is a constant, the `if false { ... }` block is eliminated entirely — no runtime cost, not even a branch instruction.

---

## Stage 5: Machine Code Generation

The optimized SSA is lowered to architecture-specific machine instructions.

```go
func add(a, b int) int {
    return a + b
}
```

On amd64, this becomes roughly:

```asm
add:
    MOVQ    BX, AX       ; move b to AX
    ADDQ    CX, AX       ; add a (in CX) to AX
    RET                   ; return (result in AX)
```

Go uses its own assembler syntax (Plan 9 syntax), which differs from Intel/AT&T syntax.

You can see the generated assembly:

```bash
# Disassemble a function
go tool objdump -s 'main.add' ./binary

# Or compile to assembly directly
go build -gcflags="-S" main.go

# Or use go tool compile
go tool compile -S main.go
```

### Register-Based Calling Convention (Go 1.17+)

Before Go 1.17, all function arguments were passed on the **stack**. This was simpler but slower.

```
Go < 1.17 (stack-based):
  caller pushes args onto stack → callee reads from stack
  ~4-10% overhead vs register passing

Go >= 1.17 (register-based):
  first args go in CPU registers (AX, BX, CX, DI, SI, R8-R11)
  overflow args go on stack
  ~5% performance improvement for typical code
```

This is why upgrading Go versions can give you free performance improvements without changing code.

---

## Stage 6: Linking

The linker combines:
1. Compiled packages (.a files)
2. The Go runtime (scheduler, GC, netpoller)
3. System call stubs
4. Debug info (DWARF)

Into a **single static binary**.

```bash
# Default: static linking (except CGO)
go build -o myapp main.go
file myapp
# myapp: ELF 64-bit LSB executable, statically linked

# Check binary size and what's inside:
go tool nm myapp | head -20

# Reduce binary size:
go build -ldflags="-s -w" -o myapp main.go
# -s: strip symbol table
# -w: strip DWARF debug info
# Typically reduces size by 30-40%
```

### Static vs Dynamic Linking

```bash
# Pure Go: statically linked by default (no dependencies at runtime)
go build -o myapp main.go
ldd myapp  # "not a dynamic executable"

# With CGO: dynamically linked to libc by default
CGO_ENABLED=1 go build -o myapp main.go
ldd myapp  # shows libc.so dependency

# Force static even with CGO:
CGO_ENABLED=1 go build -ldflags="-linkmode external -extldflags '-static'" -o myapp main.go
```

**Why static linking matters:**
- Deploy a single binary — no "it works on my machine" issues
- No dependency conflicts between applications on the same server
- Container images can be tiny (FROM scratch + binary)
- Cross-compilation just works (no cross-compiling shared libraries)

---

## Cross-Compilation

Go makes cross-compilation trivially easy because the compiler is entirely self-contained:

```bash
# Build for Linux on a Mac
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go

# Build for Windows on a Mac
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go

# Build for ARM (Raspberry Pi, AWS Graviton)
GOOS=linux GOARCH=arm64 go build -o myapp-arm main.go

# Build for WebAssembly
GOOS=js GOARCH=wasm go build -o myapp.wasm main.go
```

**All valid GOOS/GOARCH combinations:**

```bash
go tool dist list
# Shows ~50 combinations: linux/amd64, darwin/arm64, windows/amd64,
# freebsd/amd64, android/arm64, ios/arm64, js/wasm, wasip1/wasm, etc.
```

No additional toolchains or SDK installs required (unless using CGO).

---

## Build Modes

```bash
# Default: executable
go build -o myapp main.go

# Shared library (C-compatible)
go build -buildmode=c-shared -o mylib.so lib.go

# Static C archive
go build -buildmode=c-archive -o mylib.a lib.go

# Plugin (dynamically loaded at runtime)
go build -buildmode=plugin -o myplugin.so plugin.go
```

### Build Tags (Conditional Compilation)

```go
//go:build linux && amd64
// +build linux,amd64

package mypackage
// This file is only compiled on linux/amd64
```

```go
//go:build !windows

package mypackage
// This file is compiled everywhere EXCEPT Windows
```

Common uses: platform-specific code, debug vs release builds, feature flags at compile time.

---

## Build Cache

Go caches compiled packages to avoid recompiling unchanged code:

```bash
# Where the cache lives:
go env GOCACHE
# typically: ~/.cache/go-build

# Clear the cache:
go clean -cache

# See what's being cached:
go build -x ./... 2>&1 | grep "cache"
```

The cache keys on: source file content, compiler flags, Go version, dependencies. If nothing changed, the build is nearly instant (just linking).

---

## What's in the Binary? (Runtime)

A Go binary includes the full runtime:

```
┌─────────────────────────────────────┐
│            Go Binary                │
├─────────────────────────────────────┤
│ Your code (compiled to native)      │
├─────────────────────────────────────┤
│ Standard library code (used parts)  │
├─────────────────────────────────────┤
│ Goroutine Scheduler (G/M/P)        │  ~manages all goroutines
├─────────────────────────────────────┤
│ Garbage Collector                   │  ~concurrent tri-color mark-sweep
├─────────────────────────────────────┤
│ Network Poller (epoll/kqueue)       │  ~async I/O without blocking threads
├─────────────────────────────────────┤
│ Memory Allocator                    │  ~tcmalloc-inspired, per-P mcache
├─────────────────────────────────────┤
│ Stack Manager                       │  ~growable goroutine stacks
├─────────────────────────────────────┤
│ Reflection / Type Metadata          │  ~for reflect, fmt, encoding/json
├─────────────────────────────────────┤
│ DWARF Debug Info (unless stripped)  │
└─────────────────────────────────────┘
```

**Typical binary sizes:**
- Hello World: ~1.8 MB (most of it is the runtime + fmt)
- Simple HTTP server: ~6-8 MB
- Production service: ~10-25 MB
- With `-ldflags="-s -w"`: 30-40% smaller

---

## Compile-Time Tricks

### Embed Files in Binary

```go
import "embed"

//go:embed templates/*.html
var templates embed.FS

//go:embed version.txt
var version string

//go:embed logo.png
var logo []byte
```

Files are embedded at compile time — no runtime file I/O needed. The binary is self-contained.

### Inject Variables at Build Time

```go
// main.go
var (
    Version   string // set at build time
    BuildTime string
    GitCommit string
)

func main() {
    fmt.Printf("version=%s build=%s commit=%s\n", Version, BuildTime, GitCommit)
}
```

```bash
go build -ldflags="\
  -X main.Version=1.2.3 \
  -X main.BuildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -X main.GitCommit=$(git rev-parse --short HEAD)" \
  -o myapp main.go
```

This is how production Go services embed version info without config files.

### Compile-Time Interface Checks

```go
// Ensure *MyServer satisfies http.Handler at COMPILE time, not runtime
var _ http.Handler = (*MyServer)(nil)

// If MyServer doesn't have ServeHTTP, this is a compile error
// Cost: zero (it's an untyped nil assignment to a blank identifier)
```

### go:generate (Code Generation)

```go
//go:generate stringer -type=Color
type Color int

const (
    Red Color = iota
    Green
    Blue
)
```

```bash
go generate ./...
# Runs: stringer -type=Color
# Produces: color_string.go with String() method for Color
```

Common generators: `stringer`, `mockgen`, `protoc-gen-go`, `sqlc`, `ent`.

---

## Debugging the Compiler

```bash
# Show all compiler flags
go tool compile -help

# Show what the compiler is doing step by step
go build -x ./...

# Show assembly output
go build -gcflags="-S" main.go

# Show escape analysis
go build -gcflags="-m=2" main.go

# Show bounds check elimination
go build -gcflags="-d=ssa/check_bce/debug=1" main.go

# Show inlining decisions
go build -gcflags="-m" main.go

# Generate SSA HTML visualization for a specific function
GOSSAFUNC=myFunction go build main.go
# Open ssa.html in browser

# Benchmark compilation itself
time go build -a ./...  # -a forces rebuild of everything
```

---

## Practical Examples

### Optimizing a Hot Path with Compiler Knowledge

```go
// BEFORE: 3 heap allocations per call
func formatUser(name string, age int) string {
    return fmt.Sprintf("%s is %d years old", name, age) // uses reflection, allocates
}

// AFTER: 0 heap allocations (for small strings)
func formatUser(name string, age int) string {
    var buf [64]byte                   // stack-allocated fixed buffer
    b := buf[:0]                       // slice backed by stack array
    b = append(b, name...)
    b = append(b, " is "...)
    b = strconv.AppendInt(b, int64(age), 10)
    b = append(b, " years old"...)
    return string(b)                   // single allocation for the returned string
}
```

**Why this is faster:** `fmt.Sprintf` uses reflection (slow), creates intermediate allocations, and calls interface methods. The manual version uses `strconv.AppendInt` which writes directly into our buffer. The `[64]byte` array lives on the stack (fixed size, known at compile time).

### Build for Minimal Container

```dockerfile
# Multi-stage build
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
# Final image: ~5-10MB (just the binary, no OS, no libc, nothing)
```

`FROM scratch` is possible BECAUSE Go produces a fully static binary. No runtime dependencies.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Pipeline | Source → Lex → Parse → Type Check → SSA → Optimize → Machine Code → Link |
| Escape analysis | Stack = free. Heap = GC cost. Pointers/interfaces/closures force heap |
| Inlining | Small functions auto-inlined. Eliminates call overhead + enables further opts |
| BCE | Compiler skips bounds checks when it can prove safety (range, known lengths) |
| Static binary | Runtime embedded. No external deps. Cross-compile with GOOS/GOARCH |
| Build flags | `-gcflags="-m"` for escape, `-S` for assembly, `GOSSAFUNC` for SSA visualization |
| Binary size | `-ldflags="-s -w"` strips 30-40%. `FROM scratch` for minimal containers |
| Register ABI | Go 1.17+ passes args in registers — free ~5% speedup on upgrade |
