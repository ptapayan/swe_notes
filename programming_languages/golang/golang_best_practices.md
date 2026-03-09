# Go Best Practices (FAANG-Level Engineering)

**Related:** [Go as a language](golang_as_a_language.md) | [functions](function_in_golang.md) | [goroutines & channels](go_routines_and_channels.md) | [slices vs arrays](slices_vs_arrays.md) | [maps](maps_in_golang.md) | [structs](structs_in_golang.md) | [interfaces](golang_interface.md)

---

## 1. Memory Performance

### 1.1 Pre-Allocate Slices and Maps

Every time a slice or map grows, Go allocates new memory and copies everything. In a hot path, this kills performance.

```go
// BAD: 10+ allocations + copies for 1000 items
func collectIDs(users []User) []int {
    var ids []int // nil slice, cap=0
    for _, u := range users {
        ids = append(ids, u.ID) // grows: 0→1→2→4→8→16→...→1024
    }
    return ids
}

// GOOD: 1 allocation, 0 copies
func collectIDs(users []User) []int {
    ids := make([]int, 0, len(users)) // pre-allocate exact capacity
    for _, u := range users {
        ids = append(ids, u.ID) // never reallocates
    }
    return ids
}
```

Same for maps:

```go
// BAD: rehashes multiple times as it grows
result := make(map[string]int)

// GOOD: hint avoids rehashing
result := make(map[string]int, len(input))
```

**Why this matters at scale:** A service processing 100k requests/sec with a slice that grows 10 times per request = 1M unnecessary allocations/sec. Each allocation pressures the GC, increases latency, and wastes CPU on copying.

### 1.2 Reuse Buffers with sync.Pool

`sync.Pool` is a GC-aware free list. Objects are cached and reused across goroutines. The GC can reclaim pooled objects — this is NOT a memory leak.

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func handleRequest(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer) // get from pool (or allocate new)
    defer func() {
        buf.Reset()       // clear contents but keep allocated memory
        bufPool.Put(buf)  // return to pool for reuse
    }()

    buf.Write(data)
    buf.WriteString(" processed")
    return buf.String()
}
```

**When to use sync.Pool:**
- High-frequency allocations of the same type (HTTP handlers, serialization buffers)
- Objects that are expensive to create
- Short-lived objects that would otherwise churn the GC

**When NOT to use:** Long-lived objects, objects with complex state that's hard to reset, or infrequent allocations (pool overhead > allocation cost).

### 1.3 Avoid Heap Allocations in Hot Paths

The Go compiler's **escape analysis** decides whether a variable goes on the stack (free, instant) or heap (requires GC). You can check:

```bash
go build -gcflags="-m" ./...
# output shows: "moved to heap: x", "does not escape", etc.
```

**Common things that force heap allocation:**

```go
// 1. Returning a pointer to a local variable
func newUser() *User {
    u := User{Name: "Alice"} // ESCAPES to heap (returned as pointer)
    return &u
}

// 2. Closures that capture variables
func counter() func() int {
    count := 0 // ESCAPES (captured by closure that outlives this function)
    return func() int {
        count++
        return count
    }
}

// 3. Interface conversions (value must be boxed)
func log(msg any) { ... } // any argument escapes to heap
log("hello")              // "hello" gets boxed on the heap

// 4. Slice/map values that are too large or dynamically sized
data := make([]byte, size) // if size is not a constant, escapes

// 5. Sending pointers to goroutines
go func(u *User) { ... }(user) // user may escape
```

**How to reduce heap allocations:**

```go
// 1. Pass by value for small structs (avoid pointer escape)
func process(u User) { ... }  // stays on stack if User is small

// 2. Use fixed-size arrays instead of slices when size is known
var buf [256]byte // stack-allocated, no GC pressure
// vs
buf := make([]byte, 256) // may heap-allocate

// 3. Avoid interface{}/any in hot paths
func addInts(a, b int) int { return a + b }     // no boxing
func addAny(a, b any) any { return a.(int) + b.(int) } // boxing + unboxing

// 4. Use value receivers for small types
func (p Point) Distance(other Point) float64 { ... } // no heap escape
```

### 1.4 Struct Field Ordering (Memory Alignment)

CPU reads memory in aligned chunks. Misaligned fields add padding.

```go
// BAD: 32 bytes (wasted padding)
type Bad struct {
    Active bool    // 1 byte + 7 padding
    Balance float64 // 8 bytes
    Age    int32   // 4 bytes + 4 padding
    Admin  bool    // 1 byte + 7 padding
}

