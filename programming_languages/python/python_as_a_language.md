# Python as a Language

**Related:** [data types](data_types_in_python.md) | [functions](functions_in_python.md) | [classes & OOP](classes_and_oop.md) | [concurrency](concurrency_in_python.md) | [best practices](python_best_practices.md)

## Overview

Python was created by **Guido van Rossum** and first released in **1991**. It was designed for readability and productivity. Python is governed by **PEPs (Python Enhancement Proposals)** — PEP 20 (The Zen of Python) defines its philosophy: "There should be one — and preferably only one — obvious way to do it."

---

## Interpreter: CPython

Python is **interpreted** (not compiled to native code). The reference implementation is **CPython** — a bytecode compiler + virtual machine written in C.

```
Source (.py) --> Lexer --> Parser --> AST --> Compiler --> Bytecode (.pyc)
                                                              |
                                                              v
                                                    CPython VM (ceval.c)
                                                    Stack-based interpreter
                                                    Executes one opcode at a time
```

### Bytecode Compilation

Python compiles source to **bytecode** (an intermediate representation), NOT machine code. Bytecode is stored in `.pyc` files under `__pycache__/`.

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
#  0 LOAD_FAST    0 (a)
#  2 LOAD_FAST    1 (b)
#  4 BINARY_ADD
#  6 RETURN_VALUE
```

**What's happening:** Each `LOAD_FAST` pushes a local variable onto the VM's value stack. `BINARY_ADD` pops two values, adds them (calling `PyNumber_Add` in C), and pushes the result. `RETURN_VALUE` pops the top of the stack and returns it to the caller.

### CPython VM Internals

The VM is a **stack-based interpreter** implemented in `Python/ceval.c`. The main loop is a giant `switch` statement on opcodes:

```
Frame Object (per function call):
┌──────────────────────────────┐
│ f_code: pointer to code obj  │  <-- bytecode, constants, names
│ f_locals: local variables    │
│ f_globals: global namespace  │
│ f_back: previous frame       │  <-- call stack linkage
│ f_lasti: last bytecode index │
│ value_stack: [...]           │  <-- operand stack for the VM
└──────────────────────────────┘
```

Each function call creates a new **frame object** on the heap (not the C stack). This is why Python stack frames are introspectable — you can examine `sys._getframe()` at runtime.

### Other Implementations

| Implementation | What It Is |
|---|---|
| CPython | Reference. Bytecode interpreter in C. What you use by default |
| PyPy | JIT-compiled Python. 5-10x faster for long-running code |
| Jython | Python on the JVM. No GIL, but limited to Python 2.7 |
| IronPython | Python on .NET CLR |
| MicroPython | Stripped-down Python for microcontrollers |

---

## The GIL (Global Interpreter Lock)

The GIL is a **mutex** that protects access to CPython's internals. Only ONE thread can execute Python bytecode at a time, even on multi-core machines.

```
Thread 1: [acquire GIL] --execute bytecodes-- [release GIL] --wait--
Thread 2: --------wait-------- [acquire GIL] --execute-- [release GIL]
Thread 3: --------wait---------------------------wait--------...

All threads share ONE interpreter, ONE GIL
```

### Why the GIL Exists

CPython uses **reference counting** for memory management. Every object has a `ob_refcnt` field. Without the GIL, every `INCREF`/`DECREF` would need an atomic operation or per-object lock — the overhead would make single-threaded code 30-40% slower.

### GIL Release Points

The GIL is released during:
- I/O operations (file read/write, network calls)
- `time.sleep()`
- C extension code that explicitly releases it (NumPy, database drivers)
- Every N bytecode instructions (default: every 5ms, configurable via `sys.setswitchinterval()`)

**Implication:** Threading is useful for I/O-bound work but useless for CPU-bound work. Use `multiprocessing` or `concurrent.futures.ProcessPoolExecutor` for CPU parallelism.

**Related:** [concurrency](concurrency_in_python.md) for deep dive on GIL, threading, asyncio.

---

## Type System

### Dynamic Typing

Types are checked at **runtime**, not compile time. Variables are just names that reference objects — they have no type themselves.

```python
x = 42          # x references an int object
x = "hello"     # now x references a str object — no error
x = [1, 2, 3]   # now a list — Python doesn't care
```

**How to think about it:** Variables in Python are like sticky notes. The sticky note (name) doesn't have a type — the object it's attached to does.

### Strong Typing

Python will NOT implicitly convert between types:

```python
"hello" + 42     # TypeError: can only concatenate str to str
"hello" + str(42) # "hello42" — explicit conversion required

