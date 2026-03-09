# Concurrency in Rust

**Related:** [ownership and borrowing](ownership_and_borrowing.md) | [traits and generics (Send, Sync)](traits_and_generics.md) | [smart pointers (Arc, Mutex)](smart_pointers.md) | [lifetimes](lifetimes_in_rust.md) | [best practices](rust_best_practices.md)

## Fearless Concurrency

Rust's ownership system prevents data races at **compile time**. This is unique among systems languages.

```
Data race requires ALL three:
1. Two or more threads accessing the same data
2. At least one is writing
3. No synchronization

Rust's type system makes (1) + (2) impossible without synchronization,
because you need &mut T for writing, and &mut T is exclusive.
```

In Go, you discover data races at runtime with `-race`. In C++, you discover them in production at 3 AM. In Rust, you discover them at compile time.

---

## std::thread -- OS Threads

### Spawning Threads

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        println!("hello from thread");
    });

    handle.join().unwrap();  // wait for thread to finish
}
```

### Moving Ownership Into Threads

```rust
let data = vec![1, 2, 3];

// COMPILE ERROR: closure may outlive `data`
// let handle = thread::spawn(|| {
//     println!("{:?}", data);
// });

// FIX: move ownership into the thread
let handle = thread::spawn(move || {
    println!("{:?}", data);  // data is now owned by this thread
});

// println!("{:?}", data);  // COMPILE ERROR: data was moved
handle.join().unwrap();
```

**How to think about it:** `move` transfers ownership of captured variables into the closure. The compiler ensures the thread owns its data -- no shared mutable state, no data race.

**What's happening in memory:**

```
Main thread stack:           Spawned thread stack:
data: [MOVED]                data: ┌──────────┐    Heap:
                                   │ ptr ──────┼──> [1, 2, 3]
                                   │ len: 3    │
                                   │ cap: 3    │
                                   └──────────┘
```

---

## Message Passing with Channels

Rust's channels implement **multi-producer, single-consumer (mpsc)** communication.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();  // tx: Sender, rx: Receiver

    thread::spawn(move || {
        tx.send(String::from("hello")).unwrap();
        // tx is moved into this thread -- ownership of the String
        // is TRANSFERRED through the channel
    });

    let msg = rx.recv().unwrap();  // blocks until message arrives
    println!("{}", msg);
}
```

### Channel Ownership Transfer

```rust
let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    let data = vec![1, 2, 3];
    tx.send(data).unwrap();
    // println!("{:?}", data);  // COMPILE ERROR: data was moved into channel
});

let received = rx.recv().unwrap();
println!("{:?}", received);  // [1, 2, 3] -- ownership transferred
```

**How to think about it:** The channel takes ownership of the sent value. After `send(data)`, the sender can't access `data` anymore. This prevents the sender from modifying data while the receiver reads it -- enforced at compile time.

### Multiple Producers

```rust
let (tx, rx) = mpsc::channel();

for i in 0..5 {
    let tx_clone = tx.clone();  // clone the sender for each thread
    thread::spawn(move || {
        tx_clone.send(format!("thread {}", i)).unwrap();
    });
}
drop(tx);  // drop original sender so rx.iter() terminates

for msg in rx {   // iterates until all senders are dropped
    println!("{}", msg);
}
```

### Bounded Channels (sync_channel)

```rust
let (tx, rx) = mpsc::sync_channel(3);  // buffer capacity 3

// send blocks when buffer is full (backpressure)
tx.send(1).unwrap();
tx.send(2).unwrap();
tx.send(3).unwrap();
// tx.send(4).unwrap();  // BLOCKS until receiver consumes
```

---

## Shared State with Mutex<T>

When channels are overkill, use `Mutex<T>` for shared mutable state.

```rust
use std::sync::Mutex;

let m = Mutex::new(5);

{
    let mut num = m.lock().unwrap();  // acquire lock, get MutexGuard
    *num = 6;
}   // MutexGuard is dropped here -- lock is released automatically

println!("{:?}", m);  // Mutex { data: 6 }
```

**What's happening:**
- `lock()` returns a `MutexGuard<T>` which implements `Deref` to `T`
- The `MutexGuard` holds the lock as long as it lives
- When `MutexGuard` is dropped (goes out of scope), the lock is released
- No way to forget to unlock -- the ownership system guarantees it

### Mutex<T> Across Threads with Arc<T>