// GOOD: 24 bytes (minimal padding)
type Good struct {
    Balance float64 // 8 bytes
    Age     int32   // 4 bytes
    Active  bool    // 1 byte
    Admin   bool    // 1 byte + 2 padding
}
```

**Rule:** Order fields from largest alignment to smallest. Verify with `unsafe.Sizeof()`.

**When it matters:** Structs stored in large slices. 1 million items × 8 bytes wasted = 8MB wasted. Also impacts CPU cache utilization — fewer bytes per struct = more structs per cache line.

### 1.5 Beware of Hidden Memory Retention

```go
// BAD: sub-slice keeps the entire original backing array alive
func getFirstTen(data []byte) []byte {
    return data[:10]
    // The GC cannot free the rest of `data` because this slice
    // still points to the same backing array
}

// GOOD: copy to detach from original
func getFirstTen(data []byte) []byte {
    result := make([]byte, 10)
    copy(result, data[:10])
    return result
    // Now the original `data` backing array can be GC'd
}
```

Same issue with strings:

```go
// BAD: s[0:5] retains the entire original string's memory
func extractPrefix(s string) string {
    return s[:5] // holds reference to full string
}

// GOOD: force a copy via concatenation or strings.Clone (Go 1.20+)
func extractPrefix(s string) string {
    return strings.Clone(s[:5])
}
```

---

## 2. Concurrency

### 2.1 Know When to Use Channels vs Mutexes

**Use channels when:**
- Passing ownership of data between goroutines
- Coordinating between goroutines (signaling, pipeline stages)
- Fan-out/fan-in patterns

**Use mutexes when:**
- Protecting a shared data structure (cache, counter, config)
- Simple lock/unlock patterns
- Performance-critical sections (mutex is faster than channel for simple protection)

```go
// CHANNEL: transferring ownership
func producer(out chan<- Task) {
    for task := range generateTasks() {
        out <- task // ownership transfers to consumer
    }
    close(out)
}

// MUTEX: shared state protection
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}
```

### 2.2 Bounded Concurrency (Never Launch Unbounded Goroutines)

```go
// BAD: 1 million goroutines at once — OOM, connection exhaustion
for _, url := range urls { // urls has 1M entries
    go fetch(url)
}

// GOOD: bounded worker pool
func fetchAll(urls []string, concurrency int) {
    sem := make(chan struct{}, concurrency) // semaphore
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        sem <- struct{}{} // blocks when concurrency limit reached
        go func(u string) {
            defer wg.Done()
            defer func() { <-sem }()
            fetch(u)
        }(url)
    }
    wg.Wait()
}
```

**FAANG reality:** Unbounded goroutines hit connection pool limits, file descriptor limits, or trigger OOM kills. At Google/Meta scale, one runaway service can cascade and take down dependent services.

### 2.3 Always Use Context for Cancellation

Every operation that can block or take time should accept a `context.Context`.

```go
// BAD: no way to cancel
func fetchData(url string) ([]byte, error) {
    resp, err := http.Get(url) // blocks indefinitely if server is slow
    ...
}

// GOOD: respects cancellation and deadlines
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    resp, err := http.DefaultClient.Do(req)
    ...
}

// Caller controls timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
data, err := fetchData(ctx, "https://api.example.com/data")
```

**Rules:**
- `ctx` is always the first parameter
- Always `defer cancel()` to prevent resource leaks
- Check `ctx.Done()` in long-running loops
- Never store Context in a struct — pass it explicitly

### 2.4 Avoid Goroutine Leaks

A goroutine leak is when a goroutine blocks forever and can never be cleaned up.

```go
// LEAK: if nobody reads from ch, this goroutine blocks forever
func leaky() {
    ch := make(chan int)
    go func() {
        val := doExpensiveWork()
        ch <- val // blocked forever if nobody reads
    }()
    // function returns, ch is unreachable, goroutine is stuck
}

// FIX 1: use buffered channel (send doesn't block)
func fixed() {
    ch := make(chan int, 1) // buffer of 1
    go func() {
        ch <- doExpensiveWork() // won't block even if nobody reads
    }()
}

