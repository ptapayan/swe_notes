# Concurrency in Python

**Related:** [Python as a language](python_as_a_language.md) | [functions](functions_in_python.md) | [iterators & generators](iterators_and_generators.md) | [best practices](python_best_practices.md)

---

## The GIL — Deep Dive

The Global Interpreter Lock (GIL) is a **mutex in CPython** that allows only ONE thread to execute Python bytecode at any time, even on multi-core machines.

### What the GIL Protects

```
Without GIL (hypothetical):
Thread 1: LOAD ob_refcnt(42)   → reads 1
Thread 2: LOAD ob_refcnt(42)   → reads 1
Thread 1: INCREF → writes 2
Thread 2: INCREF → writes 2    ← should be 3! Race condition.
                                  Object freed prematurely → crash.

With GIL:
Thread 1: [acquire GIL] LOAD → INCREF(2) → DECREF → ... [release GIL]
Thread 2: ----waiting---- [acquire GIL] LOAD → INCREF → ... [release]
```

The GIL protects:
1. **Reference counts** on every PyObject (the `ob_refcnt` field)
2. **CPython's internal data structures** (memory allocator, object lists for GC)
3. **Non-thread-safe C extensions** that assume single-threaded access

### GIL Switching Mechanism

Since Python 3.2, the GIL uses a **time-based switching** mechanism (not instruction-count based):

```python
import sys
sys.getswitchinterval()   # 0.005 (5ms default)
sys.setswitchinterval(0.001)  # set to 1ms (for more responsive threading)
```

Every 5ms (by default), the running thread is asked to release the GIL. A waiting thread then acquires it. This is cooperative — the running thread checks a flag (`eval_breaker`) at certain bytecode boundaries.

### When the GIL is Released

```
Operation              GIL Released?    Why
─────────────────────────────────────────────
Pure Python compute    NO               bytecodes need GIL
I/O (file, network)   YES              C code releases before blocking syscall
time.sleep()           YES              releases before sleeping
C extension (NumPy)    YES              well-written C extensions release GIL
regex matching         YES              re module releases during matching
hashlib computation    YES              releases during hash computation
```

---

## threading — Limited by GIL for CPU

Python threads are **real OS threads** (pthreads on Linux, Windows threads on Windows). But the GIL means only one executes Python bytecode at a time.

### Good For: I/O-Bound Work

```python
import threading
import time

def download(url):
    """I/O bound — GIL is released during network wait."""
    import urllib.request
    data = urllib.request.urlopen(url).read()
    return len(data)

urls = ["https://example.com"] * 10

# Sequential: 10 * 0.5s = ~5 seconds
# Threaded: ~0.5 seconds (all waiting concurrently)

threads = []
for url in urls:
    t = threading.Thread(target=download, args=(url,))
    t.start()
    threads.append(t)

for t in threads:
    t.join()
```

### Bad For: CPU-Bound Work

```python
def compute(n):
    """CPU bound — GIL prevents parallel execution."""
    total = 0
    for i in range(n):
        total += i * i
    return total

# With threads: SLOWER than single-threaded (GIL contention + switching overhead)
# With multiprocessing: actually parallel
```

### Thread Synchronization

```python
import threading

lock = threading.Lock()
counter = 0

def increment():
    global counter
    for _ in range(100000):
        with lock:          # context manager — acquires and releases
            counter += 1    # without lock: counter would be < 200000

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()
print(counter)  # 200000 (correct with lock)
```

**Why a lock is needed even with the GIL:** `counter += 1` is NOT atomic. It compiles to `LOAD_GLOBAL` + `LOAD_CONST` + `BINARY_ADD` + `STORE_GLOBAL`. The GIL can switch threads between any of these bytecodes.

---

## multiprocessing — True Parallelism

`multiprocessing` spawns **separate Python interpreter processes**, each with its own GIL. True parallel execution on multiple cores.

```python
from multiprocessing import Pool
import os

def compute(n):
    print(f"PID: {os.getpid()}")  # different PID per process
    return sum(i * i for i in range(n))

if __name__ == '__main__':
    with Pool(processes=4) as pool:
        results = pool.map(compute, [10**7] * 4)
    print(sum(results))
```

```
multiprocessing architecture:

Main Process (PID 1000):
┌─────────────────────┐
│ Python interpreter   │
│ GIL (its own)       │
│ Pool manager        │
└────────┬────────────┘
         │ fork/spawn
    ┌────┴────┬────────┬────────┐
    v         v        v        v
 PID 1001  PID 1002  PID 1003  PID 1004
 ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
 │interp │ │interp │ │interp │ │interp │
 │GIL    │ │GIL    │ │GIL    │ │GIL    │
 │compute│ │compute│ │compute│ │compute│
 └───────┘ └───────┘ └───────┘ └───────┘
 Each process has its OWN interpreter and GIL.
 True parallel execution on multiple cores.
```