`Mutex<T>` is single-owner. To share across threads, wrap in `Arc<T>` (atomic reference counting).

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        let counter = Arc::clone(&counter);  // clone the Arc (increments ref count)
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });  // MutexGuard dropped, lock released. Arc ref count decremented
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());  // 10
}
```

**Memory layout:**

```
Thread 1 stack:              Thread 2 stack:
arc1: ┌────────┐            arc2: ┌────────┐
      │ ptr ───┼──┐              │ ptr ───┼──┐
      └────────┘  │              └────────┘  │
                  └──────┬───────────────────┘
                         v
            Heap: ┌──────────────────┐
                  │ strong_count: 2  │  (atomic)
                  │ weak_count: 0    │  (atomic)
                  │ Mutex<i32>       │
                  │  ┌──────────┐   │
                  │  │ data: 0  │   │
                  │  │ locked:F │   │
                  │  └──────────┘   │
                  └──────────────────┘
```

---

## Send and Sync Traits

Two marker traits that control what can cross thread boundaries:

```rust
// Send: a type can be TRANSFERRED to another thread (ownership moved)
// Sync: a type can be SHARED between threads via references (&T)

// Formal definition:
// T is Send  => T can be moved to another thread
// T is Sync  => &T is Send (safe to share references across threads)
```

**Auto-implemented:** The compiler automatically implements `Send` and `Sync` for types where it's safe. You almost never implement them manually.

| Type | Send | Sync | Why |
|---|---|---|---|
| `i32`, `bool`, `String` | Yes | Yes | No interior mutability, no raw pointers |
| `Vec<T>` (where T: Send) | Yes | Yes | Owns its data |
| `Rc<T>` | **No** | **No** | Non-atomic reference count -- not thread-safe |
| `Arc<T>` | Yes | Yes | Atomic reference count |
| `Mutex<T>` | Yes | Yes | Interior mutability with locking |
| `Cell<T>`, `RefCell<T>` | Yes | **No** | Interior mutability without locking |
| `*mut T` (raw pointer) | **No** | **No** | No safety guarantees |

```rust
// This won't compile because Rc is not Send
use std::rc::Rc;
let rc = Rc::new(5);
thread::spawn(move || {     // COMPILE ERROR: Rc<i32> cannot be sent between threads
    println!("{}", rc);
});

// Fix: use Arc instead
use std::sync::Arc;
let arc = Arc::new(5);
thread::spawn(move || {     // OK: Arc<i32> is Send
    println!("{}", arc);
});
```

**How to think about it:** `Send` and `Sync` are the type system's way of preventing data races. If a type isn't thread-safe, the compiler won't let you share it across threads. This is checked at compile time -- zero runtime cost.

---

## RwLock<T> -- Reader-Writer Lock

Like Go's `sync.RWMutex`. Multiple readers OR one writer.

```rust
use std::sync::RwLock;

let lock = RwLock::new(vec![1, 2, 3]);

// Multiple readers (concurrent)
{
    let r1 = lock.read().unwrap();
    let r2 = lock.read().unwrap();  // OK: multiple readers
    println!("{:?} {:?}", r1, r2);
}

// One writer (exclusive)
{
    let mut w = lock.write().unwrap();
    w.push(4);
}
```

---

## Rayon -- Data Parallelism

Rayon provides work-stealing parallelism with a one-line API change.

```rust
use rayon::prelude::*;

// Sequential
let sum: i32 = (0..1_000_000).map(|x| x * x).sum();

// Parallel (just change iter() to par_iter())
let sum: i32 = (0..1_000_000).into_par_iter().map(|x| x * x).sum();
```

**How it works:** Rayon uses a work-stealing thread pool. Tasks are split recursively (like fork-join) and distributed across CPU cores. The ownership system guarantees no data races.

```rust
// Parallel sort
let mut data = vec![5, 3, 1, 4, 2];
data.par_sort();  // parallel merge sort

// Parallel map + collect
let results: Vec<_> = inputs.par_iter()
    .map(|input| expensive_computation(input))
    .collect();
```

---

## Async/Await and Futures

### What Is a Future?

A `Future` represents a value that may not be ready yet. It's a state machine that gets polled to completion.

```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),     // value is available
    Pending,      // not ready yet, will wake via Waker
}
```

### async/await Syntax

```rust
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}