// FIX 2: use context for cancellation
func fixed2(ctx context.Context) {
    ch := make(chan int)
    go func() {
        val := doExpensiveWork()
        select {
        case ch <- val:
        case <-ctx.Done(): // exit if context cancelled
        }
    }()
}
```

**Detection:** Use `runtime.NumGoroutine()` in tests or monitoring. If it keeps growing, you have a leak.

```go
func TestNoLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    // ... run your code ...
    time.Sleep(100 * time.Millisecond)
    after := runtime.NumGoroutine()
    if after > before+1 {
        t.Errorf("goroutine leak: before=%d after=%d", before, after)
    }
}
```

### 2.5 Protect Shared Maps (They WILL Crash)

```go
// THIS WILL CRASH at production scale. Guaranteed.
var cache = map[string]string{}

func handler(w http.ResponseWriter, r *http.Request) {
    cache[r.URL.Path] = "visited" // concurrent write → panic
    // fatal error: concurrent map writes
}
```

Go detects concurrent map access and intentionally panics (since Go 1.6). This is not a race condition you might get away with — it is a guaranteed crash under concurrent load.

---

## 3. Error Handling

### 3.1 Wrap Errors with Context

```go
// BAD: loses context — "file not found" from where?
if err != nil {
    return err
}

// GOOD: adds context at each layer
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("loadConfig: reading %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("loadConfig: parsing %s: %w", path, err)
    }
    return &cfg, nil
}
// Error: "loadConfig: reading /etc/app.json: open /etc/app.json: permission denied"
```

**Use `%w` (not `%v`)** to wrap errors. This preserves the error chain for `errors.Is()` and `errors.As()`:

```go
if errors.Is(err, os.ErrNotExist) {
    // works through any number of %w wraps
}

var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("failed path:", pathErr.Path)
}
```

### 3.2 Define Sentinel Errors and Custom Error Types

```go
// Sentinel errors — for known, expected error conditions
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("conflict")
)

func GetUser(id string) (*User, error) {
    user, ok := db.Find(id)
    if !ok {
        return nil, fmt.Errorf("GetUser(%s): %w", id, ErrNotFound)
    }
    return user, nil
}

// Caller checks:
if errors.Is(err, ErrNotFound) {
    http.Error(w, "user not found", http.StatusNotFound)
}
```

```go
// Custom error type — when you need structured error data
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
}

// Caller extracts:
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("field %s: %s\n", valErr.Field, valErr.Message)
}
```

### 3.3 Don't Panic in Libraries

```go
// BAD: panicking in a library crashes the entire application
func ParseConfig(data []byte) *Config {
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        panic("invalid config: " + err.Error()) // caller can't recover gracefully
    }
    return &cfg
}

// GOOD: return error — let the caller decide
func ParseConfig(data []byte) (*Config, error) {
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return &cfg, nil
}
```

**When panic IS acceptable:**
- `init()` functions (failing to load required config at startup)
- Programming errors that indicate a bug (index out of bounds by broken logic, not user input)
- In `main()` when there's no sensible way to continue

---

## 4. API and Code Design

### 4.1 Accept Interfaces, Return Structs

```go
// GOOD: accepts interface (flexible — any io.Reader works)
func ProcessData(r io.Reader) error {
    data, err := io.ReadAll(r)
    // works with files, HTTP bodies, buffers, compressed streams, test mocks...
}

// GOOD: returns concrete type (specific — caller knows exactly what they get)
func NewServer(addr string) *Server {
    return &Server{addr: addr, timeout: 30 * time.Second}
}
```

**Why?**
- Accepting interfaces = maximum flexibility for callers + easy testing with mocks
- Returning structs = no hidden allocations, caller can access all methods, no interface boxing

### 4.2 Keep Interfaces Small

```go
// BAD: "fat" interface — hard to implement, hard to mock
type UserService interface {
    GetUser(id string) (*User, error)
    CreateUser(u *User) error
    UpdateUser(u *User) error
    DeleteUser(id string) error
    ListUsers(filter Filter) ([]*User, error)
    SearchUsers(query string) ([]*User, error)
    GetUserPreferences(id string) (*Prefs, error)
    // ... 15 more methods
}

// GOOD: focused interfaces — easy to implement, easy to mock
type UserReader interface {
    GetUser(id string) (*User, error)
}

type UserWriter interface {
    CreateUser(u *User) error
    UpdateUser(u *User) error
}

