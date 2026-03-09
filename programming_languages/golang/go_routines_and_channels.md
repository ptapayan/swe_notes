# Goroutines and Channels in Go

**Related:** [Go as a language](golang_as_a_language.md) | [functions (closures, goroutine trap)](function_in_golang.md) | [for loops (range over channel)](for_loops_in_golang.md) | [make function (channels)](make_function_in_golang.md)

## The Concurrency Model: CSP

Go's concurrency is based on **Communicating Sequential Processes (CSP)**. The core philosophy:

> "Don't communicate by sharing memory; share memory by communicating." — Go Proverb

Instead of threads + locks + shared memory, Go uses **goroutines** (lightweight execution units) and **channels** (typed pipes for communication).

---

## Goroutines

### What Is a Goroutine?

A goroutine is a lightweight, user-space thread managed by the Go runtime — NOT an OS thread.

```go
func sayHello(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    go sayHello("Alice") // launches a goroutine
    go sayHello("Bob")   // launches another goroutine

    time.Sleep(100 * time.Millisecond) // BAD: don't do this in production
}
```

### How Goroutines Work Under the Hood (M:N Scheduling)

Go uses an **M:N scheduler** — M goroutines are multiplexed onto N OS threads.

**Three key entities:**
- **G** (Goroutine): The unit of work. Contains the stack, instruction pointer, and other state
- **M** (Machine): An OS thread. The thing that actually runs code on the CPU
- **P** (Processor): A logical processor. Holds a local run queue of goroutines. The number of P's = `GOMAXPROCS` (defaults to number of CPU cores)

**Scheduling flow:**
```
G (goroutine) → P (processor's run queue) → M (OS thread) → CPU core
```

**Key facts:**
- A goroutine starts with a stack of only **2KB-8KB** (vs 1-8MB for OS threads)
- The stack grows dynamically by copying to a larger allocation (contiguous stack)
- Context switching between goroutines is ~100-200ns (vs ~1-10μs for OS threads)
- You can easily run **millions** of goroutines; you'd run out of memory with thousands of OS threads

### What Happens When a Goroutine Blocks?

When a goroutine makes a **blocking syscall** (file I/O, network with CGO, etc.):
1. The M (OS thread) is detached from the P
2. A new M is created (or reused) to keep the P busy
3. Other goroutines on that P continue executing
4. When the syscall completes, the goroutine is put back on a run queue

For **network I/O**, Go uses the **netpoller** (epoll/kqueue) so goroutines park without blocking an OS thread at all.

---

## Channels

### What Is a Channel?

A channel is a typed conduit for sending and receiving values between goroutines. Think of it as a thread-safe queue.

```go
func main() {
    ch := make(chan string) // unbuffered channel

    go func() {
        ch <- "hello" // send a value into the channel
    }()

    msg := <-ch // receive a value from the channel (blocks until a value is available)
    fmt.Println(msg) // "hello"
}
```

### Channel Internal Structure

A channel (`hchan` struct) in the runtime contains:
- A **circular buffer** (for buffered channels)
- A **mutex** for synchronization
- **Send queue** (linked list of goroutines waiting to send)
- **Receive queue** (linked list of goroutines waiting to receive)
- Element type info and buffer size

### Unbuffered Channels (Synchronous)

```go
ch := make(chan int) // unbuffered: capacity 0
```

**How it works:**
- `ch <- value` **blocks** until another goroutine does `<-ch`
- `<-ch` **blocks** until another goroutine does `ch <- value`
- This creates a **synchronization point** — both goroutines must be ready

```go
func main() {
    ch := make(chan int)

    go func() {
        result := heavyComputation()
        ch <- result // blocks until main() is ready to receive
    }()

    // ... do other work ...

    result := <-ch // blocks until the goroutine sends
    fmt.Println(result)
}
```

**Think of it as:** A handshake. The sender and receiver must meet at the same time.

### Buffered Channels (Asynchronous up to capacity)

```go
ch := make(chan int, 3) // buffered: capacity 3
```