# Compare with JavaScript (weakly typed):
# "hello" + 42   => "hello42"  (implicit coercion)
```

### Duck Typing

"If it walks like a duck and quacks like a duck, then it IS a duck." Python doesn't check what an object IS — it checks what an object CAN DO.

```python
class Duck:
    def quack(self):
        return "Quack!"

class Person:
    def quack(self):
        return "I'm quacking!"

def make_it_quack(thing):   # no type annotation needed
    return thing.quack()    # works with ANY object that has .quack()

make_it_quack(Duck())    # "Quack!"
make_it_quack(Person())  # "I'm quacking!"
```

**Related:** [classes & OOP](classes_and_oop.md) for protocols (structural subtyping — formalized duck typing).

---

## Memory Management

### Reference Counting (Primary Mechanism)

Every Python object has a reference count. When it drops to zero, the object is immediately freed.

```python
import sys

a = [1, 2, 3]        # refcount = 1
b = a                 # refcount = 2 (b references same list)
print(sys.getrefcount(a))  # 3 (a, b, and the getrefcount argument)

del b                 # refcount drops to 2
del a                 # refcount drops to 1 (getrefcount arg)
                      # when all references gone: freed immediately
```

```
PyObject (every object in CPython):
┌──────────────────────────────┐
│ ob_refcnt: Py_ssize_t        │  <-- reference count (8 bytes on 64-bit)
│ ob_type: *PyTypeObject       │  <-- pointer to type object
│ ... object-specific data ... │
└──────────────────────────────┘
```

### Generational Garbage Collector (Cycle Detector)

Reference counting cannot handle **circular references**:

```python
a = []
b = []
a.append(b)   # a -> b
b.append(a)   # b -> a  (cycle!)
del a, b      # refcounts are both 1, NOT 0 — leaked!
```

CPython's **generational GC** detects and collects cycles. It uses three generations:

```
Generation 0: newly created objects     (collected most frequently)
Generation 1: survived 1 collection     (collected less frequently)
Generation 2: survived 2+ collections   (collected rarely)

Thresholds (default): gen0=700, gen1=10, gen2=10
When gen0 count exceeds 700, a gen0 collection runs.
Every 10 gen0 collections triggers a gen1 collection.
Every 10 gen1 collections triggers a gen2 collection.
```

```python
import gc
gc.get_threshold()   # (700, 10, 10)
gc.collect()         # force a full collection
gc.get_stats()       # collection statistics per generation
```

### Memory Allocator (pymalloc)

CPython uses a custom allocator optimized for small objects (< 512 bytes):

```
OS (mmap/brk)
  └── Python's raw memory allocator (PyMem_Malloc)
        └── pymalloc (object allocator, for objects <= 512 bytes)
              └── Arenas (256KB chunks from OS)
                    └── Pools (4KB, one per size class: 8, 16, 24, ... 512 bytes)
                          └── Blocks (individual object allocations)
```

Objects > 512 bytes go directly to the system `malloc`. This layered design reduces fragmentation and syscall overhead for the many small objects Python creates.

---

## Comparison with Compiled Languages

| Aspect | Python (CPython) | Go | C/C++ |
|---|---|---|---|
| Execution | Bytecode interpreted | Native machine code | Native machine code |
| Type checking | Runtime (dynamic) | Compile time (static) | Compile time (static) |
| Memory management | Refcount + GC | Tracing GC | Manual (or RAII in C++) |
| Concurrency | GIL limits threads | Goroutines (true parallelism) | OS threads (true parallelism) |
| Speed | ~50-100x slower than C | ~2-5x slower than C | Baseline |
| Startup | ~30ms (import overhead) | ~1ms (static binary) | ~0ms |
| Use case | Scripting, ML, web backends | Infrastructure, servers | Systems, performance-critical |

---

## PEP Philosophy Highlights

| PEP | What It Introduced |
|---|---|
| PEP 8 | Style guide (4 spaces, snake_case, 79-char lines) |
| PEP 20 | The Zen of Python (`import this`) |
| PEP 257 | Docstring conventions |
| PEP 484 | Type hints (`def f(x: int) -> str:`) |
| PEP 572 | Walrus operator (`:=`) |
| PEP 380 | `yield from` for generator delegation |
| PEP 557 | Dataclasses |
| PEP 703 | Free-threading (no-GIL) Python |

---

## Key Takeaways

| Aspect | Python's Approach |
|---|---|
| Interpreter | CPython: bytecode compiler + stack-based VM. Not native code |
| GIL | One thread executes bytecode at a time. Released during I/O |
| Type system | Dynamic, strong, duck typing. Types checked at runtime |
| Memory | Reference counting (immediate) + generational GC (cycles) + pymalloc |
| Speed | 50-100x slower than C. Use C extensions, NumPy, or PyPy for performance |
| Philosophy | Readability, explicitness, one obvious way, batteries included |
