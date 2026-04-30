# Go 1.4 Compiler: Core Data Structures (go.h)

**Source file**: `src/cmd/gc/go.h` (1555 lines)

This header file defines every data structure used by the Go compiler.
Understanding `go.h` is **prerequisite** to reading any other compiler
source file.

---

## Table of Contents

1. [Constants and Configuration](#constants-and-configuration)
2. [String Literals (Strlit)](#string-literals)
3. [Multi-Precision Arithmetic (Mpint, Mpflt, Mpcplx)](#multi-precision-arithmetic)
4. [Constant Values (Val)](#constant-values)
5. [Types (Type)](#types)
6. [AST Nodes (Node)](#ast-nodes)
7. [Node Operations (Op Codes)](#node-operations)
8. [Type Kinds (etype)](#type-kinds)
9. [Symbols (Sym)](#symbols)
10. [Packages (Pkg)](#packages)
11. [Escape Analysis Enums](#escape-analysis-enums)
12. [Declaration Contexts](#declaration-contexts)
13. [Bit Vectors (Bits, Bvec)](#bit-vectors)
14. [Magic Division (Magic)](#magic-division)
15. [Labels](#labels)
16. [Global State](#global-state)

---

## Constants and Configuration

**go.h lines 26-68**

```c
enum
{
    NHUNK       = 50000,    // arena allocation chunk size (bytes)
    BUFSIZ      = 8192,     // I/O buffer size
    NSYMB       = 500,      // max symbol name length
    NHASH       = 1024,     // symbol hash table buckets
    STRINGSZ    = 200,      // initial string buffer size
    MAXALIGN    = 7,        // max alignment (2^7 = 128 bytes)
    UINF        = 100,      // "infinity" for Sethi-Ullman numbering

    PRIME1      = 3,        // hash multiplier prime

    AUNK        = 100,      // algorithm kind: unknown
    // ... AMEM, ANOEQ, etc.
};
```

### Algorithm Type Constants (lines 42-63)

These `AMEM*` / `ANOEQ*` constants describe how a type should be
compared for equality. They are used by the runtime for map keys
and interface comparison.

| Constant | Meaning |
|----------|---------|
| `AMEM`   | Compare by raw memory (memcmp) |
| `AMEM0`  | Zero-size type (always equal) |
| `AMEM8`  | 8-byte memcmp |
| `AMEM16` | 16-byte memcmp |
| `AMEM32` | 32-byte memcmp |
| `AMEM64` | 64-byte memcmp |
| `AMEM128`| 128-byte memcmp |
| `ANOEQ`  | Not comparable (contains non-comparable field) |
| `ASTRING` | String comparison |
| `AINTER` | Interface comparison (non-empty) |
| `ANILINTER` | Empty interface comparison |
| `ASLICE` | Slice (not comparable — compiler error) |
| `AFLOAT32` | float32 comparison (NaN-aware) |
| `AFLOAT64` | float64 comparison (NaN-aware) |
| `ACPLX64` | complex64 comparison |
| `ACPLX128` | complex128 comparison |

The `MEMx` and `NOEQx` values "run in parallel" — `ANOEQ8` is the
non-comparable equivalent of `AMEM8` (same size but contains a field
that can't be compared, like a `func` or `map`).

### Special Constants

```c
BADWIDTH = -1000000000   // sentinel: type width not yet computed
MaxStackVarSize = 10*1024*1024  // 10MB — max size for stack allocation
```

---

## String Literals

**go.h lines 76-82**

```c
struct Strlit
{
    int32   len;
    char    s[1];    // variable-length (C flexible array member trick)
};
```

This is the **compiler's internal** representation of string literals,
not the runtime representation. The `s[1]` is the classic C trick for
variable-length data — the struct is allocated with extra bytes after
it, and `s` is accessed beyond index 0.

> **Note**: This is called the "struct hack" or "flexible array member"
> in C. It predates C99's official `char s[]` syntax. See:
> - ISO/IEC 9899:1999 (C99), Section 6.7.2.1, paragraph 16.

The runtime string representation is different (see lines 842-853):
```c
// Runtime string layout:
// struct {
//     uchar *array;   // pointer to bytes
//     int32  nel;     // number of bytes (NOT runes)
// };
```

---

## Multi-Precision Arithmetic

**go.h lines 84-115**

Go constants are evaluated at **arbitrary precision** during
compilation. This is a key feature of Go's constant system — the
expression `1 << 1000` is valid in a constant context, even though
no hardware register can hold the result.

### Mpint — Multi-Precision Integer (lines 95-101)

```c
enum {
    Mpscale = 29,           // bits per word (safely < 32)
    Mpprec  = 16,           // number of words
    // Total precision: 29 * 16 = 464 bits
    Mpbase  = 1L << 29,     // 536870912
    Mpsign  = Mpbase >> 1,  // sign bit within a word
    Mpmask  = Mpbase - 1,   // mask for one word
};

struct Mpint
{
    long    a[Mpprec];  // 16 words of 29 bits each = 464 bits max
    uchar   neg;        // 1 if negative
    uchar   ovf;        // 1 if overflow occurred
};
```

**Why 29 bits per word?** Using 29 bits (not 32) leaves room for
carry bits during arithmetic without overflowing the 32-bit `long`.
When you add two 29-bit numbers, the result fits in 30 bits — still
within a 32-bit word. This avoids the need for special overflow
handling in the inner loops of addition and multiplication.

This is a standard technique in multi-precision arithmetic:

> **Reference**: Knuth, D.E. (1997). "The Art of Computer Programming,
> Volume 2: Seminumerical Algorithms", 3rd Edition. Section 4.3.1
> "The Classical Algorithms". Addison-Wesley.
> - Uses base-β arithmetic where β is chosen to avoid overflow
>   during intermediate computations.

### Mpflt — Multi-Precision Float (lines 103-108)

```c
struct Mpflt
{
    Mpint   val;    // mantissa (significand)
    short   exp;    // binary exponent
};
```

A floating-point number represented as `val * 2^exp`. The mantissa
is a multi-precision integer, giving arbitrary precision. The exponent
is a 16-bit signed integer, allowing exponents from -32768 to 32767.

### Mpcplx — Multi-Precision Complex (lines 110-115)

```c
struct Mpcplx
{
    Mpflt   real;
    Mpflt   imag;
};
```

Simply a pair of multi-precision floats.

### Arithmetic Operations

The multi-precision arithmetic is split across three files:

| File | Operations |
|------|-----------|
| `mparith1.c` | Comparisons, conversions, format printing |
| `mparith2.c` | Integer: add, sub, mul, div, mod, shift, bitwise |
| `mparith3.c` | Float: add, sub, mul, div, normalization |

> **Reference**: The algorithms follow classical multi-precision
> arithmetic as described in:
> - Knuth, D.E. (1997). TAOCP Vol. 2, Section 4.3.
> - GMP (GNU Multiple Precision Arithmetic Library) documentation
>   for algorithmic explanations: https://gmplib.org/manual/

---

## Constant Values

**go.h lines 117-130**

```c
struct Val
{
    short   ctype;      // one of CT* constants
    union {
        short   reg;    // OREGISTER
        short   bval;   // bool value (CTBOOL)
        Mpint*  xval;   // integer (CTINT) or rune (CTRUNE)
        Mpflt*  fval;   // float (CTFLT)
        Mpcplx* cval;   // complex (CTCPLX)
        Strlit* sval;   // string (CTSTR)
    } u;
};
```

A tagged union for constant values. The `ctype` field discriminates:

| ctype | Go Type | C Field |
|-------|---------|---------|
| `CTINT` | untyped int | `u.xval` (Mpint*) |
| `CTRUNE` | untyped rune | `u.xval` (Mpint*) |
| `CTFLT` | untyped float | `u.fval` (Mpflt*) |
| `CTCPLX` | untyped complex | `u.cval` (Mpcplx*) |
| `CTSTR` | untyped string | `u.sval` (Strlit*) |
| `CTBOOL` | untyped bool | `u.bval` (short) |
| `CTNIL` | nil | (no data needed) |

This implements Go's "untyped constant" system. Go constants are
**not** the same as variables — they have arbitrary precision and
don't have a fixed type until they're assigned to a typed variable.

> **Reference**: Go Specification, Section "Constants":
> https://go.dev/ref/spec#Constants
> "Numeric constants represent exact values of arbitrary size and
>  do not overflow."

---

## Types

**go.h lines 144-207**

The `Type` struct represents every Go type in the compiler.

```c
struct Type
{
    uchar   etype;          // kind — one of T* constants (TINT, TSTRING, etc.)
    uchar   nointerface;    // type does not implement any interface
    uchar   noalg;          // no algorithm table (not comparable)
    uchar   chan;            // channel direction (Csend, Crecv, Cboth)
    uchar   trecur;         // recursion detector for type walks
    uchar   embedded;       // is embedded field (for TFIELD)
    uchar   funarg;         // function argument (on TSTRUCT/TFIELD)
    uchar   broke;          // type definition has errors
    uchar   isddd;          // is "..." parameter
    uchar   align;          // required alignment
    uchar   haspointers;    // 0=unknown, 1=no, 2=yes

    Node*   nod;            // canonical OTYPE node
    Type*   orig;           // original type (for type aliases)
    int     lineno;         // source line number

    // Function types (TFUNC)
    int     thistuple;      // number of receiver params
    int     outtuple;       // number of return values
    int     intuple;        // number of input params
    uchar   outnamed;       // return values are named

    Type*   method;         // method set (linked list)
    Type*   xmethod;        // extended method set (includes embedded)

    Sym*    sym;            // type name (nil for anonymous types)
    int32   vargen;         // unique ID for unnamed types

    Node*   nname;          // ONAME node for function types
    vlong   argwid;         // total argument width (bytes)

    // Element/field types
    Type*   type;           // element type (arrays, chans, maps, ptrs),
                            // or actual type (for TFIELD)
    vlong   width;          // size in bytes (or field offset for TFIELD)

    // Struct fields (linked list via 'down')
    Type*   down;           // next field in struct, key type in map
    Type*   outer;          // enclosing struct (for embedded fields)
    Strlit* note;           // struct tag string

    // Arrays
    vlong   bound;          // element count (-1 = slice, -100 = [...]T)

    // Maps (internal representation)
    Type*   bucket;         // hash bucket type
    Type*   hmap;           // map header type
    Type*   hiter;          // map iterator type
    Type*   map;            // back-pointer from internal types

    // Forward references
    NodeList* copyto;       // where to copy type when resolved
};
```

### Type Representation of Go Constructs

| Go Code | etype | key fields |
|---------|-------|-----------|
| `int` | `TINT` | `width=8` (on amd64) |
| `string` | `TSTRING` | `width=16` (ptr+len) |
| `*T` | `TPTR64` | `type=T` |
| `[]T` | `TARRAY` | `type=T, bound=-1` |
| `[5]T` | `TARRAY` | `type=T, bound=5` |
| `map[K]V` | `TMAP` | `down=K, type=V` |
| `chan T` | `TCHAN` | `type=T, chan=Cboth` |
| `func(A)R` | `TFUNC` | `type=TSTRUCT(results)` |
| `struct{...}` | `TSTRUCT` | `type=first_field(TFIELD)` |
| `interface{...}` | `TINTER` | `type=first_method(TFIELD)` |

### Struct Fields as Linked List

Struct fields are represented as a linked list of `Type` nodes with
`etype == TFIELD`, linked through the `down` pointer:

```
TSTRUCT
  └─ type ──► TFIELD{name:"X", type:TINT, width:0}
                └─ down ──► TFIELD{name:"Y", type:TSTRING, width:8}
                               └─ down ──► nil
```

The `width` field on a `TFIELD` stores the **offset** of that field
within the struct, not the field's own size. The field's size is in
`type->width`.

---

## AST Nodes

**go.h lines 245-352**

The `Node` struct is the most important data structure in the compiler.
Every expression, statement, and declaration is a `Node`.

```c
struct Node
{
    // === Tree Structure ===
    // Generic recursive walks should follow these fields
    Node*       left;       // left operand or subject
    Node*       right;      // right operand
    Node*       ntest;      // condition (for loops, if)
    Node*       nincr;      // increment (for loops)
    NodeList*   ninit;      // init statements (for := in if/for)
    NodeList*   nbody;      // body statements
    NodeList*   nelse;      // else block
    NodeList*   list;       // general list (args, elements)
    NodeList*   rlist;      // right-hand list (multi-assign)

    // === Classification ===
    uchar   op;             // *** THE operation type (OADD, OIF, etc.) ***
    uchar   ullman;         // Sethi-Ullman register pressure number
    uchar   addable;        // addressability: 0=not, higher=more addressable
    uchar   etype;          // sub-operation (OASOP), or type kind (OTYPE)
    uchar   bounded;        // bounds check proven unnecessary
    uchar   class;          // storage class: PPARAM, PAUTO, PEXTERN, etc.
    uchar   embedded;       // embedded struct field
    uchar   colas;          // := assignment
    uchar   noescape;       // arguments don't escape
    uchar   nosplit;        // don't split stack
    uchar   builtin;        // built-in function (len, cap, etc.)
    uchar   walkdef;        // definition has been walked
    uchar   typecheck;      // type checking status
    uchar   used;           // variable is used
    uchar   readonly;       // variable is read-only
    uchar   addrtaken;      // address was taken (&x)
    uchar   wrapper;        // is method wrapper
    schar   likely;         // branch prediction hint (-1/0/1)
    uchar   hasbreak;       // contains break statement
    uchar   needzero;       // must zero on function entry
    uchar   needctxt;       // needs closure context register
    uint    esc;            // escape analysis result (EscXXX)
    int     funcdepth;      // nesting depth of function

    // === Type and Identity ===
    Type*   type;           // type of this expression
    Node*   orig;           // original form (before rewrites)

    // === Function-specific ===
    Node*       nname;      // function name node
    Node*       shortname;  // short name for methods
    NodeList*   enter;      // function prologue
    NodeList*   exit;       // function epilogue
    NodeList*   cvars;      // closure variables
    NodeList*   dcl;        // local declarations
    NodeList*   inl;        // inlined body copy
    NodeList*   inldcl;     // inlined declarations copy

    // === Constant Value ===
    Val     val;            // for OLITERAL

    // === Name-specific ===
    Node*   ntype;          // explicit type in declaration
    Node*   defn;           // initializing assignment
    Node*   pack;           // package for import . names
    Node*   curfn;          // enclosing function

    // === Heap-escaped parameters ===
    Node*   heapaddr;       // temp holding heap address
    Node*   stackparam;     // on-stack copy reference
    Node*   alloc;          // allocation call node

    // === Closure references ===
    Node*   outer;          // outer closure param ref
    Node*   closure;        // ONAME/PHEAP <-> ONAME/PPARAMREF

    // === Inlining ===
    Node*   inlvar;         // substitution variable

    // === Package ===
    Pkg*    pkg;            // for OPACK

    // === Composite Literals ===
    InitPlan* initplan;     // initialization plan

    // === Escape Analysis ===
    NodeList* escflowsrc;   // flow(this, src) edges
    NodeList* escretval;    // dummy return values on calls
    int     escloopdepth;   // loop nesting depth for escape

    // === Position ===
    Sym*    sym;            // symbol
    int32   vargen;         // unique name ID
    int32   lineno;         // source line
    int32   endlineno;      // end line (for blocks)
    vlong   xoffset;        // memory offset
    vlong   stkdelta;       // stack compaction adjustment
    int32   iota;           // iota value in const block
    uint32  walkgen;        // walk generation (for graph traversal)
    void*   opt;            // optimization pass data
};
```

### The `ullman` Field — Sethi-Ullman Numbering

The `ullman` field stores the **Sethi-Ullman number**, which estimates
the minimum number of registers needed to evaluate the expression.

The algorithm (from `ullmancalc()` in `subr.c`):
- Leaf nodes (constants, variables): `ullman = 1`
- Function calls: `ullman = UINF` (infinity — always spill)
- Binary operations:
  - If `left.ullman == right.ullman`: result is `left.ullman + 1`
  - Otherwise: result is `max(left.ullman, right.ullman)`

The insight: if both subtrees need the same number of registers, you
need one extra register to hold the result of the first while
computing the second. If they differ, you evaluate the harder one
first and reuse registers.

> **Reference**: Sethi, R. and Ullman, J.D. (1970). "The Generation
> of Optimal Code for Arithmetic Expressions". Journal of the ACM,
> 17(4), pp. 715-728. DOI: 10.1145/321607.321620
>
> This paper proves that their algorithm generates code using the
> minimum number of registers for expression trees, assuming a
> machine with a fixed number of registers and no instruction
> reordering.

### The `walkgen` Field — Graph Traversal Guard

Lines 356-369 explain the `walkgen` pattern:

```c
// To avoid visiting the same nodes twice during a graph walk:
//   if(n->walkgen == walkgen)
//       return;
//   n->walkgen = walkgen;
```

This is a **generation counter** technique — instead of marking nodes
as "visited" and then clearing the marks, you increment a global
counter and compare. This avoids the O(n) clear step.

---

## Node Operations (Op Codes)

**go.h lines 436-604**

The `op` field of a `Node` is one of ~150 constants that classify
every possible AST node. They are organized into categories:

### Names (lines 439-444)

| Op | Example | Meaning |
|----|---------|---------|
| `ONAME` | `x` | Variable, constant, or function name |
| `ONONAME` | `_` | Unnamed parameter |
| `OTYPE` | `int` | Type name |
| `OPACK` | `fmt` | Package name |
| `OLITERAL` | `42`, `"hello"` | Literal constant |

### Arithmetic & Logic (lines 447-467)

| Op | Go Syntax | Notes |
|----|-----------|-------|
| `OADD` | `x + y` | Also used for pointer arithmetic |
| `OSUB` | `x - y` | |
| `OMUL` | `x * y` | |
| `ODIV` | `x / y` | |
| `OMOD` | `x % y` | |
| `OAND` | `x & y` | Bitwise AND |
| `OOR` | `x \| y` | Bitwise OR |
| `OXOR` | `x ^ y` | Bitwise XOR |
| `OANDNOT` | `x &^ y` | Bit clear (Go-specific) |
| `OLSH` | `x << n` | Left shift |
| `ORSH` | `x >> n` | Right shift |
| `OPLUS` | `+x` | Unary plus |
| `OMINUS` | `-y` | Unary minus |
| `OCOM` | `^x` | Bitwise complement |
| `ONOT` | `!b` | Logical NOT |

### Comparison (lines 498-503)

| Op | Go | Notes |
|----|----|-------|
| `OEQ` | `==` | |
| `ONE` | `!=` | |
| `OLT` | `<` | |
| `OLE` | `<=` | |
| `OGE` | `>=` | |
| `OGT` | `>` | |

### Assignment (lines 460-466)

| Op | Go | Notes |
|----|----|-------|
| `OAS` | `x = y` | Simple assignment |
| `OAS2` | `x, y = a, b` | Multi-value assignment |
| `OAS2FUNC` | `x, y = f()` | Multi-return function |
| `OAS2RECV` | `x, ok = <-ch` | Channel receive with ok |
| `OAS2MAPR` | `x, ok = m[k]` | Map lookup with ok |
| `OAS2DOTTYPE` | `x, ok = i.(T)` | Type assertion with ok |
| `OASOP` | `x += y` | Compound assignment |

### Calls (lines 467-475)

| Op | Go | Meaning |
|----|----|---------|
| `OCALL` | `f()` | Generic call (before type-checking) |
| `OCALLFUNC` | `f()` | Direct function call |
| `OCALLMETH` | `t.Method()` | Method call on concrete type |
| `OCALLINTER` | `i.Method()` | Method call on interface |
| `OCALLPART` | `t.Method` | Method value (without ()) |

### Control Flow (lines 548-567)

| Op | Go | Notes |
|----|----|-------|
| `OIF` | `if` | `ntest`=cond, `nbody`=then, `nelse`=else |
| `OFOR` | `for` | `ntest`=cond, `nincr`=post, `nbody`=body |
| `ORANGE` | `range` | `list`=range expr, `nbody`=body |
| `OSWITCH` | `switch` | `ntest`=tag, `list`=cases |
| `OSELECT` | `select` | `list`=cases |
| `ORETURN` | `return` | `list`=return values |
| `OPROC` | `go` | `left`=call expression |
| `ODEFER` | `defer` | `left`=call expression |
| `OBREAK` | `break` | |
| `OCONTINUE` | `continue` | |
| `OGOTO` | `goto` | |
| `OLABEL` | `label:` | |
| `OFALL` | `fallthrough` | |

### Special Operations (lines 576-603)

| Op | Meaning | Used By |
|----|---------|---------|
| `OINLCALL` | Inlined function call | `inl.c` |
| `OEFACE` | Empty interface construction | `walk.c` |
| `OITAB` | Get itable from interface | `walk.c` |
| `OSPTR` | Get data ptr from slice/string | `walk.c` |
| `OCLOSUREVAR` | Closure variable reference | `closure.c` |
| `OCHECKNIL` | Nil pointer check | `gen.c` |
| `OVARKILL` | Mark variable as dead | `plive.c` |
| `OHMUL` | High multiplication (128-bit result) | `6g/` |
| `OLROT` | Left rotate | `6g/` |
| `ORETJMP` | Tail return to other func | `6g/` |

---

## Type Kinds (etype)

**go.h lines 607-650**

| etype | Go Type | Notes |
|-------|---------|-------|
| `TINT8` - `TUINT64` | `int8` through `uint64` | Sized integers |
| `TINT`, `TUINT`, `TUINTPTR` | `int`, `uint`, `uintptr` | Platform-sized |
| `TFLOAT32`, `TFLOAT64` | `float32`, `float64` | IEEE 754 |
| `TCOMPLEX64`, `TCOMPLEX128` | `complex64`, `complex128` | |
| `TBOOL` | `bool` | |
| `TSTRING` | `string` | ptr + len |
| `TPTR32`, `TPTR64` | `*T` | Pointer (size depends on arch) |
| `TARRAY` | `[]T` or `[N]T` | `bound` distinguishes |
| `TMAP` | `map[K]V` | |
| `TCHAN` | `chan T` | |
| `TSTRUCT` | `struct{...}` | |
| `TINTER` | `interface{...}` | |
| `TFUNC` | `func(...)...` | |
| `TFIELD` | (struct/func field) | Not a real Go type |
| `TFORW` | (forward reference) | Placeholder during parsing |
| `TANY` | (any type) | For bootstrapping |
| `TUNSAFEPTR` | `unsafe.Pointer` | |
| `TIDEAL` | (untyped numeric) | Constants before type assignment |
| `TNIL` | (nil) | |
| `TBLANK` | `_` | |

### Pseudo-types

| etype | Purpose |
|-------|---------|
| `TFUNCARGS` | Frame layout for function arguments |
| `TCHANARGS` | Channel operation arguments |
| `TINTERMETH` | Intermediate method representation |

---

## Symbols

**go.h lines 387-406**

```c
struct Sym
{
    ushort  lexical;    // token type from lexer
    uchar   flags;      // SymExport, SymPackage, etc.
    uchar   sym;        // Huffman encoding for object file
    Sym*    link;       // hash chain (separate chaining)
    int32   npkg;       // number of packages defining this name
    uint32  uniqgen;    // unique ID generator

    Pkg*    importdef;  // package where definition was imported from
    Pkg*    pkg;        // owning package
    char*   name;       // the identifier string
    Node*   def;        // definition: ONAME, OTYPE, OPACK, or OLITERAL
    Label*  label;      // goto label (ephemeral)
    int32   block;      // block number for scoping
    int32   lastlineno; // last declaration line (for diagnostics)
    Pkg*    origpkg;    // original package (for dot imports)
    LSym*   lsym;       // linker symbol
};
```

### Symbol Table Implementation

The symbol table uses **hash table with separate chaining**:
- Hash table: `hash[NHASH]` (1024 buckets) — go.h line 877
- Hash function: `stringhash()` in `subr.c`
- Collision resolution: linked list via `Sym.link`
- Lookup: `lookup()` or `pkglookup()` in `subr.c`

Each symbol is scoped to a package. The lookup `pkglookup(name, pkg)`
finds the symbol named `name` in package `pkg`. The shorthand
`lookup(name)` looks in the current package.

### Symbol Flags

```c
enum {
    SymExport   = 1<<0,    // marked for export
    SymPackage  = 1<<1,    // is a package name
    SymExported = 1<<2,    // already written to export data
    SymUniq     = 1<<3,    // unique (for type switch temps)
    SymSiggen   = 1<<4,    // type signature generated
};
```

### Block Scoping

The `block` field implements **lexical scoping**. Each `{...}` block
increments `blockgen` (go.h line 951). When a symbol is declared,
its `block` is set to the current block number. When the block ends
(`popdcl()` in `dcl.c`), symbols declared in that block are popped
off the declaration stack.

---

## Packages

**go.h lines 411-422**

```c
struct Pkg
{
    char*   name;       // short name ("fmt")
    Strlit* path;       // import path ("encoding/json")
    Sym*    pathsym;    // symbol for path string
    char*   prefix;     // mangled path for symbol table
    Pkg*    link;       // hash chain
    uchar   imported;   // export data parsed
    char    exported;   // import line written to export
    char    direct;     // directly imported (not transitively)
    char    safe;       // package is marked safe
};
```

Packages are stored in a hash table `phash[128]` (go.h line 893),
looked up by `mkpkg()` in `subr.c`.

---

## Escape Analysis Enums

**go.h lines 231-243**

```c
enum {
    EscUnknown,         // 0 — not yet analyzed
    EscHeap,            // 1 — escapes to heap (must heap-allocate)
    EscScope,           // 2 — stays in declaring scope
    EscNone,            // 3 — does not escape at all
    EscReturn,          // 4 — returned from function
    EscNever,           // 5 — never escapes (compiler builtin)
    EscBits = 3,        // number of bits for escape level
    EscMask = 7,        // mask for escape level
    EscContentEscapes = 1<<3,  // indirect value escapes
    EscReturnBits = 4,  // starting bit for return tracking
};
```

The escape analysis (in `esc.c`) determines whether a variable's
lifetime exceeds its declaring function's stack frame. If `esc ==
EscHeap`, the variable must be heap-allocated.

See [06-escape-analysis.md](06-escape-analysis.md) for full details.

---

## Declaration Contexts

**go.h lines 676-690**

```c
enum {
    Pxxx,
    PEXTERN,    // global variable
    PAUTO,      // local variable (stack)
    PPARAM,     // function input parameter
    PPARAMOUT,  // function return value
    PPARAMREF,  // closure variable reference
    PFUNC,      // global function
    PDISCARD,   // discard during duplicate import parse
    PHEAP = 1<<7,  // bit flag: variable escaped to heap
};
```

The `class` field of a `Node` uses these values. `PHEAP` is an
**additional bit** that can be OR'd with any class — for example,
`PPARAM | PHEAP` means "parameter that escaped to the heap".

---

## Bit Vectors

**go.h lines 707-723**

Two different bit vector implementations are used:

### Bits — Fixed-Size (lines 707-714)

```c
#define BITS  5
#define NVAR  (BITS*sizeof(uint32)*8)  // = 160 variables max

struct Bits {
    uint32  b[BITS];    // 5 * 32 = 160 bits
};
```

Used by the **register allocator** (`reg.c`) to track which variables
are live at each program point. Limited to 160 variables — if a
function has more than 160 variables, the register allocator ignores
the extras.

### Bvec — Dynamic-Size (lines 718-722)

```c
struct Bvec {
    int32   n;      // number of bits
    uint32  b[];    // flexible array (C99)
};
```

Used by **liveness analysis** (`plive.c`) for arbitrary-size bit
vectors. Supports the full set of operations: and, or, not, andnot.

---

## Magic Division

**go.h lines 792-808**

```c
struct Magic {
    int     w;      // input: bit width
    int     s;      // output: shift amount
    int     bad;    // output: 1 if failed

    // Signed division
    int64   sd;     // input: divisor
    int64   sm;     // output: magic multiplier

    // Unsigned division
    uint64  ud;     // input: divisor
    uint64  um;     // output: magic multiplier
    int     ua;     // output: add flag
};
```

This implements the **"magic number" division optimization** — replacing
division by a constant with multiplication by a magic constant plus a
shift. This is much faster because multiplication is ~3-4x faster than
division on modern CPUs.

For example, `x / 7` becomes approximately `(x * 0x24924925) >> 33`.

> **Reference**: Granlund, T. and Montgomery, P.L. (1994). "Division
> by Invariant Integers using Multiplication". Proceedings of the
> ACM SIGPLAN 1994 Conference on Programming Language Design and
> Implementation (PLDI), pp. 61-72. DOI: 10.1145/178243.178249
>
> Also: Warren, H.S. Jr. (2012). "Hacker's Delight", 2nd Edition.
> Chapter 10: "Integer Division by Constants". Addison-Wesley.

The functions `smagic()` and `umagic()` in `subr.c` compute these
magic constants for signed and unsigned division respectively.

---

## Labels

**go.h lines 810-823**

```c
struct Label {
    uchar   used;       // label was referenced by goto/break/continue
    Sym*    sym;        // label name
    Node*   def;        // labeled statement
    NodeList* use;      // list of gotos targeting this label
    Label*  link;       // next label in function

    // Code generation
    Prog*   gotopc;     // unresolved goto instructions
    Prog*   labelpc;    // instruction at label position
    Prog*   breakpc;    // break target instruction
    Prog*   continpc;   // continue target instruction
};
```

Labels are used for `goto`, `break`, `continue`, and labeled
statements (`outer:` for). The `gotopc` field holds a list of
forward goto instructions that need to be patched when the label's
position is known.

---

## Global State

**go.h lines 826-996**

The compiler has extensive global state (a design choice of its era).
Key globals:

### Runtime Layout Constants (lines 837-853)

```c
EXTERN int  Array_array;    // offset of array/slice data pointer
EXTERN int  Array_nel;      // offset of length
EXTERN int  Array_cap;      // offset of capacity
EXTERN int  sizeof_Array;   // total slice header size
EXTERN int  sizeof_String;  // total string header size
```

These must match the runtime's memory layout exactly. They are
initialized in `betypeinit()` based on the target architecture.

### Compilation State (lines 855-996)

| Global | Purpose |
|--------|---------|
| `curio` | Current I/O state for lexer |
| `lineno` | Current source line number |
| `nerrors` | Error count |
| `hash[NHASH]` | Symbol table |
| `localpkg` | Package being compiled |
| `xtop` | Top-level declaration list |
| `curfn` | Function currently being compiled |
| `widthptr` | Pointer width (4 or 8) |
| `widthint` | Int width (4 or 8) |
| `thechar` | Architecture character ('5', '6', '8') |
| `stksize` | Current function's stack frame size |
| `debug[]` | Debug flag array (indexed by character) |

### The EXTERN Macro Pattern

```c
#ifndef EXTERN
#define EXTERN extern
#endif
```

This is a classic C pattern. In the one file that `#defines EXTERN`
to empty before including `go.h`, the variables are **defined**.
In all other files, they are **declared** as `extern`. This avoids
multiple definition errors while keeping all globals in one place.

---

## Function Declarations

**go.h lines 997-1555**

The remainder of `go.h` consists of function declarations organized
by source file. Each section header (`/* align.c */`, `/* bits.c */`,
etc.) lists the public functions from that file.

This serves as a **table of contents** for the entire compiler.