**How it works:**
- `ch <- value` only blocks when the buffer is **full**
- `<-ch` only blocks when the buffer is **empty**
- Acts like a queue with a fixed size

```go
func main() {
    ch := make(chan string, 2)

    ch <- "first"  // doesn't block (buffer has space)
    ch <- "second" // doesn't block (buffer has space)
    // ch <- "third" // WOULD BLOCK — buffer is full, no receiver

    fmt.Println(<-ch) // "first"  (FIFO order)
    fmt.Println(<-ch) // "second"
}
```

**When to use buffered vs unbuffered:**
- **Unbuffered:** When you need synchronization (guarantee the other side processed the value)
- **Buffered:** When sender and receiver run at different speeds, to decouple them

---

## Channel Direction (Restricting Send/Receive)

You can restrict a channel to send-only or receive-only in function signatures. This is a **compile-time safety** feature.

```go
// producer can only SEND to ch
func producer(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}

// consumer can only RECEIVE from ch
func consumer(ch <-chan int) {
    for val := range ch {
        fmt.Println(val)
    }
}

func main() {
    ch := make(chan int, 5)
    go producer(ch)
    consumer(ch)
}
```

A bidirectional `chan int` is implicitly convertible to `chan<- int` or `<-chan int`, but not the reverse.

---

## Closing Channels

```go
close(ch)
```

**Rules:**
- Only the **sender** should close a channel, never the receiver
- Sending on a closed channel causes a **panic**
- Receiving from a closed channel returns the **zero value** immediately
- You can check if a channel is closed: `val, ok := <-ch` (ok is false if closed and empty)

```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch)

    // Range automatically stops when channel is closed and drained
    for v := range ch {
        fmt.Println(v) // 1, 2, 3
    }

    // Reading from closed empty channel
    val, ok := <-ch
    fmt.Println(val, ok) // 0 false
}
```

---

## Select Statement

`select` lets a goroutine wait on multiple channel operations simultaneously. It's like a `switch` for channels.

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "from channel 1"
    }()

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "from channel 2"
    }()

    // Waits for whichever channel is ready first
    select {
    case msg := <-ch1:
        fmt.Println(msg)
    case msg := <-ch2:
        fmt.Println(msg)
    }
}
```

### Select with Default (Non-Blocking)

```go
select {
case msg := <-ch:
    fmt.Println(msg)
default:
    fmt.Println("no message ready, moving on")
}
```

### Select with Timeout

```go
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(3 * time.Second):
    fmt.Println("timed out waiting for result")
}
```

### Select in a Loop (Common Pattern)

```go
func worker(jobs <-chan int, done <-chan struct{}) {
    for {
        select {
        case job := <-jobs:
            fmt.Printf("processing job %d\n", job)
        case <-done:
            fmt.Println("shutting down")
            return
        }
    }
}
```

**How select works internally:** If multiple cases are ready, one is chosen at **random** (pseudorandom, to prevent starvation). If no cases are ready and there's no `default`, the goroutine parks.

---

## Common Concurrency Patterns

### Fan-Out / Fan-In

Fan-out: Multiple goroutines reading from the same channel.
Fan-in: Multiple channels funneled into one.

```go
// Fan-out: distribute work
func fanOut(input <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = process(input)
    }
    return channels
}

func process(input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range input {
            out <- n * n // some processing
        }
    }()
    return out
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                merged <- val
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

### Worker Pool

```go
func workerPool(numWorkers int, jobs <-chan int, results chan<- int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                fmt.Printf("worker %d processing job %d\n", id, job)
                results <- job * 2
            }
        }(i)
    }
    wg.Wait()
    close(results)
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    go workerPool(5, jobs, results)

    // Send jobs
    for i := 0; i < 20; i++ {
        jobs <- i
    }
    close(jobs)

    // Collect results
    for r := range results {
        fmt.Println(r)
    }
}
```

---

## sync.WaitGroup