### IPC Overhead

```python
# Data must be serialized (pickled) to send between processes.
# This is expensive for large objects.

# Shared memory (Python 3.8+) avoids pickling:
from multiprocessing import shared_memory

shm = shared_memory.SharedMemory(create=True, size=1024)
# Both processes can access shm.buf directly — no serialization

# For numpy arrays:
import numpy as np
from multiprocessing import shared_memory

a = np.array([1, 2, 3, 4, 5])
shm = shared_memory.SharedMemory(create=True, size=a.nbytes)
shared_array = np.ndarray(a.shape, dtype=a.dtype, buffer=shm.buf)
shared_array[:] = a[:]  # copy data to shared memory
```

---

## asyncio — Single-Threaded Concurrency

`asyncio` provides **cooperative multitasking** in a single thread. No GIL contention, no thread overhead. Perfect for I/O-bound workloads with many concurrent connections.

### Event Loop Architecture

```
asyncio event loop (single thread):

┌─────────────────────────────────────────┐
│  Event Loop                             │
│  ┌─────────────────────────────────┐    │
│  │ Ready queue: [coro1, coro3]     │    │
│  │ I/O polling: [socket_fd: coro2] │    │
│  │ Timers: [(t+5s, coro4)]        │    │
│  └─────────────────────────────────┘    │
│                                         │
│  Loop iteration:                        │
│  1. Run all ready coroutines            │
│  2. Poll for I/O (epoll/kqueue)         │
│  3. Check timers                        │
│  4. Repeat                              │
└─────────────────────────────────────────┘

Coroutine 1: fetch(url_1) ──await──> [suspended, waiting for I/O]
Coroutine 2: fetch(url_2) ──await──> [suspended, waiting for I/O]
Coroutine 3: process(data) ──running──> ...

Only ONE coroutine runs at a time. They VOLUNTARILY yield
at `await` points. No preemption, no race conditions on
Python data structures.
```

### async/await Basics

```python
import asyncio
import aiohttp

async def fetch(session, url):
    """Coroutine — suspends at await, resumes when I/O completes."""
    async with session.get(url) as response:
        return await response.text()

async def main():
    async with aiohttp.ClientSession() as session:
        # Launch 100 requests concurrently (NOT in parallel):
        tasks = [fetch(session, f"https://example.com/{i}") for i in range(100)]
        results = await asyncio.gather(*tasks)
        # All 100 requests happen concurrently in ONE thread.
        # Total time ≈ slowest single request, not 100x.

asyncio.run(main())
```

### Coroutine Internals

A coroutine is a **generator-like state machine**. When you `await`, the coroutine suspends and returns control to the event loop. The event loop resumes it when the awaited operation completes.

```python
async def example():
    print("start")
    await asyncio.sleep(1)   # suspends here, event loop runs other tasks
    print("end")             # resumes here after 1 second

# Under the hood, this is similar to:
# def example():
#     print("start")
#     yield SLEEP_EVENT(1)   # yield control to event loop
#     print("end")
```

### asyncio Primitives

```python
# Semaphore — limit concurrent operations
sem = asyncio.Semaphore(10)  # max 10 concurrent
async def limited_fetch(url):
    async with sem:
        return await fetch(url)

# Queue — producer/consumer
queue = asyncio.Queue(maxsize=100)
async def producer():
    await queue.put(item)
async def consumer():
    item = await queue.get()

# Lock — mutual exclusion (rarely needed in async — single threaded)
lock = asyncio.Lock()
async def critical_section():
    async with lock:
        await do_work()
```

---

## concurrent.futures — High-Level API

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# Thread pool (I/O-bound):
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(download, url) for url in urls]
    results = [f.result() for f in futures]

# Process pool (CPU-bound):
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(compute, data) for data in datasets]
    results = [f.result() for f in futures]

# map() shorthand:
with ProcessPoolExecutor() as executor:
    results = list(executor.map(compute, datasets))
```

**Why concurrent.futures over raw threading/multiprocessing:**
- Uniform API for both threads and processes
- Future objects with `.result()`, `.exception()`, `.done()`
- `as_completed()` for processing results as they finish
- Automatic cleanup with context manager

```python
from concurrent.futures import as_completed

