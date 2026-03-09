# Go (Golang) as a Language

## Overview

Go (Golang) was created at **Google in 2007** by Robert Griesemer, Rob Pike, and Ken Thompson. It was publicly released in 2009. It was designed to solve problems Google faced: slow builds, uncontrolled dependencies, hard-to-read code, and difficulty writing concurrent servers.

---

## Compiler and Compilation

### Compiler: Single-Pass, Ahead-of-Time (AOT)

Go uses an **ahead-of-time (AOT) compiler** — source code is compiled to machine code BEFORE execution (not interpreted or JIT-compiled).

```bash
go build main.go    # produces a native binary
./main              # runs directly on the CPU — no VM, no runtime interpreter
```

**Compiler toolchain:**
- **gc** — the standard Go compiler (NOT "garbage collector" here). Compiles to native machine code
- **gccgo** — alternative frontend using GCC's backend (slower compile, sometimes faster binaries)
- The compiler is written in Go itself (self-hosting since Go 1.5)

### Compilation Pipeline

```
Source (.go) → Lexer → Parser → AST → Type Checker → SSA (Static Single Assignment)
→ Optimization → Machine Code → Linker → Static Binary
```

**Key stages:**
1. **Parsing:** Go's grammar is intentionally simple (no header files, no circular imports). This makes parsing fast
2. **Type checking:** Fully static. All types resolved at compile time
3. **Escape analysis:** Decides whether variables live on stack or heap
4. **SSA optimization:** Dead code elimination, inlining, bounds check elimination
5. **Linking:** Produces a **statically linked binary** by default — no shared libraries needed at runtime

### Why Go Compiles So Fast

- No header files (each file is self-contained)
- Explicit imports (unused imports = compile error)
- No circular dependencies between packages
- Simple grammar (no templates/generics complexity like C++)
- Dependency graph is a DAG — packages compile in parallel

---

## Type System

### Statically Typed

Every variable has a type known at compile time. No dynamic types.

```go
var x int = 42      // explicit type
y := "hello"        // type inferred as string at compile time
// y = 42           // COMPILE ERROR: cannot use int as string
```

### Strongly Typed

No implicit conversions between types:

```go
var i int = 42
var f float64 = float64(i) // explicit conversion required
// var f float64 = i        // COMPILE ERROR

var x int32 = 10
var y int64 = int64(x)     // even between int sizes!
```

### Structural Typing (for Interfaces)

Go uses **structural typing** (duck typing, but at compile time). A type satisfies an interface by having the right methods — no `implements` keyword.

```go
type Writer interface {
    Write([]byte) (int, error)
}
// Any type with a Write method satisfies Writer automatically
```

**Related:** [interfaces](golang_interface.md) for deep dive.

---

## Memory Management

### Stack vs Heap

Go has **both** a stack and a heap, and the compiler decides where variables live via **escape analysis**.

```go
func example() *int {
    x := 42     // escape analysis sees this is returned as a pointer
    return &x   // x ESCAPES to the heap
}

func example2() int {
    x := 42     // x stays on the stack (doesn't escape)
    return x
}
```

You can see escape analysis decisions:
```bash
go build -gcflags="-m" main.go
# main.go:5:2: moved to heap: x
```

**Stack:** Fast allocation (just move stack pointer), automatically freed when function returns. Each goroutine has its own stack (starts at 2-8KB, grows dynamically).

**Heap:** Slower allocation, managed by garbage collector. Used for values that outlive the function or whose size is unknown at compile time.

### Garbage Collector

Go uses a **concurrent, tri-color mark-and-sweep garbage collector**.

**Key properties:**
- **Concurrent:** GC runs alongside application goroutines (mostly)
- **Non-generational:** Unlike Java, Go doesn't separate young/old generations
- **Low latency:** GC pause times are typically < 1ms (prioritizes latency over throughput)
- **Tri-color algorithm:** Objects are white (unreachable), grey (reachable, not scanned), black (reachable, scanned)

**GC cycle:**
1. **Mark phase:** Trace all reachable objects from roots (globals, stacks, registers)
2. **Sweep phase:** Free unreachable (white) objects

```go
// You can tune GC (but rarely need to):
import "runtime/debug"
debug.SetGCPercent(200) // GC triggers when heap doubles (default 100 = heap grows 100%)
```

---

## Concurrency Model

Go implements **CSP (Communicating Sequential Processes)** through goroutines and channels.