Used to wait for a collection of goroutines to finish.

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done() // decrements counter when goroutine finishes
            fmt.Printf("goroutine %d done\n", id)
        }(i)
    }

    wg.Wait() // blocks until counter reaches 0
    fmt.Println("all goroutines finished")
}
```

**Internal:** WaitGroup uses atomic operations on a counter. `Add(n)` increments, `Done()` decrements (same as `Add(-1)`), `Wait()` blocks until zero.

**Critical rules:**
- Call `wg.Add()` BEFORE launching the goroutine, not inside it
- Never copy a WaitGroup after first use (pass by pointer)

---

## sync.Mutex and sync.RWMutex

When you DO need shared memory (sometimes channels are overkill):

```go
type SafeCounter struct {
    mu sync.Mutex
    count map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count[key]++
}

func (c *SafeCounter) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count[key]
}
```

**RWMutex** allows multiple concurrent readers OR one writer:

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Read(key string) string {
    c.mu.RLock()         // multiple goroutines can RLock simultaneously
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *Cache) Write(key, val string) {
    c.mu.Lock()          // exclusive access
    defer c.mu.Unlock()
    c.data[key] = val
}
```

---

## Context for Cancellation and Timeouts

`context.Context` is the standard way to propagate cancellation, deadlines, and request-scoped values across goroutines.

```go
func longRunningTask(ctx context.Context) error {
    for i := 0; i < 100; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err() // context.Canceled or context.DeadlineExceeded
        default:
            // do work
            time.Sleep(50 * time.Millisecond)
        }
    }
    return nil
}

func main() {
    // Cancel after 1 second
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel() // always defer cancel to release resources

    err := longRunningTask(ctx)
    if err != nil {
        fmt.Println("task stopped:", err) // "task stopped: context deadline exceeded"
    }
}
```

---

## Deadlock Detection

The Go runtime detects when ALL goroutines are blocked and panics:

```go
func main() {
    ch := make(chan int)
    ch <- 1 // DEADLOCK: no goroutine to receive
}
// fatal error: all goroutines are asleep - Loss of deadlock!
```

The runtime only catches **global** deadlock (every goroutine stuck). Partial deadlocks (a subset of goroutines stuck) are NOT detected — you need to find those yourself.

---

## Race Detector

Go has a built-in race detector. Use it during development:

```bash
go run -race main.go
go test -race ./...
```

It instruments memory accesses at compile time and detects concurrent unsynchronized access at runtime. Always run your tests with `-race`.

---

---

## Practical Examples

### Parallel Web Scraper — Bounded Concurrency

```go
func fetchAll(urls []string, maxConcurrent int) []string {
    results := make([]string, len(urls))
    sem := make(chan struct{}, maxConcurrent) // semaphore
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(i int, url string) {
            defer wg.Done()
            sem <- struct{}{}        // acquire (blocks if maxConcurrent goroutines active)
            defer func() { <-sem }() // release

            resp, err := http.Get(url)
            if err != nil {
                results[i] = "error"
                return
            }
            defer resp.Body.Close()
            body, _ := io.ReadAll(resp.Body)
            results[i] = string(body)
        }(i, url)
    }

    wg.Wait()
    return results
}
```

**Patterns at play:** Buffered channel as semaphore, WaitGroup for completion, goroutine-per-task with bounded concurrency. Writing to `results[i]` is safe because each goroutine writes to a different index.

### Pipeline Pattern — Processing Stages

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func main() {
    // Pipeline: generate → square → print
    for v := range square(square(generate(2, 3, 4))) {
        fmt.Println(v) // 16, 81, 256 (squared twice)
    }
}
```

**Channel direction types** (`<-chan`, `chan<-`) enforce at compile time that each stage only reads or writes. Closing the channel propagates the "done" signal through the pipeline via `range`.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Goroutines | 2-8KB stack, M:N scheduled, ~100ns context switch |
| Unbuffered channel | Synchronous handshake — both sides must be ready |
| Buffered channel | Async queue — blocks only when full/empty |
| Select | Wait on multiple channels, random choice if multiple ready |
| WaitGroup | Coordinate goroutine completion. Add before go, Done via defer |
| Mutex | For shared memory when channels are overkill |
| Context | Cancellation + timeout propagation across goroutines |
| Race detector | Always use `-race` in tests |
