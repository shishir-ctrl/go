# Go 1.4 Runtime

**Source directory**: `src/runtime/` (C + Assembly)

The Go runtime provides goroutine scheduling, garbage collection,
memory allocation, channel operations, and other fundamental services.
In Go 1.4, the runtime is written in **C and assembly** (it was later
rewritten in Go for 1.5).

---

## Table of Contents

1. [Runtime Architecture](#runtime-architecture)
2. [Goroutine Scheduler (proc.c)](#goroutine-scheduler)
3. [Garbage Collector (mgc0.c)](#garbage-collector)
4. [Memory Allocator (malloc.c)](#memory-allocator)
5. [Channels (chan.c)](#channels)
6. [Maps (hashmap.c)](#maps)
7. [Stack Management (stack.c)](#stack-management)
8. [Defer/Panic/Recover (panic.c)](#deferpanic-recover)
9. [Select (select.c)](#select)

---

## Runtime Architecture

The runtime is organized around three core entities:

```
┌─────────────────────────────────────────┐
│              Go Program                  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐  │
│  │  G   │ │  G   │ │  G   │ │  G   │  │  G = Goroutine
│  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘  │
│     │        │        │        │       │
│  ┌──▼───┐ ┌──▼───┐                    │
│  │  P   │ │  P   │    (GOMAXPROCS)     │  P = Processor
│  └──┬───┘ └──┬───┘                    │
│     │        │                         │
│  ┌──▼───┐ ┌──▼───┐ ┌──────┐          │
│  │  M   │ │  M   │ │  M   │ (idle)   │  M = Machine (OS thread)
│  └──────┘ └──────┘ └──────┘          │
└─────────────────────────────────────────┘
```

### G — Goroutine (lines 19)

A goroutine is a lightweight thread of execution. Each G has:
- Its own stack (initially 8KB in Go 1.4, growable)
- Program counter and registers (saved when descheduled)
- Status (running, runnable, waiting, dead)

### M — Machine / OS Thread (line 20)

An M is an OS thread. Each M executes goroutines. An M must have
an associated P to execute Go code, but can exist without one
(when blocked in a syscall).

### P — Processor (line 21-23)

A P is a logical processor — a resource required to execute Go code.
The number of Ps is controlled by `GOMAXPROCS`. Each P has:
- A local run queue of goroutines
- A cache of memory allocation spans (mcache)
- State for the garbage collector

> **Reference**: Vyukov, D. (2012). "Scalable Go Scheduler Design Doc".
> https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw
>
> This design doc describes the M:P:G model introduced in Go 1.1.
> Go 1.4 uses the same model.

---

## Goroutine Scheduler (proc.c)

**Source**: `src/runtime/proc.c`

### The schedule() Function

The heart of the scheduler. Called when:
- A goroutine yields or blocks
- A goroutine finishes
- A new goroutine becomes runnable

```c
static void schedule(void) {
    // 1. If GC is waiting, assist with GC
    // 2. Try to get a goroutine from the local run queue
    // 3. If empty, try to steal from other Ps
    // 4. If still empty, check the global run queue
    // 5. If still empty, poll network
    // 6. If found, execute the goroutine
}
```

### Work Stealing

When a P's local queue is empty, it **steals** goroutines from
other Ps' queues. This is a key technique for load balancing:

```c
static G* runqsteal(P *p, P *p2) {
    // Steal half of p2's run queue and put it in p's queue
}
```

> **Reference**: Blumofe, R.D. and Leiserson, C.E. (1999).
> "Scheduling Multithreaded Computations by Work Stealing".
> Journal of the ACM, 46(5), pp. 720-748.
> DOI: 10.1145/324133.324234

### Key Operations

| Function | Purpose |
|----------|---------|
| `runtime·newproc` | Create new goroutine (`go f()`) |
| `runtime·gosched` | Yield the processor |
| `runtime·gopark` | Park goroutine (block) |
| `runtime·goready` | Wake up parked goroutine |
| `runtime·goexit` | Goroutine exits |
| `schedule()` | Pick next goroutine to run |
| `runqput()` | Add G to P's local queue |
| `runqget()` | Get G from P's local queue |
| `runqsteal()` | Steal G from another P |
| `sysmon()` | System monitor (preemption, deadlock detection) |

### sysmon() — System Monitor

A special goroutine that runs periodically to:
- Preempt goroutines that have been running too long (>10ms)
- Detect deadlocks (all goroutines blocked)
- Force GC if needed
- Poll the network for I/O completions

### Syscall Handling

When a goroutine enters a syscall:
1. Its M releases the P (`handoffp()`)
2. Another M can pick up the P and run other goroutines
3. When the syscall returns, the goroutine tries to reacquire a P

This ensures syscalls don't block all Go code.

---

## Garbage Collector (mgc0.c)

**Source**: `src/runtime/mgc0.c`

### GC Characteristics (from the header comment)

```
- Mark & Sweep
- Mostly precise (knows pointer vs non-pointer)
- Parallel (up to MaxGcproc threads)
- Partially concurrent (mark is STW, sweep is concurrent)
- Non-moving / non-compacting
- Full (non-generational)
```

### GC Algorithm

```
Phase 1: STOP THE WORLD
    - Stop all goroutines
    - Mark all reachable objects using tri-color marking

Phase 2: MARK (parallel, stop-the-world)
    - Start from roots (stacks, globals)
    - Trace all reachable pointers
    - Mark each reachable object as "black"

Phase 3: START THE WORLD + SWEEP (concurrent)
    - Resume goroutines
    - Sweep unmarked objects in background
    - Reclaim memory lazily
```

### GC Rate

```c
// GOGC=100 (default): GC when heap doubles
// next_gc = live_heap * (1 + GOGC/100)
// If using 4MB, GC at 8MB. If using 8MB, GC at 16MB.
```

### Tri-Color Abstraction

Objects are conceptually colored:
- **White**: Not yet scanned (potentially garbage)
- **Grey**: Reachable but children not yet scanned
- **Black**: Reachable and all children scanned

The invariant: no black object points to a white object. The
**write barrier** maintains this invariant during concurrent marking.

> **Reference**: Dijkstra, E.W., Lamport, L., Martin, A.J.,
> Scholten, C.S., and Steffens, E.F.M. (1978). "On-the-fly
> garbage collection: An exercise in cooperation". Communications
> of the ACM, 21(11), pp. 966-975. DOI: 10.1145/359642.359655

### Concurrent Sweep

The sweep phase runs concurrently with the program:
- **Lazy sweeping**: When a goroutine needs a new span, it sweeps
  existing spans first
- **Background sweeper**: A dedicated goroutine sweeps in background

---

## Memory Allocator (malloc.c)

**Source**: `src/runtime/malloc.c`

Go's memory allocator is inspired by **TCMalloc** (Thread-Caching
Malloc) from Google.

### Architecture

```
┌──────────────────────────────┐
│          Heap                │
│  ┌────────────────────────┐ │
│  │    MHeap               │ │  Central heap manager
│  │  ┌──────────────────┐ │ │
│  │  │  MCentral (×67)  │ │ │  Per-size-class freelists
│  │  └──────────────────┘ │ │
│  └────────────────────────┘ │
│                              │
│  ┌──────┐ ┌──────┐         │
│  │MCache│ │MCache│  (per P) │  Thread-local caches
│  └──────┘ └──────┘         │
└──────────────────────────────┘
```

### Size Classes

Small objects (< 32KB) are allocated from size-class buckets:
- 67 size classes from 8 bytes to 32KB
- Objects are rounded up to the nearest size class
- Reduces fragmentation

### Allocation Path

```
1. Tiny allocation (< 16 bytes, no pointers)?
   → Use tiny allocator (pack multiple into one object)

2. Small allocation (< 32KB)?
   → Get from MCache (per-P, no locking!)
   → If MCache empty, refill from MCentral (lock required)
   → If MCentral empty, get from MHeap

3. Large allocation (>= 32KB)?
   → Allocate directly from MHeap
```

> **Reference**: Ghemawat, S. and Menage, P. "TCMalloc: Thread-Caching
> Malloc". Google Performance Tools.
> https://google.github.io/tcmalloc/design.html

---

## Channels (chan.c)

**Source**: `src/runtime/chan.c`

Channels implement Go's primary synchronization primitive, based
on **Communicating Sequential Processes (CSP)**.

### Channel Structure

```c
struct Hchan {
    uintgo  qcount;    // total data in the queue
    uintgo  dataqsiz;  // size of circular queue (buffer)
    byte*   buf;       // circular buffer
    uint16  elemsize;  // element size
    uint32  closed;    // is channel closed?
    Type*   elemtype;  // element type
    uintgo  sendx;     // send index in circular buffer
    uintgo  recvx;     // receive index
    WaitQ   recvq;     // list of recv waiters
    WaitQ   sendq;     // list of send waiters
    Mutex   lock;      // protects all fields
};
```

### Send Operation (`runtime·chansend`)

```
1. If channel is nil → block forever
2. Lock the channel
3. If channel is closed → panic
4. If there's a waiting receiver:
   → Copy data directly to receiver, wake it
5. If buffer has space:
   → Copy data to buffer
6. Otherwise:
   → Park the goroutine (block until space available)
```

### Receive Operation (`runtime·chanrecv`)

```
1. If channel is nil → block forever
2. Lock the channel
3. If there's a waiting sender:
   → Copy data directly from sender, wake it
4. If buffer has data:
   → Copy data from buffer
5. If channel is closed:
   → Return zero value
6. Otherwise:
   → Park the goroutine (block until data available)
```

> **Reference**: Hoare, C.A.R. (1978). "Communicating Sequential
> Processes". Communications of the ACM, 21(8), pp. 666-677.
> DOI: 10.1145/359576.359585
>
> Hoare, C.A.R. (1985). "Communicating Sequential Processes".
> Prentice Hall. ISBN: 0-13-153289-8.
> Available free: http://www.usingcsp.com/

---

## Maps (hashmap.c)

**Source**: `src/runtime/hashmap.c`

Go maps are hash tables with separate chaining using **buckets**.

### Map Structure

```c
struct Hmap {
    uintgo  count;       // number of elements
    uint32  flags;
    uint8   B;           // log_2 of number of buckets
    uint16  bucketsize;
    byte*   buckets;     // array of 2^B Buckets
    byte*   oldbuckets;  // previous bucket array (during grow)
    uintptr nevacuate;   // progress counter for evacuation
};

struct Bucket {
    uint8   tophash[8];  // top byte of hash for each key
    // followed by 8 keys, then 8 values
    Bucket* overflow;    // overflow bucket
};
```

### Key Design Decisions

1. **8 entries per bucket**: Each bucket holds 8 key-value pairs.
   This improves cache locality vs one-entry-per-bucket.

2. **Top hash optimization**: The top byte of each hash is stored
   in `tophash[]`. This enables fast comparison without loading
   the full key — most mismatches are caught by comparing one byte.

3. **Incremental growth**: When the map grows (load factor > 6.5),
   entries are evacuated from old buckets to new ones **incrementally**
   during normal map operations, not all at once.

4. **Key/value layout**: Keys are stored together, then values
   together (not interleaved). This improves alignment for types
   where key and value have different alignments.

---

## Stack Management (stack.c)

**Source**: `src/runtime/stack.c`

### Contiguous Stacks (Go 1.4)

Go 1.4 uses **contiguous stacks** (also called "copied stacks"):

1. Each goroutine starts with a small stack (8KB default)
2. If a function needs more stack, the runtime:
   - Allocates a new, larger stack (2x the old size)
   - Copies the old stack contents to the new stack
   - Updates all pointers that reference the old stack
   - Frees the old stack

**Stack check**: At function entry, the compiler inserts a check:
```asm
CMPQ    SP, stackguard
JBE     morestack
```

If SP is below the stack guard, `morestack()` is called to grow
the stack.

### Previous Approach: Segmented Stacks

Go versions before 1.3 used **segmented stacks** — multiple small
stack segments linked together. This was abandoned because of the
**"hot split" problem**: a function at a segment boundary would
repeatedly trigger allocation and deallocation as it was called
in a loop.

> **Reference**: Go Team (2013). "Contiguous Stacks Design Doc".
> https://docs.google.com/document/d/1wAaf1rYoM4nCTUWO94WDGb73ZhIMZnJ_PkNAupvLa5o

---

## Defer/Panic/Recover (panic.c)

**Source**: `src/runtime/panic.c`

### Defer

`defer f()` pushes a function call onto a **per-goroutine defer stack**.
When the function returns (normally or via panic), deferred calls
are executed in **LIFO order**.

```c
struct Defer {
    int32   siz;        // argument size
    bool    started;    // execution started
    FuncVal *fn;        // function to call
    Defer   *link;      // next deferred call
    void    *args;      // arguments
};
```

### Panic

`panic(v)` starts **unwinding** the goroutine's call stack:
1. Run all deferred functions in the current function
2. Pop to the calling function
3. Run its deferred functions
4. Repeat until the goroutine exits (→ program crash)

### Recover

`recover()` can only be called from a deferred function. It:
1. Stops the panic unwinding
2. Returns the panic value
3. Continues execution after the deferred function returns

```c
void runtime·gorecover(byte *argp) {
    Panic *p = g->panic;
    if(p != nil && !p->recovered && p->argp == argp) {
        p->recovered = true;
        return p->arg;
    }
    return nil;
}
```

---

## Select (select.c)

**Source**: `src/runtime/select.c`

The `select` statement is compiled to runtime calls:

### Algorithm

1. **Lock all channels** (in address order to avoid deadlock)
2. **Scan all cases** for a ready channel:
   - Send case: channel has buffer space or waiting receiver
   - Receive case: channel has data or waiting sender
3. **If found**: execute the operation and return
4. **If not found and there's a default**: execute default
5. **If not found**: park the goroutine on ALL channels' wait queues
6. **When woken**: determine which case fired, clean up other queues

The channels are locked **in address order** (sorted by memory
address) to prevent deadlock when two goroutines select on the
same set of channels in different orders.

### Randomized Case Selection

If multiple cases are ready, `select` chooses one **at random**
(using a fast PRNG). This prevents starvation and is specified
by the Go language specification.

---

## References

### Scheduling
- Vyukov, D. (2012). "Scalable Go Scheduler Design Doc".
  https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw
- Blumofe, R.D. and Leiserson, C.E. (1999). "Scheduling Multithreaded
  Computations by Work Stealing". JACM, 46(5), pp. 720-748.
  DOI: 10.1145/324133.324234

### Garbage Collection
- Dijkstra, E.W. et al. (1978). "On-the-fly garbage collection".
  CACM, 21(11), pp. 966-975. DOI: 10.1145/359642.359655
- Wilson, P.R. (1992). "Uniprocessor Garbage Collection Techniques".
  IWMM '92. LNCS 637, pp. 1-42.
- Jones, R., Hosking, A., Moss, E. (2011). "The Garbage Collection
  Handbook". Chapman and Hall/CRC. ISBN: 978-1420082791.

### Memory Allocation
- Ghemawat, S. and Menage, P. "TCMalloc: Thread-Caching Malloc".
  https://google.github.io/tcmalloc/design.html
- Bonwick, J. (1994). "The Slab Allocator". USENIX Summer Technical
  Conference, pp. 87-98.

### CSP and Channels
- Hoare, C.A.R. (1978). "Communicating Sequential Processes".
  CACM, 21(8), pp. 666-677. DOI: 10.1145/359576.359585
- Hoare, C.A.R. (1985). "Communicating Sequential Processes".
  Prentice Hall. Free: http://www.usingcsp.com/

### Stack Management
- Go Team (2013). "Contiguous Stacks Design Doc".
  https://docs.google.com/document/d/1wAaf1rYoM4nCTUWO94WDGb73ZhIMZnJ_PkNAupvLa5o
