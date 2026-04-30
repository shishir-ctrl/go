# Go 1.4 Compiler: amd64 Backend (6g)

**Source directory**: `src/cmd/6g/` (8180 lines total)

The amd64 backend converts the lowered AST (after walk.c) into
amd64 machine instructions. The name "6g" comes from Plan 9's
convention where `6` = amd64.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Code Generation (cgen.c)](#code-generation)
3. [General Code Generation (ggen.c)](#general-code-generation)
4. [Code Generation Subroutines (gsubr.c)](#code-generation-subroutines)
5. [Register Allocation (reg.c)](#register-allocation)
6. [Peephole Optimizer (peep.c)](#peephole-optimizer)
7. [Architecture Constants (galign.c)](#architecture-constants)
8. [amd64 Register Set](#amd64-register-set)

---

## Architecture Overview

The backend processes each function through this pipeline:

```
AST (after walk)
    │
    ▼
compile()     pgen.c — Entry point for function compilation
    │
    ├── cgen()    cgen.c — Recursive expression code generation
    │               Emits intermediate Prog instructions
    │
    ├── regopt()  reg.c  — Graph-coloring register allocation
    │               Assigns registers to variables
    │
    └── peep()    peep.c — Peephole optimization
                    Removes redundant instructions
    │
    ▼
Object File (.6)
```

### Heritage

The code is derived from the **Inferno** operating system's compiler:

```c
// Derived from Inferno utils/6c/reg.c
// http://code.google.com/p/inferno-os/source/browse/utils/6c/reg.c
```

Inferno was a successor to Plan 9, also from Bell Labs. The register
allocator and code generator trace their lineage to Ken Thompson's
original Plan 9 compilers.

> **Reference**: Winterbottom, P. and Pike, R. (1997). "The Design
> of the Inferno Virtual Machine". Bell Labs Technical Report.

---

## Code Generation (cgen.c)

**1732 lines**

`cgen(Node *n, Node *res)` is the main code generation function.
It recursively generates code to compute expression `n` and store
the result in `res`.

### The cgen() Strategy

```c
void cgen(Node *n, Node *res) {
    // 1. Handle special cases (slices, eface, fat types)
    // 2. If both n and res need many registers,
    //    use a temp to avoid register pressure
    // 3. Handle fat types (structs) with block copy (sgen)
    // 4. Generate code based on n->op
}
```

### Key Decisions

**Ullman number usage** (line 59):
```c
if(n->ullman >= UINF) {
    // Expression needs infinite registers (has call)
    // Spill to temp first
    if(res->ullman >= UINF) {
        tempname(&n1, n->type);
        cgen(n, &n1);
        cgen(&n1, res);
    }
}
```

**Fat types** (line 70):
```c
if(isfat(n->type)) {
    sgen(n, res, n->type->width);  // structure copy
}
```

**The main switch** handles each operation by emitting `Prog`
instructions via helper functions like:
- `gins(op, from, to)` — emit instruction
- `gmove(from, to)` — emit move instruction
- `regalloc(&n, type, hint)` — allocate a register
- `regfree(&n)` — free a register

### Instruction Emission

Instructions are represented as `Prog` structures (linked list):

```c
Prog *p = gins(AMOVQ, &src, &dst);
// Generates: MOVQ src, dst
```

The `Prog` struct contains:
- `as`: instruction opcode (AMOVQ, AADDQ, etc.)
- `from`: source operand
- `to`: destination operand
- `link`: next instruction

---

## General Code Generation (ggen.c)

**1228 lines**

Contains code generation for control flow and function setup:

### compile() — Function Entry Point

```c
void compile(Node *fn) {
    // 1. Set up function prologue
    // 2. Generate code for function body: gen(fn)
    // 3. Set up function epilogue (return)
    // 4. Run register allocation: regopt()
    // 5. Run peephole optimization: peep()
    // 6. Output the instruction list
}
```

### gen() — Statement Code Generation

```c
void gen(Node *n) {
    switch(n->op) {
    case OIF:     // if statement
    case OFOR:    // for loop
    case OSWITCH: // switch
    case ORETURN: // return
    case OPROC:   // go statement
    case ODEFER:  // defer statement
    // etc.
    }
}
```

### bgen() — Boolean Code Generation

```c
void bgen(Node *n, int true, int likely, Prog *to) {
    // Generate code that jumps to 'to' if n is true/false
    // 'likely' is a branch prediction hint
}
```

This is a separate function because boolean expressions in
conditions (`if x > 0`) can generate more efficient code by
directly emitting conditional jumps rather than computing a
boolean value and testing it.

### Function Prologue/Epilogue

```asm
# Prologue (stack frame setup):
SUBQ    $framesize, SP     # allocate stack frame
MOVQ    BP, -8(SP)         # save base pointer
LEAQ    -8(SP), BP         # set new base pointer

# Epilogue (return):
MOVQ    -8(SP), BP         # restore base pointer
ADDQ    $framesize, SP     # deallocate stack frame
RET
```

---

## Code Generation Subroutines (gsubr.c)

**2296 lines**

The largest file in the backend. Contains:

### Register Management

```c
void regalloc(Node *n, Type *t, Node *o)
    // Allocate a register for type t
    // Use register hint from o if available

void regfree(Node *n)
    // Free the register held by n
```

### Instruction Emission

```c
Prog* gins(int as, Node *f, Node *t)
    // Emit instruction: as f, t
    // Returns the Prog node

Prog* ginscall(Node *f, int proc)
    // Emit function call
    // proc: 0=normal, 1=go, 2=defer

void gmove(Node *f, Node *t)
    // Emit move from f to t
    // Handles type conversions (int→float, widening, etc.)
```

### gmove() — The Move Generator

`gmove()` is particularly important — it handles all type conversions
via moves. It contains a large table mapping (source type, dest type)
pairs to the correct instruction sequence.

For example:
- `int32 → int64`: `MOVLQSX` (sign-extend)
- `uint32 → int64`: `MOVLQZX` (zero-extend)
- `int64 → float64`: `CVTSQ2SD` (integer to SSE double)
- `float64 → int64`: `CVTTSD2SQ` (SSE double to integer, truncate)

---

## Register Allocation (reg.c)

**1315 lines**

The register allocator uses a **graph-coloring** approach derived
from Plan 9's original allocator (which predates Chaitin's published
work and uses a simpler variant).

### amd64 Register Set

```c
static char* regname[] = {
    ".AX", ".CX", ".DX", ".BX",
    ".SP", ".BP", ".SI", ".DI",
    ".R8", ".R9", ".R10", ".R11",
    ".R12", ".R13", ".R14", ".R15",
    // + 16 SSE registers (X0-X15)
};
#define NREGVAR 32    // 16 general + 16 floating
```

### Algorithm

1. **Build flow graph**: Create a graph of basic blocks from
   the instruction list.

2. **Compute liveness**: For each program point, determine which
   variables are live (needed later).

3. **Build interference graph**: Two variables that are live at
   the same point interfere and cannot share a register.

4. **Color the graph**: Assign registers to variables. Variables
   that don't interfere can share a register.

5. **Spill**: If a variable can't be colored (not enough registers),
   spill it to the stack.

### Data Flow Analysis

The register allocator uses **iterative data flow analysis** to
compute liveness:

```c
// Iterate until fixed point
do {
    changed = 0;
    for(each block in reverse postorder) {
        // live_out = union of live_in of all successors
        // live_in = (live_out - def) | use
        if(live_in changed)
            changed = 1;
    }
} while(changed);
```

> **Reference**: Kildall, G.A. (1973). "A Unified Approach to
> Global Program Optimization". Proceedings of the 1st ACM SIGACT-
> SIGPLAN Symposium on Principles of Programming Languages,
> pp. 194-206. DOI: 10.1145/512927.512945
>
> Chaitin, G.J. (1982). "Register Allocation & Spilling via Graph
> Coloring". Proceedings of the ACM SIGPLAN 1982 Symposium on
> Compiler Construction, pp. 98-105. DOI: 10.1145/800230.806984

---

## Peephole Optimizer (peep.c)

**990 lines**

The peephole optimizer examines small windows of instructions and
replaces suboptimal sequences with better ones.

### Common Optimizations

| Before | After | Reason |
|--------|-------|--------|
| `MOVQ X, Y; MOVQ Y, X` | `MOVQ X, Y` | Redundant copy-back |
| `MOVQ X, R; op R, Y` | `op X, Y` | Eliminate temp register |
| `MOVQ $0, X` | `XORQ X, X` | Shorter encoding |
| `CMPQ X, $0; JEQ L` | `TESTQ X, X; JEQ L` | Shorter encoding |
| `MOVQ X, R; MOVQ R, Y` | `MOVQ X, Y` | Dead register elimination |

> **Reference**: McKeeman, W.M. (1965). "Peephole Optimization".
> Communications of the ACM, 8(7), pp. 443-444.
> DOI: 10.1145/364995.365000

---

## Architecture Constants (galign.c)

**66 lines**

Sets architecture-specific constants:

```c
void betypeinit(void) {
    widthptr = 8;        // pointer width: 8 bytes
    widthint = 8;        // int width: 8 bytes
    widthreg = 8;        // register width: 8 bytes

    // Calling convention:
    // All arguments passed on stack (Go 1.4 ABI)
    // No register-based calling convention
}
```

### Go 1.4 Calling Convention

Go 1.4 uses an **all-stack** calling convention:
- All function arguments are passed on the stack
- All return values are on the stack
- No registers are used for argument passing

This is unusual compared to the System V AMD64 ABI (which uses
RDI, RSI, RDX, RCX, R8, R9 for the first 6 args). Go chose this
for simplicity and to support goroutine stack copying.

> **Note**: Go 1.17 later introduced register-based calling convention
> for improved performance, but Go 1.4 keeps it simple.

---

## References

- Thompson, K. and Pike, R. (2002). "Plan 9 C Compilers".
  https://9p.io/sys/doc/compiler.html

- Chaitin, G.J. (1982). "Register Allocation & Spilling via Graph
  Coloring". ACM SIGPLAN, pp. 98-105. DOI: 10.1145/800230.806984

- Briggs, P., Cooper, K.D., Torczon, L. (1994). "Improvements to
  Graph Coloring Register Allocation". ACM TOPLAS, 16(3), pp. 428-455.
  DOI: 10.1145/177492.177575

- McKeeman, W.M. (1965). "Peephole Optimization". CACM, 8(7),
  pp. 443-444. DOI: 10.1145/364995.365000

- Sethi, R. and Ullman, J.D. (1970). "The Generation of Optimal
  Code for Arithmetic Expressions". JACM, 17(4), pp. 715-728.
  DOI: 10.1145/321607.321620

- Kildall, G.A. (1973). "A Unified Approach to Global Program
  Optimization". POPL '73, pp. 194-206.
  DOI: 10.1145/512927.512945

- Granlund, T. and Montgomery, P.L. (1994). "Division by Invariant
  Integers using Multiplication". PLDI '94, pp. 61-72.
  DOI: 10.1145/178243.178249

- Intel Corporation (2024). "Intel 64 and IA-32 Architectures
  Software Developer's Manual". Volumes 1-3.
  https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