#[tokio::main]
async fn main() {
    let result = fetch_data("https://httpbin.org/get").await;
    println!("{:?}", result);
}
```

**How `async` works under the hood:** The compiler transforms `async fn` into a state machine (an enum implementing `Future`). Each `.await` point becomes a state transition. The state machine is **stack-allocated** (no heap allocation for the future itself unless boxed).

```
async fn example() {       Compiler generates:
    let a = step1().await;    enum ExampleFuture {
    let b = step2(a).await;       State0 { ... },   // before first await
    a + b                         State1 { a, ... }, // between awaits
}                                 State2 { a, b },   // after second await
                              }
                              impl Future for ExampleFuture { ... }
```

### Tokio Runtime

Rust has no built-in async runtime. Tokio is the de facto standard.

```rust
use tokio;

#[tokio::main]
async fn main() {
    // Spawn concurrent tasks
    let handle1 = tokio::spawn(async {
        fetch("url1").await
    });
    let handle2 = tokio::spawn(async {
        fetch("url2").await
    });

    let (r1, r2) = tokio::join!(handle1, handle2);
}
```

**Tokio's architecture:**
- Multi-threaded work-stealing scheduler (like Go's M:N scheduler)
- Epoll/kqueue-based I/O driver (like Go's netpoller)
- Timer wheel for timeouts
- Each `tokio::spawn` creates a task (~allocation of the future state machine)

### Pin and Why It Exists

Self-referential structs (which async state machines often are) cannot be moved in memory. `Pin` guarantees the value stays at its memory address.

```rust
// Simplified: Pin prevents moves after the value is pinned
let future = async {
    let data = vec![1, 2, 3];
    let reference = &data;     // reference points to data's location
    some_await_point().await;   // state machine is suspended here
    println!("{:?}", reference); // reference must still be valid
};
// If the state machine were moved in memory, `reference` would dangle
// Pin prevents this by guaranteeing the memory location is stable
```

---

## Practical Examples / LeetCode Patterns

### Parallel Web Scraper -- Bounded Concurrency

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn fetch_all(urls: Vec<String>, max_concurrent: usize) -> Vec<String> {
    let results = Arc::new(Mutex::new(Vec::with_capacity(urls.len())));
    let semaphore = Arc::new(Mutex::new(max_concurrent));

    let mut handles = vec![];
    for url in urls {
        let results = Arc::clone(&results);
        let handle = thread::spawn(move || {
            let body = fetch(&url);  // blocking fetch
            results.lock().unwrap().push(body);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    Arc::try_unwrap(results).unwrap().into_inner().unwrap()
}
```

### Producer-Consumer with Channels

```rust
use std::sync::mpsc;
use std::thread;

fn producer_consumer() {
    let (tx, rx) = mpsc::sync_channel(100);  // bounded buffer

    // Producer
    let producer = thread::spawn(move || {
        for i in 0..1000 {
            tx.send(i).unwrap();  // blocks when buffer full (backpressure)
        }
    });

    // Consumer
    let consumer = thread::spawn(move || {
        let mut sum = 0i64;
        for val in rx {    // iterates until sender is dropped
            sum += val;
        }
        sum
    });

    producer.join().unwrap();
    let result = consumer.join().unwrap();
    println!("Sum: {}", result);
}
```

### Async HTTP Server Pattern

```rust
use tokio::net::TcpListener;
use std::sync::Arc;

struct AppState {
    db: DatabasePool,
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db: connect_db().await,
    });

    let listener = TcpListener::bind("0.0.0.0:8080").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        let state = Arc::clone(&state);

        tokio::spawn(async move {
            handle_connection(socket, state).await;
        });
    }
}
```

**Concurrency insight:** Each connection gets its own task via `tokio::spawn`. `Arc` allows shared read-only access to app state. The state is `Send + Sync` (required for `tokio::spawn`), which the compiler verifies at compile time.

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Fearless concurrency | Ownership system prevents data races at compile time |
| `move` closures | Transfer ownership into threads. Compiler enforces no shared mutable state |
| Channels (mpsc) | Ownership transfers through the channel. Sender loses access after send |
| `Mutex<T>` | Lock guard auto-releases via Drop. Wrap in `Arc` for multi-thread |
| `Send` | Type can be moved across thread boundaries. Auto-implemented |
| `Sync` | `&T` can be shared across threads. Auto-implemented |
| `Rc` vs `Arc` | `Rc` is not thread-safe (not `Send`). Use `Arc` for multi-threaded code |
| Rayon | Drop-in data parallelism. Change `iter()` to `par_iter()` |
| async/await | Compiler generates state machines. Zero-cost. Needs runtime (tokio) |
| Pin | Prevents moving self-referential async state machines in memory |