// Compose when you need both:
type UserReadWriter interface {
    UserReader
    UserWriter
}
```

### 4.3 Functional Options Pattern (For Complex Configuration)

```go
type Server struct {
    addr    string
    port    int
    timeout time.Duration
    logger  *slog.Logger
}

type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:    addr,
        port:    8080,              // sensible default
        timeout: 30 * time.Second,  // sensible default
        logger:  slog.Default(),    // sensible default
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Clean usage:
srv := NewServer("localhost",
    WithPort(9090),
    WithTimeout(10*time.Second),
)
```

**Why this over a config struct:** Optional parameters with defaults, backward-compatible (adding a new option doesn't break existing callers), self-documenting.

### 4.4 Use Struct Embedding for Composition, Not Inheritance

```go
// BAD mental model: "Server IS-A Logger"
// GOOD mental model: "Server HAS-A Logger"

type Server struct {
    *slog.Logger // embedded — Server "has" logging capability
    addr string
}

func (s *Server) Start() {
    s.Info("starting server", "addr", s.addr) // promoted method from Logger
}
```

**Pitfall:** Embedding promotes ALL methods. If the embedded type satisfies an interface, the outer type does too — this can cause accidental interface satisfaction. Be intentional about what you embed.

---

## 5. Testing

### 5.1 Table-Driven Tests

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed", -1, 5, 4},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

**Why table-driven:** Easy to add new cases, clear failure messages via `t.Run` (shows which case failed), consistent structure across the codebase.

### 5.2 Always Run Tests with the Race Detector

```bash
go test -race ./...
```

The race detector instruments memory accesses at compile time and catches unsynchronized concurrent access at runtime. It slows tests ~2-10x but catches bugs that would be nearly impossible to find in production.

**At FAANG:** Race detector is typically part of CI. Code that fails `-race` does not merge.

### 5.3 Benchmark Before Optimizing

```go
func BenchmarkConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "hello" // O(n²) — allocates a new string each time
        }
    }
}

func BenchmarkBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("hello") // O(n) amortized
        }
        _ = sb.String()
    }
}
```

```bash
go test -bench=. -benchmem ./...
# BenchmarkConcat-8      10000    150000 ns/op    53000 B/op    99 allocs/op
# BenchmarkBuilder-8    200000      8000 ns/op     2048 B/op     8 allocs/op
```

`-benchmem` shows allocations per operation — this is often more important than raw speed. Every allocation = GC pressure.

### 5.4 Profile in Production

```go
import _ "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil) // pprof endpoint
    }()
    // ... your server ...
}
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile (current allocations)
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine dump (find leaks)
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

---

## 6. Common Pitfalls (Things That Bite in Production)

### 6.1 Nil Map Write Panic

```go
var m map[string]int
m["key"] = 1 // PANIC: assignment to entry in nil map
```

**Fix:** Always initialize with `make()` or a literal.

### 6.2 Slice Append Gotcha

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3] // [2, 3] — shares backing array, cap=4

b = append(b, 99)
fmt.Println(a) // [1, 2, 3, 99, 5] — a[3] was overwritten!
```

**Fix:** Use full slice expression to limit capacity: `b := a[1:3:3]`

### 6.3 Loop Variable Capture (Pre-Go 1.22)

```go
// Go < 1.22: all goroutines share one `i` variable
for i := 0; i < 5; i++ {
    go func() { fmt.Println(i) }() // likely prints 5,5,5,5,5
}

// Fixed in Go 1.22+ (each iteration gets its own i)
// But if targeting older versions, pass as parameter:
for i := 0; i < 5; i++ {
    go func(n int) { fmt.Println(n) }(i) // 0,1,2,3,4
}
```

### 6.4 Deferred Calls in Loops

```go
// BAD: files don't close until the function returns (not per iteration)
func processFiles(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close() // accumulates! All defers run at function exit
        // 10000 files = 10000 open file descriptors
    }
    return nil
}

// GOOD: close immediately or use a helper function
func processFiles(paths []string) error {
    for _, path := range paths {
        if err := processOne(path); err != nil {
            return err
        }
    }
    return nil
}

func processOne(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // closes when processOne returns (each iteration)
    // ... process file ...
    return nil
}
```

### 6.5 Nil Interface vs Nil Value (The Billion Dollar Bug)

```go
func getError() error {
    var err *MyError = nil
    return err // NOT nil! Interface{type: *MyError, data: nil}
}