- **Goroutines:** Lightweight, user-space threads (~2-8KB stack, multiplexed onto OS threads)
- **Channels:** Typed pipes for communication between goroutines
- **M:N scheduler:** M goroutines scheduled across N OS threads

**Related:** [goroutines and channels](go_routines_and_channels.md) for full details.

---

## Package System and Modules

### Packages

Every Go file belongs to a package. The package system enforces:
- **No circular imports** (dependency graph is a DAG)
- **Unused imports are compile errors** (keeps code clean)
- **Exported names start with uppercase** (visibility by naming convention)

```go
package mypackage

func PublicFunction() {}  // exported (visible outside package)
func privateFunction() {} // unexported (only within package)
```

### Modules (Go 1.11+)

```bash
go mod init github.com/user/project  # creates go.mod
go get github.com/lib/pq             # adds a dependency
```

`go.mod` defines the module path and dependencies. `go.sum` contains checksums for verification. Go uses **Minimum Version Selection (MVS)** — always picks the minimum version that satisfies all requirements (deterministic, no SAT solver).

---

## Error Handling Philosophy

Go uses **explicit error returns** instead of exceptions. Errors are values.

```go
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err)
}
```

- No try/catch/finally
- No exception stack unwinding
- Errors are just values that implement the `error` interface
- `panic`/`recover` exist but are reserved for truly unrecoverable situations (not control flow)

**Related:** [if/else (error pattern)](if_else_statements.md) | [interfaces (error interface)](golang_interface.md) | [functions (multiple returns)](function_in_golang.md)

---

## Zero Values

Every type has a zero value — the default when not explicitly initialized:

```go
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // "" (empty string)
var p *int      // nil
var sl []int    // nil
var m map[K]V   // nil
var ch chan int  // nil
var fn func()   // nil
var iface error // nil
```

Go guarantees there are **no uninitialized variables**. This eliminates an entire class of bugs.

---

## No Feature Bloat (Intentional Simplicity)

Go deliberately OMITS many features:

| What Go DOESN'T have | Why |
|---|---|
| Classes / Inheritance | Uses composition via struct embedding |
| Exceptions | Uses explicit error returns |
| Generics (pre-1.18) | Added in Go 1.18 with type parameters |
| Ternary operator `?:` | Use if/else (one way to do things) |
| Default function parameters | Use variadic or option structs |
| Method overloading | Each function name is unique |
| Implicit type conversions | All conversions must be explicit |
| Macros / Preprocessor | What you see is what compiles |
| Operator overloading | Operators have fixed semantics |

---

## Generics (Go 1.18+)

```go
func Map[T any, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

// Type constraints
type Number interface {
    ~int | ~float64 | ~int32 | ~int64
}

func Sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

Go generics use **monomorphization + dictionary passing** hybrid approach (called "GC shape stenciling"). Types with the same GC shape (pointer, int, etc.) share a single compiled function + dictionary.

---

## Toolchain

Go ships with a comprehensive built-in toolchain:

```bash
go build       # compile
go run         # compile and run
go test        # run tests
go fmt         # format code (canonical style — no debates)
go vet         # static analysis for common mistakes
go mod tidy    # clean up dependencies
go doc         # view documentation
go generate    # run code generators
go tool pprof  # profiling
go test -race  # race condition detection
go test -bench # benchmarking
```

**`gofmt`** is especially important: ALL Go code is formatted the same way. No style debates. This was a deliberate design decision.

---

## Binary Output

Go produces **statically linked binaries** by default:
- No runtime dependencies (no libc, no .so/.dll files needed)
- Single binary deployment — copy the file and run it
- Cross-compilation is trivial:

```bash
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go
GOOS=windows GOARCH=amd64 go build -o myapp.exe main.go
GOOS=darwin GOARCH=arm64 go build -o myapp-mac main.go
```

Binary sizes are larger than C (typically 5-15MB for a simple program) because the Go runtime (scheduler, GC, etc.) is included.

---

## Key Takeaways

| Aspect | Go's Approach |
|---|---|
| Compiler | AOT compiled to native machine code. Very fast compilation |
| Type system | Static, strong, structural typing for interfaces |
| Memory | Stack + heap via escape analysis. Concurrent GC with <1ms pauses |
| Concurrency | CSP model: goroutines + channels. M:N scheduling |
| Errors | Explicit values, not exceptions |
| Deployment | Single static binary, trivial cross-compilation |
| Philosophy | Simplicity, readability, fast compilation, one way to do things |