with ThreadPoolExecutor(max_workers=5) as executor:
    future_to_url = {executor.submit(download, url): url for url in urls}

    for future in as_completed(future_to_url):
        url = future_to_url[future]
        try:
            data = future.result()
            print(f"{url}: {len(data)} bytes")
        except Exception as e:
            print(f"{url} failed: {e}")
```

---

## When to Use Which

```
Decision tree:

Is the work I/O-bound or CPU-bound?
├── I/O-bound (network, disk, database):
│   ├── Many concurrent connections (100+)?
│   │   └── asyncio (best: single thread, no overhead)
│   ├── Few connections, simple code?
│   │   └── ThreadPoolExecutor (simpler than asyncio)
│   └── Need compatibility with sync libraries?
│       └── ThreadPoolExecutor
│
└── CPU-bound (computation, data processing):
    ├── Can you use NumPy/C extensions?
    │   └── Just use them (they release the GIL internally)
    ├── Pure Python computation?
    │   └── ProcessPoolExecutor / multiprocessing
    └── Need shared state between workers?
        └── multiprocessing with shared_memory or Manager
```

| Approach | Parallelism | GIL Impact | Overhead | Best For |
|---|---|---|---|---|
| threading | Concurrent, not parallel | Blocked by GIL (CPU) | Low (shared memory) | I/O-bound with few tasks |
| multiprocessing | True parallel | Each process has own GIL | High (IPC, pickling) | CPU-bound |
| asyncio | Concurrent, not parallel | No GIL issues (single thread) | Very low | I/O-bound with many tasks |
| concurrent.futures | Either | Depends on executor | Medium | Clean API for both |

---

## Free-Threading: PEP 703 (Python 3.13+)

Python 3.13 introduced **experimental free-threaded builds** (no GIL). This is the most significant change to CPython's architecture in decades.

```python
# Python 3.13+ with free-threading enabled:
# - GIL is removed entirely
# - Reference counting uses atomic operations
# - All threads can execute Python bytecode truly in parallel
# - Biased reference counting: objects start with thread-local refcount,
#   switch to atomic only when shared across threads

# Check if running free-threaded:
import sys
sys._is_gil_enabled()  # False on free-threaded builds

# Opt-in per-interpreter GIL (for compatibility):
# PYTHON_GIL=1 python script.py
```

**Trade-offs of free-threading:**
- ~5-10% single-threaded performance penalty (atomic refcounting)
- C extensions must be updated for thread safety
- Some C extensions may break (those relying on GIL for thread safety)
- Still experimental — not the default build

---

## Practical Examples / LeetCode Patterns

### Parallel Web Scraping

```python
import asyncio
import aiohttp

async def fetch_all(urls):
    sem = asyncio.Semaphore(50)  # limit to 50 concurrent connections

    async def fetch_one(session, url):
        async with sem:
            async with session.get(url) as resp:
                return await resp.text()

    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        return await asyncio.gather(*tasks, return_exceptions=True)
```

### CPU-Bound Batch Processing

```python
from concurrent.futures import ProcessPoolExecutor
from functools import partial

def process_chunk(data, threshold):
    return [x for x in data if x > threshold]

def parallel_filter(data, threshold, num_workers=4):
    chunk_size = len(data) // num_workers
    chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        results = executor.map(partial(process_chunk, threshold=threshold), chunks)

    return [item for chunk in results for item in chunk]
```

### Producer-Consumer with asyncio

```python
import asyncio

async def producer(queue, items):
    for item in items:
        await queue.put(item)
        print(f"Produced: {item}")
    await queue.put(None)  # sentinel

async def consumer(queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        await asyncio.sleep(0.1)  # simulate processing
        print(f"Consumed: {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=10)
    await asyncio.gather(
        producer(queue, range(20)),
        consumer(queue),
    )

asyncio.run(main())
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| GIL | One thread executes bytecode at a time. Released during I/O. Time-based switching (5ms) |
| threading | Real OS threads. Good for I/O-bound. Useless for CPU-bound (GIL) |
| multiprocessing | Separate interpreters. True parallelism. High IPC overhead (pickling) |
| asyncio | Single-threaded cooperative multitasking. Best for many concurrent I/O operations |
| concurrent.futures | Clean API wrapping both threads and processes. Use `as_completed()` |
| Free-threading | PEP 703, Python 3.13+. Removes GIL. Atomic refcounting. Experimental |
| Rule of thumb | I/O-bound: asyncio or threads. CPU-bound: multiprocessing. NumPy: just use it (releases GIL) |
