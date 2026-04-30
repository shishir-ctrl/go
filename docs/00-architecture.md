# Go 1.4 Compiler Architecture

## Overview

The Go 1.4.3 compiler is the **last version of the Go compiler written in C**.
Starting with Go 1.5, the compiler was rewritten in Go itself (bootstrapped).
This C codebase represents the original compiler as conceived by
**Rob Pike**, **Ken Thompson**, and **Robert Griesemer** at Google, starting in 2007.

The compiler is written in **Plan 9 C**, a dialect of C from Bell Labs'
Plan 9 operating system. It differs from ANSI C in several ways (see
[Appendix: Plan 9 C Dialect](#plan-9-c-dialect)).

### References

- Pike, R., Thompson, K., Griesemer, R. (2009). "The Go Programming Language".
  Announcement at Google, November 10, 2009.
- "The Go Programming Language Specification" (2014).
  https://go.dev/ref/spec (version matching Go 1.4)
- Pike, R. et al. "Plan 9 from Bell Labs".
  https://9p.io/plan9/
- Thompson, K., Pike, R. "Plan 9 C Compilers" (2002).
  https://9p.io/sys/doc/compiler.html

---

## Compilation Pipeline

The Go 1.4 compiler processes source code through **7 phases**, all
implemented in `src/cmd/gc/`. The phases execute sequentially in `main()`
in `lex.c` (lines 199-518).

```
                    Source Code (.go files)
                           │
                    ┌──────▼──────┐
          Phase 0   │    LEXER    │  lex.c: tokenizes source into tokens
                    │  + PARSER   │  go.y/y.tab.c: builds AST from tokens
                    └──────┬──────┘
                           │ AST (Abstract Syntax Tree)
                    ┌──────▼──────┐
          Phase 1   │  TYPECHECK  │  typecheck.c: types for consts, types, func signatures
                    │  (pass 1)   │  lex.c:429-431
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 2   │  TYPECHECK  │  typecheck.c: variable assignments
                    │  (pass 2)   │  lex.c:435-438 — needs interface info from pass 1
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 3   │  TYPECHECK  │  typecheck.c: function bodies
                    │  (pass 3)   │  lex.c:441-450
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 4   │  INLINING   │  inl.c: identify inlineable functions, expand inline calls
                    │             │  lex.c:458-481
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 5   │   ESCAPE    │  esc.c: determine which variables escape to heap
                    │  ANALYSIS   │  lex.c:489
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 6   │   COMPILE   │  For each function:
                    │  FUNCTIONS  │    order.c → walk.c → gen.c → 6g/
                    │             │  lex.c:496-498
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
          Phase 7   │  EMIT OBJ   │  obj.c: write object file (.6, .8, .5)
                    │   FILE      │  lex.c:511
                    └──────┴──────┘
                           │
                        .6 file ──► linker (6l) ──► executable
```

### Why Multiple Type-Checking Phases?

The compiler splits type checking into 3 phases because of **declaration
order dependencies** in Go:

- **Phase 1**: Process `const`, `type`, and function signatures. This
  gathers all type information without depending on variable values.
- **Phase 2**: Process `var` declarations and assignments. These may
  involve interface satisfaction checks, which require the type
  information from Phase 1.
- **Phase 3**: Type-check function bodies. These can reference any
  declaration from Phases 1 and 2.

This mirrors the Go specification's rule that top-level declarations
can be in any order (unlike C, which requires declaration before use).

---

## Directory Structure

```
src/cmd/gc/          Frontend (architecture-independent)
├── go.h             Master header — all data structures
├── lex.c            Lexer + main() entry point
├── go.y             YACC grammar (source for y.tab.c)
├── y.tab.c          Generated parser
├── typecheck.c      Type checking
├── dcl.c            Declarations
├── walk.c           AST rewriting / lowering
├── order.c          Order of evaluation
├── esc.c            Escape analysis
├── inl.c            Function inlining
├── gen.c            Intermediate code generation
├── pgen.c           Portable code generation
├── plive.c          Liveness analysis
├── popt.c           Portable optimization (data flow)
├── const.c          Constant folding / evaluation
├── subr.c           Utility subroutines
├── closure.c        Closure handling
├── sinit.c          Static initializer optimization
├── swt.c            Switch statement compilation
├── select.c         Select statement compilation
├── range.c          Range loop compilation
├── reflect.c        Reflection / type descriptor generation
├── export.c         Package export data
├── fmt.c            AST pretty-printing
├── obj.c            Object file output
├── align.c          Type size / alignment calculation
├── unsafe.c         unsafe.Sizeof / Offsetof
├── racewalk.c       Race detector instrumentation
├── cplx.c           Complex number operations
├── mparith1.c       Multi-precision arithmetic (comparisons)
├── mparith2.c       Multi-precision arithmetic (integer ops)
├── mparith3.c       Multi-precision arithmetic (float ops)
├── array.c          Dynamic array utility
├── bits.c           Bitset operations
├── bv.c             Bit vector operations
└── md5.c            MD5 hash (for type identity)

src/cmd/6g/          amd64 backend
├── cgen.c           Code generation (amd64)
├── ggen.c           General code generation
├── gsubr.c          Code generation subroutines
├── galign.c         Architecture constants
├── peep.c           Peephole optimizer
├── prog.c           Instruction representation
├── reg.c            Register allocation
└── pgen.c           Platform-specific code gen

src/cmd/6l/          amd64 linker
src/cmd/8g/          x86 (32-bit) backend
src/cmd/5g/          ARM backend
src/cmd/ld/          Shared linker code

src/runtime/         Runtime (C + assembly)
├── proc.c           Goroutine scheduler
├── mgc0.c           Garbage collector
├── malloc.c         Memory allocator
├── chan.c            Channel operations
├── hashmap.c        Map implementation
├── select.c         Select implementation
├── panic.c          Panic/recover/defer
├── stack.c          Stack management
├── asm_amd64.s      Assembly entry points
└── ...
```

---

## Architecture-Specific Naming Convention

The Go 1.4 compiler uses a naming convention inherited from Plan 9:

| Prefix | Architecture | Register Width |
|--------|-------------|---------------|
| `5`    | ARM (ARMv5+)| 32-bit        |
| `6`    | amd64       | 64-bit        |
| `8`    | x86 (i386)  | 32-bit        |

| Suffix | Tool        | Example (amd64) |
|--------|-------------|-----------------|
| `g`    | Compiler    | `6g`            |
| `l`    | Linker      | `6l`            |
| `a`    | Assembler   | `6a`            |
| `c`    | C compiler  | `6c` (for runtime C code) |

This naming comes directly from Plan 9's compiler suite:

> "The names of the compilers are derived from the names of the
>  processors: `8c` is the x86 C compiler, `6c` is the amd64 C compiler."
>  — Ken Thompson, Rob Pike, "Plan 9 C Compilers"

### References

- Thompson, K., Pike, R. "Plan 9 C Compilers" (2002).
  https://9p.io/sys/doc/compiler.html
- Winterbottom, P., Pike, R. "The Design of the Inferno Virtual Machine" (1997).
  Bell Labs Technical Report.

---

## The Compilation of a Single Function

When the compiler reaches Phase 6 (line 496-498 of `lex.c`), each
function goes through a sub-pipeline:

```
funccompile(fn)     dcl.c:funccompile()
    │
    ▼
order(fn)           order.c — Establish order of evaluation
    │                         Break complex expressions into temps
    ▼
walk(fn)            walk.c — Lower AST to simpler operations
    │                        Convert high-level ops (map access,
    │                        channel ops, etc.) to runtime calls
    ▼
compile(fn)         6g/pgen.c — Generate machine instructions
    │
    ├── cgen()      6g/cgen.c — Recursive code generation
    │                           for each AST node
    ├── regalloc()  6g/reg.c  — Graph-coloring register allocation
    └── peep()      6g/peep.c — Peephole optimization
```

### order.c: Order of Evaluation

The `order.c` pass ensures Go's evaluation order guarantees are met.
Go specifies left-to-right evaluation in most contexts. This pass:

1. Introduces temporary variables for complex sub-expressions
2. Ensures function arguments are evaluated in the correct order
3. Moves map index expressions into temporaries (maps can rehash)

### walk.c: AST Lowering

The `walk.c` pass is the **largest single file** (~3900 lines). It
transforms high-level Go operations into lower-level representations:

| Go Operation        | Lowered To                     |
|--------------------|--------------------------------|
| `m[k]`             | `runtime.mapaccess1(m, &k)`    |
| `m[k] = v`         | `runtime.mapassign1(m, &k, &v)`|
| `ch <- v`          | `runtime.chansend1(ch, &v)`    |
| `<-ch`             | `runtime.chanrecv1(ch, &v)`    |
| `go f()`           | `runtime.newproc(f, args)`     |
| `defer f()`        | `runtime.deferproc(f, args)`   |
| `make([]T, n)`     | `runtime.makeslice(T, n, n)`   |
| `make(map[K]V)`    | `runtime.makemap(T, hint)`     |
| `make(chan T, n)`   | `runtime.makechan(T, n)`       |
| `s1 + s2` (string) | `runtime.concatstrings()`      |
| `new(T)`           | `runtime.newobject(T)`         |
| `panic(v)`         | `runtime.gopanic(v)`           |
| `recover()`        | `runtime.gorecover()`          |

---

## Plan 9 C Dialect

The Go 1.4 compiler is written in Plan 9 C, which has notable
differences from ANSI/ISO C:

### Key Differences

1. **No `#include <stdio.h>`**: Plan 9 uses `<u.h>` and `<libc.h>`.
2. **`vlong`**: 64-bit integer type (instead of `long long`).
3. **`USED(x)`**: Macro to suppress "unused variable" warnings.
4. **`nelem(a)`**: Macro for array element count.
5. **`nil`** instead of `NULL`.
6. **No bitfield restrictions**: Bit fields in structs are used freely.
7. **`#pragma varargck`**: Type-safe printf-like format checking.
8. **`exits()`**: Instead of `exit()`.
9. **`Biobuf`**: Buffered I/O (like stdio's `FILE*`, but from Plan 9).
10. **`Fmt`**: Plan 9 formatted output system (replaces `printf`).

### Why Plan 9 C?

The Go creators (Pike, Thompson) were the designers of Plan 9. They
carried their tools and conventions into Go's initial implementation.
This is why the compiler uses Plan 9 libraries (`lib9/`, `libbio/`)
even when building on Linux/macOS.

### References

- Pike, R., Presotto, D., Dorward, S., Flandrena, B., Thompson, K.,
  Trickey, H., Winterbottom, P. (1995). "Plan 9 from Bell Labs".
  Computing Systems, 8(3), pp. 221-254.
- Pike, R. "How to Use the Plan 9 C Compiler" (2002).
  https://9p.io/sys/doc/comp.html

---

## Key Data Structures (go.h)

The entire compiler revolves around four central data structures,
all defined in `go.h`:

### Node (lines 245-352)

The `Node` struct is the **AST node** — the fundamental unit of the
compiler's intermediate representation. Every expression, statement,
declaration, and type is represented as a `Node`.

```c
struct Node {
    // Tree structure — recursive walks follow these
    Node*     left;      // left child
    Node*     right;     // right child
    NodeList* ninit;     // initialization statements
    NodeList* nbody;     // body (for func, for, if, etc.)
    NodeList* list;      // generic list (arguments, etc.)

    uchar     op;        // operation — one of O* constants (OADD, OIF, etc.)
    uchar     ullman;    // Sethi-Ullman complexity number
    Type*     type;      // type of this node
    Sym*      sym;       // symbol (for names)
    Val       val;       // constant value (for OLITERAL)
    uint      esc;       // escape analysis result
    // ... many more fields
};
```

The `op` field is the most important — it determines what kind of node
this is. There are ~150 operation types defined as an enum starting at
line 436.

**Sethi-Ullman Numbers** (`ullman` field): This is a classic algorithm
from compiler theory that computes the minimum number of registers
needed to evaluate an expression tree.

> **Reference**: Sethi, R. and Ullman, J.D. (1970). "The Generation of
> Optimal Code for Arithmetic Expressions". Journal of the ACM, 17(4),
> pp. 715-728. DOI: 10.1145/321607.321620

### Type (lines 144-207)

Represents a Go type. The `etype` field indicates the kind (one of
`TINT`, `TSTRING`, `TSTRUCT`, `TFUNC`, `TMAP`, `TCHAN`, etc.).

```c
struct Type {
    uchar  etype;      // kind of type (TINT, TSTRING, etc.)
    Type*  type;       // element type (for arrays, channels, maps, pointers)
    Type*  down;       // next field (structs), key type (maps)
    vlong  width;      // size in bytes
    vlong  bound;      // array bound (-1 for slices)
    Sym*   sym;        // type name
    // ... map internals, function signature info, etc.
};
```

Types form a linked list through `down` for struct fields and a tree
through `type` for element types.

### Sym (lines 387-406)

A symbol table entry. Every named entity (variable, function, type,
package) has a `Sym`.

```c
struct Sym {
    char*  name;       // the identifier text
    Node*  def;        // definition: ONAME, OTYPE, OPACK, or OLITERAL
    Pkg*   pkg;        // package this symbol belongs to
    Sym*   link;       // hash chain
    int32  block;      // block number (for scoping)
};
```

Symbols are stored in a **hash table** (`hash[NHASH]` in go.h line 877),
using separate chaining. The hash function is in `subr.c:stringhash()`.

### Pkg (lines 411-422)

Represents an imported package.

```c
struct Pkg {
    char*   name;      // short name ("fmt")
    Strlit* path;      // import path ("fmt" or "encoding/json")
    char*   prefix;    // mangled name for symbol table
    uchar   imported;  // whether export data has been parsed
};
```

---

## Memory Management

The compiler uses a **simple arena allocator** (`mal()` in `subr.c`),
not `malloc()`. Memory is allocated in large hunks (`NHUNK = 50000`
bytes) and never freed during compilation. This is a deliberate design
choice:

1. **Speed**: Arena allocation is just a pointer bump.
2. **Simplicity**: No need to track ownership or lifetimes.
3. **Correctness**: No use-after-free or double-free bugs.

The compiler is designed to compile one package and exit, so memory
leaks don't matter — the OS reclaims everything.

### Reference

- Hanson, D.R. (1990). "Fast Allocation and Deallocation of Memory
  Based on Object Lifetimes". Software: Practice and Experience, 20(1),
  pp. 5-12. (Arena allocation technique.)

---

## What's Next

The following documents cover each component in detail:

| Document | Component |
|----------|-----------|
| [01-data-structures.md](01-data-structures.md) | go.h — All types, enums, and structures |
| [02-lexer.md](02-lexer.md) | lex.c — Lexical analysis and main() |
| [03-parser.md](03-parser.md) | go.y/y.tab.c — YACC grammar and AST construction |
| [04-typechecker.md](04-typechecker.md) | typecheck.c — Type checking |
| [05-declarations.md](05-declarations.md) | dcl.c — Declaration processing |
| [06-escape-analysis.md](06-escape-analysis.md) | esc.c — Escape analysis |
| [07-inlining.md](07-inlining.md) | inl.c — Function inlining |
| [08-order.md](08-order.md) | order.c — Order of evaluation |
| [09-walk.md](09-walk.md) | walk.c — AST lowering |
| [10-codegen.md](10-codegen.md) | gen.c — Code generation |
| [11-backend-amd64.md](11-backend-amd64.md) | 6g/ — amd64 code emission |
| [12-optimizer.md](12-optimizer.md) | popt.c, plive.c, peep.c — Optimization |
| [13-linker.md](13-linker.md) | 6l/, ld/ — Linking |
| [14-runtime.md](14-runtime.md) | runtime/ — Scheduler, GC, memory |
| [15-runtime-assembly.md](15-runtime-assembly.md) | runtime/asm*.s — Assembly |