err := getError()
if err != nil {
    fmt.Println("error!") // THIS RUNS — err is not nil
}
```

**Fix:** Return untyped `nil` explicitly:

```go
func getError() error {
    var err *MyError = nil
    if err == nil {
        return nil // return the untyped nil interface
    }
    return err
}
```

### 6.6 time.After Leaks in Loops

```go
// BAD: each time.After creates a timer that isn't GC'd until it fires
for {
    select {
    case msg := <-ch:
        handle(msg)
    case <-time.After(5 * time.Second): // LEAK: new timer every iteration
        return
    }
}

// GOOD: reuse the timer
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()
for {
    select {
    case msg := <-ch:
        if !timer.Stop() {
            <-timer.C
        }
        timer.Reset(5 * time.Second)
        handle(msg)
    case <-timer.C:
        return
    }
}
```

### 6.7 HTTP Response Body Must Be Closed and Drained

```go
// BAD: connection stays open, can't be reused by the transport pool
resp, err := http.Get(url)
if err != nil {
    return err
}
body, _ := io.ReadAll(resp.Body)
// forgot to close!

// GOOD: close and drain to allow connection reuse
resp, err := http.Get(url)
if err != nil {
    return err
}
defer func() {
    io.Copy(io.Discard, resp.Body) // drain remaining bytes
    resp.Body.Close()
}()
```

If you don't drain and close, the TCP connection can't be returned to the pool and you'll exhaust connections under load.

### 6.8 json.Decoder vs json.Unmarshal

```go
// json.Unmarshal: reads entire body into memory first
body, _ := io.ReadAll(resp.Body) // allocates entire body as []byte
json.Unmarshal(body, &result)

// json.Decoder: streams — no full body in memory
json.NewDecoder(resp.Body).Decode(&result)
```

Use `Decoder` for HTTP responses and large files. Use `Unmarshal` when you already have `[]byte` in memory.

---

## 7. Production Patterns

### 7.1 Graceful Shutdown

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // Give in-flight requests 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("shutdown error: %v", err)
    }
    log.Println("server stopped gracefully")
}
```

### 7.2 Structured Logging (slog, Go 1.21+)

```go
// BAD: unstructured — impossible to parse at scale
log.Printf("user %s logged in from %s", userID, ip)

// GOOD: structured — queryable in log aggregators (Datadog, Splunk, etc.)
slog.Info("user login",
    "user_id", userID,
    "ip", ip,
    "latency_ms", latency.Milliseconds(),
)
// output: {"time":"...","level":"INFO","msg":"user login","user_id":"abc","ip":"1.2.3.4","latency_ms":42}
```

### 7.3 Circuit Breaker Pattern

```go
type CircuitBreaker struct {
    mu          sync.Mutex
    failures    int
    threshold   int
    lastFailure time.Time
    cooldown    time.Duration
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    if cb.failures >= cb.threshold {
        if time.Since(cb.lastFailure) < cb.cooldown {
            cb.mu.Unlock()
            return ErrCircuitOpen // fail fast — don't even try
        }
        cb.failures = 0 // cooldown passed, try again
    }
    cb.mu.Unlock()

    err := fn()
    if err != nil {
        cb.mu.Lock()
        cb.failures++
        cb.lastFailure = time.Now()
        cb.mu.Unlock()
    }
    return err
}
```

---

## Quick Reference: Decision Table

| Situation | Do This | Not This |
|---|---|---|
| Small struct (<64B) | Pass by value | Pointer everywhere |
| Large struct or needs mutation | Pass by pointer | Copy 500-byte struct on every call |
| Known collection size | `make([]T, 0, n)` | `var s []T` then append |
| Protecting shared data | `sync.RWMutex` | Unbuffered channel as lock |
| Transferring data between goroutines | Channel | Shared variable + mutex |
| Launching N goroutines | Bounded worker pool | Unbounded `go func()` |
| String concatenation in loop | `strings.Builder` | `s += "text"` |
| Timeout on operation | `context.WithTimeout` | `time.Sleep` + manual check |
| Error from function | `fmt.Errorf("context: %w", err)` | Return bare `err` |
| Checking error type | `errors.Is()` / `errors.As()` | String matching on `err.Error()` |
| Hot-path allocations | `sync.Pool` or stack allocation | `new(T)` every call |
| Configuration with many options | Functional options pattern | 15-field config struct |
