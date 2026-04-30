# Go 1.4 Compiler: Middle-End Passes

This document covers the compiler passes between type checking
and code generation: **escape analysis**, **inlining**, **order of
evaluation**, and **AST lowering (walk)**.

---

## Table of Contents

1. [Escape Analysis (esc.c)](#escape-analysis)
2. [Function Inlining (inl.c)](#function-inlining)
3. [Order of Evaluation (order.c)](#order-of-evaluation)
4. [AST Lowering / Walk (walk.c)](#ast-lowering)

---

## Escape Analysis

**Source file**: `src/cmd/gc/esc.c` (1281 lines)

Escape analysis determines whether a variable's lifetime extends
beyond the stack frame of the function that declares it. If a
variable "escapes," it must be heap-allocated; otherwise, it can
stay on the stack (which is cheaper — no GC pressure).

### Algorithm Overview

The analysis runs in four phases:

```
Phase 1: Find Strongly Connected Components (SCCs)
         visit() uses Tarjan-style SCC algorithm
         Groups mutually recursive functions together

Phase 2: Build Data Flow Graph
         escfunc()/esc()/escassign() walk the AST
         Create flow(dst, src) edges for all pointer flows

Phase 3: Flood from Sink
         escflood()/escwalk() walk upstream from theSink
         Mark reachable nodes as escaping

Phase 4: Tag Functions
         esctag() annotates function parameters with
         escape information for cross-package use
```

### Phase 1: Strongly Connected Components (lines 52-157)

The escape analysis first finds **strongly connected components** (SCCs)
in the function call graph. Functions in the same SCC are mutually
recursive and must be analyzed together.

The algorithm is from:

> **Reference**: Sedgewick, R. (1988). "Algorithms", 2nd Edition.
> Addison-Wesley. Page 482. (Adapted Tarjan's SCC algorithm.)
>
> Tarjan, R.E. (1972). "Depth-first search and linear graph
> algorithms". SIAM Journal on Computing, 1(2), pp. 146-160.
> DOI: 10.1137/0201010

**Two adaptations** (from the comment at line 11):

1. **Closures can't be SCC roots**: A closure is forced into the
   same component as its enclosing function, ensuring their
   variables are analyzed together.

2. **Virtual node pair**: Each function gets two node numbers `n`
   and `n+1`. Search starts from `n+1`. If the component number
   comes back as `n+1`, the function is non-recursive. If it comes
   back as `n`, there's a cycle (mutual recursion). This distinction
   lets the analysis be more precise for non-recursive functions.

```c
visit(Node *n) {
    visitgen++;
    n->walkgen = visitgen;    // node n
    visitgen++;
    min = visitgen;           // node n+1 (search starts here)
    // ... search ...
    if(min == n->walkgen)
        recursive = 1;        // path from n+1 back to n → cycle
    else
        recursive = 0;        // trivial component
    analyze(block, recursive);
}
```

### Phase 2: Data Flow Graph (lines 159-900+)

The core of escape analysis builds a **data flow graph** of pointer
flows. The key concept is the `flow(dst, src)` edge, stored in
`dst->escflowsrc`.

**theSink** (line 200): A fake node representing "escapes to heap."
Anything that can flow to theSink must be heap-allocated:
- Return values
- Assignments to global variables
- Parameters of imported functions (unless tagged safe)

**Key flow rules** (from `escassign`, `esc`, `esccall`):

| Go Code | Flow Edge | Meaning |
|---------|-----------|---------|
| `x = y` | `flow(x, y)` | y flows to x |
| `x = &y` | `flow(x, &y)` | address of y flows to x |
| `x = *y` | `flow(x, *y)` | dereference |
| `return x` | `flow(theSink, x)` | x escapes via return |
| `global = x` | `flow(theSink, x)` | x escapes to global |
| `go f(x)` | `flow(theSink, x)` | goroutine args escape |
| `x = m[k]` | `flow(x, m)` | map access |
| `x = <-ch` | `flow(x, ch)` | channel receive |
| `ch <- x` | `flow(theSink, x)` | channel send escapes |

**Loop depth tracking** (lines 374-418):

Variables declared inside loops are treated more conservatively.
The `escloopdepth` field tracks how deeply nested in loops a variable
is. A variable declared at loop depth 2 flowing to a destination
at loop depth 1 might escape (because the outer scope outlives
the inner loop iteration).

Labels that create back-edges (loops via goto) increment the
loop depth:
```c
case OLABEL:
    if(n->left->sym->label == &nonlooping)
        // forward label, no loop
    else if(n->left->sym->label == &looping)
        e->loopdepth++;   // back-jump creates loop
```

### Phase 3: Flood from Sink (escflood/escwalk)

Starting from `theSink`, walk upstream through flow edges. Any
`&` (address-of) node reachable from theSink means the addressed
variable escapes to the heap:

```c
escwalk(level, dst, src) {
    if(src->op == OADDR) {
        // address of src->left flows to the heap
        addrescapes(src->left);  // move to heap
        src->left->esc = EscHeap;
    }
}
```

### Phase 4: Tag Parameters (esctag)

For exported functions, the escape analysis results are encoded
as **tags** on function parameters. These tags are written into
the export data so that callers in other packages know which
parameters escape.

```c
// Tag format: "esc:0xNN"
// Bits encode which return values a parameter flows to
mktag(mask);    // creates the tag string
parsetag(note); // reads the tag from import data
```

### Escape Analysis Results

Each variable gets one of these escape statuses:

| Status | Meaning | Allocation |
|--------|---------|-----------|
| `EscNone` (3) | Does not escape | Stack |
| `EscHeap` (1) | Escapes to heap | Heap (`runtime.newobject`) |
| `EscReturn` (4) | Returned from function | Depends on caller |
| `EscScope` (2) | Stays in declaring scope | Stack |
| `EscNever` (5) | Compiler builtin | N/A |

When `-m` flag is used, the compiler prints decisions:
```
./main.go:5: moved to heap: x
./main.go:8: y does not escape
```

### References

- Choi, J.D., Gupta, M., Serrano, M.J., Sreedhar, V.C., Midkiff, S.P.
  (1999). "Escape Analysis for Java". Proceedings of OOPSLA 1999,
  pp. 1-19. DOI: 10.1145/320384.320386

- Blanchet, B. (1999). "Escape Analysis for Object-Oriented Languages:
  Application to Java". Proceedings of OOPSLA 1999, pp. 20-34.
  DOI: 10.1145/320384.320387

- Tarjan, R.E. (1972). "Depth-first search and linear graph algorithms".
  SIAM Journal on Computing, 1(2), pp. 146-160.
  DOI: 10.1137/0201010

---

## Function Inlining

**Source file**: `src/cmd/gc/inl.c` (986 lines)

Inlining replaces function calls with the function's body,
eliminating call overhead and enabling further optimizations.

### Algorithm Overview (from comment, lines 5-28)

The inliner makes **two passes**:

1. **`caninl(fn)`**: Determine if `fn` is inlineable. If yes, save
   a copy of the body in `fn->nname->inl`.

2. **`inlcalls(fn)`**: Walk `fn`'s body and replace calls to
   inlineable functions with their inlined bodies.

### Inlining Levels (lines 10-17)

```
Level 0: Inlining disabled (-l flag)
Level 1: DEFAULT — 40-node leaf functions, oneliners
Level 2: Early typechecking of all imported bodies
Level 3: Allow variadic functions
Level 4: Allow non-leaf functions (breaks runtime.Caller)
Level 5: Transitive inlining
```

### caninl() — Inlineability Check (lines 112-161)

A function is inlineable if:
1. It has a body (not extern/assembly)
2. It's already type-checked
3. No variadic parameters (unless level >= 3)
4. It's not too "hairy" (complexity budget <= 40 nodes)

```c
void caninl(Node *fn) {
    if(fn->nbody == nil) return;     // no body
    budget = 40;                     // hairiness budget
    if(ishairylist(fn->nbody, &budget))
        return;                       // too complex
    fn->nname->inl = fn->nbody;      // save body copy
    fn->nbody = inlcopylist(fn->nname->inl);  // substitute copy
}
```

### ishairy() — Complexity Check (lines 163-219)

Certain operations are **always too hairy** regardless of budget:

| Always rejected | Reason |
|----------------|--------|
| `OCALL*` | Non-leaf (unless level >= 4) |
| `OPANIC` | Changes control flow |
| `ORECOVER` | Depends on stack frame |
| `OCLOSURE` | Captures variables |
| `ORANGE` | Range loops are complex |
| `OFOR` | Loops are complex |
| `OSELECT` | Select is complex |
| `OSWITCH` | Switch is complex |
| `OPROC` | go statement |
| `ODEFER` | defer statement |

Everything else costs 1 from the budget. If budget goes below 0,
the function is too complex.

**The budget of 40** means roughly: a function with <= 40 simple
operations (assignments, arithmetic, returns) can be inlined.

### mkinlcall() — Inline Expansion

When a call to an inlineable function is found:

```
Before:  result = f(arg1, arg2)

After:   {
             // declare temps for params and returns
             .param0 = arg1
             .param1 = arg2
             // substitute body
             ... body with params replaced by .param temps ...
             ... returns replaced by goto .retlabel ...
         .retlabel:
             result = .retvar
         }
```

The substitution uses `inlsubst()` which:
1. Creates temporary variables for all parameters
2. Creates temporary variables for all return values
3. Walks the body, replacing:
   - Parameter references → param temps
   - Return statements → assignment to return temps + goto retlabel
   - Local variables → fresh local variables (to avoid name collision)

### References

- Allen, F.E. and Cocke, J. (1972). "A Catalogue of Optimizing
  Transformations". In Design and Optimization of Compilers,
  pp. 1-30. Prentice-Hall.

- Cooper, K.D. and Torczon, L. (2011). "Engineering a Compiler",
  2nd Edition. Section 8.7: "Inline Substitution".
  Morgan Kaufmann. ISBN: 978-0-12-088478-0.

---

## Order of Evaluation

**Source file**: `src/cmd/gc/order.c` (1101 lines)

The `order()` pass ensures Go's **left-to-right evaluation order**
is maintained and introduces temporary variables where needed.

### Why Is This Needed?

Consider:
```go
m[k] = f()  // What if f() modifies m or k?
```

The order pass ensures `k` is evaluated before `f()` by introducing
a temporary:
```go
tmp := k
m[tmp] = f()
```

### Key Transformations

1. **Map index expressions**: Map lookups can trigger rehashing.
   The map and key are copied to temps before any side effects.

2. **Multi-value assignments**: `a, b = b, a` needs temps to
   avoid overwriting before reading.

3. **Function arguments**: Evaluated left-to-right into temps.

4. **Channel operations in select**: Each case's channel expression
   is pre-evaluated.

5. **Composite literal elements**: Complex elements are evaluated
   into temps in order.

---

## AST Lowering (walk.c)

**Source file**: `src/cmd/gc/walk.c` (3937 lines)

The `walk()` pass is the **final transformation** before code
generation. It "lowers" high-level Go operations into sequences
of simpler operations and runtime function calls.

### Overview

```c
void walk(Node *fn) {
    // Walk function body
    walkstmtlist(fn->nbody);
}

void walkstmt(Node **np) {
    // Handle statements: if, for, return, assignments, etc.
}

void walkexpr(Node **np, NodeList **init) {
    // Handle expressions: arithmetic, calls, type conversions, etc.
}
```

### Major Transformations

#### Map Operations → Runtime Calls

```go
// Source:         Lowered to:
v = m[k]       → v = *runtime.mapaccess1(type, m, &k)
v, ok = m[k]   → v, ok = runtime.mapaccess2(type, m, &k)
m[k] = v       → runtime.mapassign1(type, m, &k, &v)
delete(m, k)   → runtime.mapdelete(type, m, &k)
```

#### Channel Operations → Runtime Calls

```go
c <- v         → runtime.chansend1(c, &v)
v = <-c        → runtime.chanrecv1(c, &v)
v, ok = <-c    → runtime.chanrecv2(c, &v)
```

#### Goroutines and Defer → Runtime Calls

```go
go f(args)     → runtime.newproc(size, f, args)
defer f(args)  → runtime.deferproc(size, f, args)
```

The arguments are packed into a contiguous block on the stack.

#### Make Operations → Runtime Calls

```go
make([]T, n, m) → runtime.makeslice(type, n, m)
make(map[K]V, n) → runtime.makemap(type, n)
make(chan T, n) → runtime.makechan(type, n)
```

#### String Operations

```go
s1 + s2 + s3     → runtime.concatstrings([s1, s2, s3])
string([]byte)    → runtime.slicebytetostring(buf, b)
[]byte(string)    → runtime.stringtoslicebyte(buf, s)
string(rune)      → runtime.intstring(buf, r)
```

#### Type Assertions → Runtime Calls

```go
v = i.(T)        → runtime.assertI2T(type, i)
v, ok = i.(T)    → runtime.assertI2T2(type, i)
v = i.(I)        → runtime.assertI2I(type, i)
```

#### Interface Operations

```go
I(concrete)      → runtime.convT2I(type, itab, &concrete)
interface{}(x)   → runtime.convT2E(type, &x)
```

#### New and Allocation

```go
new(T)           → runtime.newobject(type)
&T{...}          → tmp := T{...}; &tmp  (or heap alloc if escapes)
```

#### Select Statement → Runtime Calls

The `select` statement is transformed into a complex sequence:
```go
select {         → hselect = runtime.newselect(n)
case c <- v:        runtime.selectsend(hselect, c, &v)
case v = <-c:       runtime.selectrecv(hselect, c, &v)
default:            runtime.selectdefault(hselect)
}                   chosen = runtime.selectgo(hselect)
                    switch chosen { ... }
```

#### Panic/Recover

```go
panic(v)         → runtime.gopanic(v)
recover()        → runtime.gorecover()
```

#### Bounds Checking

Unless `-B` flag is set, array/slice index operations get
bounds checks inserted:
```go
a[i]             → if i >= len(a) { runtime.panicindex() }; a[i]
```

#### Closure Variables

Closure variable access is lowered to indirect access through
the closure's context pointer:
```go
func() { x++ }  → func(ctx) { *ctx.x++ }
```

### Write Barriers

When `use_writebarrier` is enabled (default in Go 1.4), pointer
writes get write barrier wrappers:
```go
*p = v           → runtime.writebarrierptr(p, v)
```

This is essential for the **concurrent garbage collector** — the
write barrier notifies the GC when pointer fields are modified,
preventing it from missing live objects.

> **Reference**: Dijkstra, E.W., Lamport, L., Martin, A.J.,
> Scholten, C.S., and Steffens, E.F.M. (1978). "On-the-fly
> garbage collection: An exercise in cooperation". Communications
> of the ACM, 21(11), pp. 966-975. DOI: 10.1145/359642.359655
>
> (The tri-color abstraction and write barrier concept.)

### References

- Wilson, P.R. (1992). "Uniprocessor Garbage Collection Techniques".
  Proceedings of the International Workshop on Memory Management
  (IWMM '92). LNCS 637, pp. 1-42.

- Appel, A.W. (1998). "Modern Compiler Implementation in C".
  Cambridge University Press. Chapter 15: "Garbage Collection".
