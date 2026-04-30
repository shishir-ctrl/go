# Go 1.4 Compiler: Type Checker (typecheck.c)

**Source file**: `src/cmd/gc/typecheck.c` (3582 lines)

The type checker is the **largest and most complex** component of the
compiler frontend. It walks every AST node, assigns types, evaluates
constant expressions, resolves names, checks operator validity, and
rewrites certain node operations to more specific forms.

---

## Table of Contents

1. [Overview and Entry Point](#overview-and-entry-point)
2. [The typecheck() Function](#the-typecheck-function)
3. [Context Flags (top parameter)](#context-flags)
4. [The Big Switch: typecheck1()](#the-big-switch)
5. [Name Resolution](#name-resolution)
6. [Type Construction](#type-construction)
7. [Arithmetic and Comparison](#arithmetic-and-comparison)
8. [Dot Expressions (Field and Method Access)](#dot-expressions)
9. [Index and Slice Operations](#index-and-slice-operations)
10. [Channel Operations](#channel-operations)
11. [Function Calls](#function-calls)
12. [Built-in Functions](#built-in-functions)
13. [Type Assertions](#type-assertions)
14. [Composite Literals](#composite-literals)
15. [Assignments](#assignments)
16. [Constant Evaluation](#constant-evaluation)
17. [Error Handling Patterns](#error-handling-patterns)

---

## Overview and Entry Point

**Lines 1-11** describe the type checker's responsibilities:

```c
/*
 * type check the whole tree of an expression.
 * calculates expression types.
 * evaluates compile time constants.
 * marks variables that escape the local frame.
 * rewrites n->op to be more specific in some cases.
 */
```

The type checker is called from `lex.c` in three phases:

```c
// Phase 1: const, type, and func signatures (lex.c:429)
for(l=xtop; l; l=l->next)
    if(l->n->op != ODCL && l->n->op != OAS)
        typecheck(&l->n, Etop);

// Phase 2: Variable assignments (lex.c:435)
for(l=xtop; l; l=l->next)
    if(l->n->op == ODCL || l->n->op == OAS)
        typecheck(&l->n, Etop);

// Phase 3: Function bodies (lex.c:441)
for(l=xtop; l; l=l->next)
    if(l->n->op == ODCLFUNC || l->n->op == OCLOSURE)
        typechecklist(l->n->nbody, Etop);
```

---

## The typecheck() Function

**Lines 139-233**

```c
Node* typecheck(Node **np, int top)
```

The function takes a **pointer to a pointer** (`Node **np`) because
it may replace the node entirely (e.g., resolving names, constant
folding, or converting node types).

### Algorithm

1. **Skip parentheses** (line 159): `while(n->op == OPAREN) n = n->left`
2. **Resolve names** (line 163): `n = resolve(n)` — replaces `ONONAME`
   with the actual definition and substitutes `iota` values
3. **Check for already done** (line 169): `if(n->typecheck == 1)` skip
4. **Detect loops** (line 182): `if(n->typecheck == 2)` — we're in a
   cycle (e.g., `const x = x + 1`). Report error.
5. **Mark in progress** (line 210): `n->typecheck = 2`
6. **Push onto stack** (lines 212-219): Maintain a stack for error reporting
7. **Do the work** (line 221): `typecheck1(&n, top)`
8. **Mark done** (line 223): `n->typecheck = 1`
9. **Pop stack** (lines 225-229): Clean up

### Loop Detection

The `typecheck` field has three states:
- `0` = not yet type-checked
- `1` = successfully type-checked
- `2` = currently being type-checked (in progress)

If we encounter `typecheck == 2`, we've found a cycle. The function
prints the dependency chain using `sprint_depchain()` (line 118).

---

## Context Flags

**go.h lines 693-705**

The `top` parameter is a bitmask describing the context in which
the expression appears:

```c
enum {
    Etop      = 1<<1,  // evaluated at statement level (value discarded)
    Erv       = 1<<2,  // evaluated in value context (result used)
    Etype     = 1<<3,  // must be a type
    Ecall     = 1<<4,  // call expressions are ok
    Efnstruct = 1<<5,  // multi-value function returns ok
    Eiota     = 1<<6,  // iota is ok
    Easgn     = 1<<7,  // left-hand side of assignment
    Eindir    = 1<<8,  // through an indirection
    Eaddr     = 1<<9,  // taking address
    Eproc     = 1<<10, // inside a go statement
    Ecomplit  = 1<<11, // type in composite literal
};
```

These flags drive key decisions:

| Context | Meaning | Example |
|---------|---------|---------|
| `Etop` | Value is discarded | `f()` as statement |
| `Erv` | Value is used | `x := f()` |
| `Etype` | Must be a type | `var x T` |
| `Ecall` | Function call ok | `len(x)` |
| `Easgn` | Assignment target | `x = 5` |
| `Eaddr` | Taking address | `&x` |
| `Ecomplit` | Composite literal | `T{...}` |

### Example: How Context Affects Type Checking

```go
x       // ONAME: if Erv → ok; if Easgn → "_ is not used" suppressed
_       // ONAME: if Erv → error "cannot use _ as value"
fmt     // OPACK: always error "use of package without selector"
len     // ONAME with builtin: if Ecall → ok; else → error
```

---

## The Big Switch: typecheck1()

**Lines 304-end (3280+ lines)**

`typecheck1()` is a **massive switch statement** on `n->op` with
~100 cases. It's organized into sections:

### Section: Names (lines 341-376)

```c
case OLITERAL:  ok |= Erv; goto ret;    // literals are values
case ONONAME:   ok |= Erv; goto ret;    // unresolved names
case ONAME:                              // variables/functions
    if(n->etype != 0) { ok |= Ecall; }  // builtins need Ecall
    if(!Easgn && isblank(n))
        yyerror("cannot use _ as value");
    n->used = 1;                          // mark as used
    ok |= Erv; goto ret;
case OPACK:
    yyerror("use of package %S without selector");
```

### Section: Type Construction (lines 381-488)

Handles `OTARRAY`, `OTMAP`, `OTCHAN`, `OTSTRUCT`, `OTINTER`, `OTFUNC`.
Each constructs the appropriate `Type` structure:

```c
case OTARRAY:    // []T or [N]T
    t = typ(TARRAY);
    if(l == nil)
        t->bound = -1;         // slice
    else if(l->op == ODDD)
        t->bound = -100;       // [...]T, filled later
    else {
        // array: evaluate bound as constant
        t->bound = mpgetfix(v.u.xval);
        if(t->bound < 0)
            yyerror("array bound must be non-negative");
    }
    t->type = r->type;         // element type
    n->op = OTYPE;             // rewrite: OTARRAY → OTYPE
    n->type = t;
```

**Key pattern**: Type syntax nodes (`OTARRAY`, `OTMAP`, etc.) are
**rewritten** to `OTYPE` nodes after type checking. The original op
was just for parsing; after type checking, the type information lives
in `n->type`.

### Section: Arithmetic (lines 522-708)

All binary arithmetic and comparison operators share the `arith:` label:

```c
case OADD: case OSUB: case OMUL: case ODIV: case OMOD:
case OAND: case OOR: case OXOR: case OANDNOT:
case OLSH: case ORSH:
case OEQ: case ONE: case OLT: case OLE: case OGT: case OGE:
case OANDAND: case OOROR:
    l = typecheck(&n->left, Erv);
    r = typecheck(&n->right, Erv);
    // fall through to arith:

arith:
    if(op == OLSH || op == ORSH) goto shift;
    defaultlit2(&l, &r, 0);     // resolve untyped constants
    // ... validate types, check operator applicability
```

**Type compatibility rules** (lines 572-601):

For comparison operators, Go allows comparing values of different
types if one is **assignable** to the other. The type checker inserts
an implicit conversion:

```c
if(iscmp[n->op] && !eqtype(l->type, r->type)) {
    if((aop = assignop(l->type, r->type, nil)) != 0) {
        l = nod(aop, l, N);   // insert conversion
        l->type = r->type;
    }
}
```

**Special restrictions** (lines 617-636):

Go restricts what types can be compared:
- Fixed arrays: only if element type is comparable
- Slices: only compared to `nil`
- Maps: only compared to `nil`
- Functions: only compared to `nil`
- Structs: only if all fields are comparable

**String operations** (lines 651-669):

String `+` is rewritten to `OADDSTR` with a flattened list:
```c
if(et == TSTRING && n->op == OADD) {
    n->op = OADDSTR;
    // flatten: "a" + "b" + "c" → OADDSTR{list: ["a", "b", "c"]}
}
```

String comparison is rewritten to `OCMPSTR`:
```c
if(et == TSTRING && iscmp[n->op]) {
    n->etype = n->op;   // save comparison type
    n->op = OCMPSTR;    // rewrite to string comparison
}
```

**Interface comparison** (lines 670-681):

Interface `==` is rewritten to `OCMPIFACE` and nil comparisons
are normalized (nil always on the right).

**Shift operations** (lines 692-708):

Shifts have special rules:
- Right operand must be unsigned integer
- Left operand must be integer
- Left operand keeps its original type (no defaultlit)

### Section: Dot Expressions (lines 754-824)

**Lines 754-824**: Field and method access (`x.Field`, `x.Method()`).

```c
case OXDOT:
    n = adddot(n);      // insert implicit field traversal
    n->op = ODOT;        // rewrite OXDOT → ODOT
    // fall through

case ODOT:
    typecheck(&n->left, Erv|Etype);
    // if left is a type (not a value), it's a method expression
    if(n->left->op == OTYPE) {
        looktypedot(n, t, 0);  // T.Method
        n->op = ONAME;
    }
    // auto-dereference pointers
    if(isptr[t->etype] && t->type->etype != TINTER) {
        t = t->type;
        n->op = ODOTPTR;   // x.Field → (*x).Field
    }
    lookdot(n, t, 0);      // find field or method
```

**Key transformations**:
- `OXDOT` → `ODOT`: Generic dot becomes specific
- `ODOT` → `ODOTPTR`: Pointer auto-dereference
- `ODOT` → `ODOTMETH`: Concrete method call
- `ODOT` → `ODOTINTER`: Interface method call

The `lookdot()` function (defined later in the file) searches the
type's fields and methods, including **embedded types**. The `adddot()`
function in `subr.c` handles multi-level embedding: if `x.A.B.Field`
needs to be accessed as `x.Field`, `adddot` inserts the intermediate
`A.B` accesses.

### Section: Index and Slice (lines 864-1000+)

**Index** (`a[i]`, line 864):

```c
case OINDEX:
    switch(t->etype) {
    case TSTRING:
        n->type = types[TUINT8];    // string index → byte
    case TARRAY:
        n->type = t->type;          // array/slice index → element type
        // compile-time bounds checking:
        if(isconst(n->right, CTINT)) {
            if(x < 0)
                yyerror("invalid index (must be non-negative)");
            if(isfixedarray(t) && x >= t->bound)
                yyerror("invalid index (out of bounds)");
        }
    case TMAP:
        n->op = OINDEXMAP;         // rewrite: OINDEX → OINDEXMAP
        n->type = t->type;         // map index → value type
    }
```

**Slice** (`a[i:j]`, line 966):

```c
case OSLICE:
    if(isfixedarray(l->type)) {
        // array slice: take address implicitly
        n->left = nod(OADDR, n->left, N);
        n->left->implicit = 1;
    }
    if(istype(t, TSTRING))     n->op = OSLICESTR;
    if(isptr && isfixedarray)  n->op = OSLICEARR;
    if(isslice(t))             // stays OSLICE
```

### Section: Channel Operations (lines 922-964)

**Receive** (`<-ch`):
```c
case ORECV:
    if(t->etype != TCHAN)
        yyerror("receive from non-chan type");
    if(!(t->chan & Crecv))
        yyerror("receive from send-only type");
    n->type = t->type;     // result is channel's element type
```

**Send** (`ch <- v`):
```c
case OSEND:
    if(t->etype != TCHAN)
        yyerror("send to non-chan type");
    if(!(t->chan & Csend))
        yyerror("send to receive-only type");
    n->right = assignconv(r, l->type->type, "send");
    n->type = T;           // send has no result type
```

---

## Function Calls

The type checker handles function calls in several cases:

### OCALL (generic call)

The generic `OCALL` is rewritten to a specific call type:

```c
case OCALL:
    typecheck(&n->left, Erv | Etype | Ecall);
    l = n->left;

    if(l->op == ONAME && l->etype != 0) {
        // built-in function
        goto builtin;    // → len, cap, make, etc.
    }

    if(l->op == OTYPE) {
        // type conversion: T(x)
        n->op = OCONV;
    }

    // Determine call kind:
    switch(l->op) {
    case ODOTINTER:
        n->op = OCALLINTER;  // interface method call
    case ODOTMETH:
        n->op = OCALLMETH;   // concrete method call
    default:
        n->op = OCALLFUNC;   // regular function call
    }
```

### typecheckaste() — Argument/Parameter Matching

This function checks that arguments match parameters:
- Count matches
- Types are assignable
- Variadic parameters handle `...` correctly
- Multi-return function call as arguments

---

## Built-in Functions

**Lines ~1200-1800** (the `builtin:` label)

Each built-in has special type-checking logic:

### len / cap

```c
case OLEN: case OCAP:
    // Allowed types:
    // len: string, array, slice, map, chan
    // cap: array, slice, chan
    switch(t->etype) {
    case TCHAN: case TMAP: case TARRAY: case TSTRING:
        ok;
    default:
        yyerror("invalid argument for %O", n->op);
    }
    n->type = types[TINT];
```

### make

```c
case OMAKE:
    // make(T, args...)
    // T must be slice, map, or chan
    switch(t->etype) {
    case TARRAY:   n->op = OMAKESLICE;  // make([]T, len, cap)
    case TMAP:     n->op = OMAKEMAP;    // make(map[K]V, hint)
    case TCHAN:    n->op = OMAKECHAN;   // make(chan T, bufsize)
    }
```

### new

```c
case ONEW:
    // new(T) → *T
    n->type = ptrto(t);
```

### append

```c
case OAPPEND:
    // append(slice, elems...)
    // First arg must be slice
    // Remaining args must be assignable to element type
    // Or: append([]byte, string...) — special case
```

### delete

```c
case ODELETE:
    // delete(map, key)
    // First arg must be map
    // Second arg must be assignable to key type
```

### complex / real / imag

```c
case OCOMPLEX:
    // complex(float, float) → complex
case OREAL: case OIMAG:
    // real(complex) → float
    // imag(complex) → float
```

### copy

```c
case OCOPY:
    // copy(dst, src)
    // Both must be slices of same element type
    // Or: copy([]byte, string) — special case
```

### panic / print / println / recover

These have minimal type checking — they accept any types.

---

## Type Assertions

**Lines 826-862**

```c
case ODOTTYPE:    // x.(T)
    // Left side must be an interface
    if(!isinter(t))
        yyerror("invalid type assertion (non-interface on left)");

    // If T is concrete, check at compile time that T
    // could possibly implement the interface
    if(n->type->etype != TINTER)
        if(!implements(n->type, t, &missing, &have, &ptr))
            yyerror("impossible type assertion: T does not implement I");
```

The `implements()` function in `subr.c` checks whether a concrete
type has all methods required by the interface. If not, it reports
**exactly** what's missing:

- Missing method entirely
- Method with wrong type signature
- Method with pointer receiver (but value used)

This is one of Go's best error messages — the type checker explains
precisely why a type doesn't implement an interface.

---

## Composite Literals

**typecheckcomplit(), later in the file**

The function `typecheckcomplit()` handles `T{k:v, ...}` syntax:

```c
static void typecheckcomplit(Node **np)
{
    // Determine the type from context or explicit type
    // For each element:
    //   - If keyed (k:v), check key type
    //   - Type-check value against expected type
    //   - For structs: match field names
    //   - For arrays/slices: validate indices
    //   - For maps: check key/value types
}
```

Special case: string-to-byte-array conversion in composite literals:
```go
[]byte("hello")  // This is a conversion, handled by stringtoarraylit()
```

---

## Assignments

### typecheckas() — Single Assignment

```c
case OAS:
    typecheck(&n->left, Erv | Easgn);
    checkassign(n->left);    // verify LHS is assignable
    typecheck(&n->right, Erv);
    if(n->right != N)
        n->right = assignconv(n->right, n->left->type, "assignment");
```

### typecheckas2() — Multi-Assignment

```c
case OAS2:
    // x, y = a, b
    // or: x, y = f()  → OAS2FUNC
    // or: x, ok = m[k] → OAS2MAPR
    // or: x, ok = <-ch → OAS2RECV
    // or: x, ok = i.(T) → OAS2DOTTYPE
```

The type checker recognizes special multi-value patterns:

| Pattern | Rewritten To |
|---------|-------------|
| `x, y = f()` | `OAS2FUNC` |
| `x, ok = m[k]` | `OAS2MAPR` |
| `x, ok = <-ch` | `OAS2RECV` |
| `x, ok = i.(T)` | `OAS2DOTTYPE` |

---

## Constant Evaluation

**const.c** (called from typecheck)

During type checking, the compiler evaluates constant expressions
at **arbitrary precision**. Key functions:

```c
evconst(n)      // evaluate constant expression
convconst()     // convert constant to target type
defaultlit()    // assign default type to untyped constant
defaultlit2()   // reconcile two untyped constants
```

### The Go Constant Model

Go constants are **untyped** until they're used in a typed context.
The constant `42` has type `TIDEAL` (untyped number) and can be
assigned to any numeric type. When assigned to `int`, `defaultlit()`
converts it.

The rules are in `const.c`:
- Untyped int → `int` (by default)
- Untyped float → `float64` (by default)
- Untyped complex → `complex128` (by default)
- Untyped bool → `bool`
- Untyped string → `string`
- Untyped rune → `rune` (int32)

### Constant Folding

The type checker folds constant expressions during checking:
```go
const x = 1 + 2*3    // evaluated at compile time to 7
const y = "hello" + " world"  // → "hello world"
```

> **Reference**: The constant folding and arbitrary-precision
> arithmetic follow standard compiler practice:
> - Aho, A.V. et al. (2006). "Compilers: Principles, Techniques,
>   and Tools", 2nd Edition. Section 6.1.1: "Constant Folding".

---

## Error Handling Patterns

### Graceful Error Recovery

The type checker uses `goto error` for error recovery:
```c
error:
    n->type = T;    // set type to nil
    // fall through to ret
ret:
    // validate context flags
    if(!(ok & ~(Erv|Etype)))
        if(!(top & (Erv|Etype)))
            yyerror("... not used");
    *np = n;
```

After an error, the type is set to `T` (nil), and subsequent
type checks on this node will propagate the error silently (since
`T == nil` is always checked).

### Error Messages

The type checker produces Go's famously clear error messages:

```
"cannot use X (type T) as type U in assignment"
"invalid operation: X (operator + not defined on string)"
"impossible type assertion: T does not implement I (missing method M)"
"cannot take the address of X"
"non-integer string index X"
```

---

## Node Rewriting Summary

The type checker rewrites many generic operations to specific ones:

| Before | After | Condition |
|--------|-------|-----------|
| `OXDOT` | `ODOT` | Always |
| `ODOT` | `ODOTPTR` | Through pointer |
| `ODOT` | `ODOTMETH` | Method call |
| `ODOT` | `ODOTINTER` | Interface method |
| `OCALL` | `OCALLFUNC` | Regular function |
| `OCALL` | `OCALLMETH` | Method call |
| `OCALL` | `OCALLINTER` | Interface call |
| `OCALL` | `OCONV` | Type conversion |
| `OINDEX` | `OINDEXMAP` | Map index |
| `OSLICE` | `OSLICESTR` | String slice |
| `OSLICE` | `OSLICEARR` | Array slice |
| `OADD` (string) | `OADDSTR` | String concat |
| `OEQ` (string) | `OCMPSTR` | String compare |
| `OEQ` (iface) | `OCMPIFACE` | Interface compare |
| `OMAKE` | `OMAKESLICE` | make([]T) |
| `OMAKE` | `OMAKEMAP` | make(map) |
| `OMAKE` | `OMAKECHAN` | make(chan) |
| `OCOMPLIT` | `OARRAYLIT` | []T{} |
| `OCOMPLIT` | `OMAPLIT` | map[K]V{} |
| `OCOMPLIT` | `OSTRUCTLIT` | T{} |
| `OAS2` | `OAS2FUNC` | x, y = f() |
| `OAS2` | `OAS2MAPR` | x, ok = m[k] |
| `OAS2` | `OAS2RECV` | x, ok = <-ch |
| `OAS2` | `OAS2DOTTYPE` | x, ok = i.(T) |
| `OTARRAY` | `OTYPE` | Type syntax resolved |
| `OTMAP` | `OTYPE` | Type syntax resolved |
| `OTCHAN` | `OTYPE` | Type syntax resolved |

These rewrites are crucial — they convert the **syntactic** AST into
a **semantic** AST where each node carries precise meaning.

---

## References

- Go Specification (2014). "Types", "Expressions", "Assignability",
  "Operators", "Built-in functions".
  https://go.dev/ref/spec

- Aho, A.V., Lam, M.S., Sethi, R., Ullman, J.D. (2006).
  "Compilers: Principles, Techniques, and Tools", 2nd Edition.
  Chapter 6: Intermediate-Code Generation.
  Addison-Wesley. ISBN: 0-321-48681-1.

- Pierce, B.C. (2002). "Types and Programming Languages".
  MIT Press. ISBN: 0-262-16209-1.
  (Comprehensive reference on type systems and type checking.)

- Cardelli, L. and Wegner, P. (1985). "On Understanding Types,
  Data Abstraction, and Polymorphism". Computing Surveys, 17(4),
  pp. 471-522. DOI: 10.1145/6041.6042
  (Theory of structural vs nominal typing — Go uses both.)
