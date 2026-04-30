# Go 1.4 Compiler: Parser (go.y)

**Source files**:
- `src/cmd/gc/go.y` (2223 lines) — YACC grammar definition
- `src/cmd/gc/y.tab.c` (5132 lines) — Generated LALR(1) parser

The parser converts the token stream from the lexer into an
**Abstract Syntax Tree (AST)** made of `Node` structures.

---

## Table of Contents

1. [YACC and LALR(1) Parsing](#yacc-and-lalr1-parsing)
2. [Grammar Structure Overview](#grammar-structure-overview)
3. [Token and Type Declarations](#token-and-type-declarations)
4. [Operator Precedence](#operator-precedence)
5. [Top-Level Structure (file, package, imports)](#top-level-structure)
6. [Declarations (var, const, type)](#declarations)
7. [Function Declarations](#function-declarations)
8. [Statements](#statements)
9. [Expressions](#expressions)
10. [Types](#types)
11. [Composite Literals](#composite-literals)
12. [Import Data Grammar](#import-data-grammar)
13. [Shift/Reduce Conflict Resolution](#shiftreduce-conflict-resolution)
14. [AST Construction Patterns](#ast-construction-patterns)

---

## YACC and LALR(1) Parsing

The Go 1.4 compiler uses **YACC** (Yet Another Compiler-Compiler) to
generate its parser from a grammar specification.

### What is YACC?

YACC takes a **context-free grammar** (in BNF-like notation) and
generates a C function `yyparse()` that recognizes strings in the
language defined by that grammar. The generated parser uses the
**LALR(1)** parsing algorithm.

> **Reference**: Johnson, S.C. (1975). "Yacc: Yet Another
> Compiler-Compiler". Bell Laboratories Computing Science Technical
> Report No. 32. Murray Hill, New Jersey.
> https://people.cs.pitt.edu/~mock/cs2210/yacc.pdf

### What is LALR(1)?

**LALR(1)** stands for **Look-Ahead LR(1)**:
- **L**: Scans input Left to right
- **R**: Produces a Rightmost derivation (in reverse)
- **1**: Uses 1 token of lookahead
- **LA**: Uses look-ahead sets to resolve ambiguities

The algorithm was invented by Frank DeRemer:

> **Reference**: DeRemer, F.L. (1969). "Practical Translators for
> LR(k) Languages". PhD dissertation, MIT.
>
> DeRemer, F.L. and Pennello, T.J. (1982). "Efficient Computation
> of LALR(1) Look-Ahead Sets". ACM Transactions on Programming
> Languages and Systems, 4(4), pp. 615-649.
> DOI: 10.1145/69622.357187

### How the Parser Works

1. The lexer (`yylex()`) provides tokens one at a time.
2. The parser maintains a **stack** of states and symbols.
3. At each step, it either:
   - **Shifts**: pushes the next token onto the stack
   - **Reduces**: pops symbols matching a grammar rule's right side
     and pushes the rule's left side, executing the rule's **action**
     (C code in `{...}` blocks)
4. Actions build AST nodes using `nod()`, `list()`, etc.

### Why LALR(1) for Go?

Go was **deliberately designed** to be parseable by simple algorithms.
Rob Pike has stated that Go's grammar was kept simple to enable fast
compilation. The entire Go grammar fits in ~2200 lines of YACC,
which is very small for a practical programming language.

> "One of the reasons Go compiles so fast is that the grammar is simple
>  enough to be parsed without a symbol table."
>  — Rob Pike, various talks on Go design

---

## Grammar Structure Overview

The `go.y` file has this structure:

```
Lines 1-19:     Copyright and semicolon rules comment
Lines 20-27:    %{ C prologue %}
Lines 28-35:    %union declaration (parser value stack type)
Lines 37-95:    Token and type declarations
Lines 96-121:   Precedence declarations
Lines 122-131:  Top-level file rule
Lines 132-279:  Package and import rules
Lines 280-351:  Common declarations (var, const, type)
Lines 352-471:  Statements (simple_stmt, assignments)
Lines 473-582:  Case blocks (switch/select)
Lines 583-778:  Control flow (for, if, switch, select)
Lines 780-908:  Expressions
Lines 910-1064: Primary expressions (calls, index, slice, etc.)
Lines 1066-1305: Type syntax
Lines 1306-1485: Function declarations and literals
Lines 1487-1865: Lists (left-recursive for stack efficiency)
Lines 1866-1916: Optional productions
Lines 1917-2223: Hidden import grammar (for reading export data)
```

---

## Token and Type Declarations

**Lines 28-95**

### The %union (line 28)

The parser's value stack entries can hold different types:

```yacc
%union {
    Node*     node;    // AST node
    NodeList* list;    // list of nodes
    Type*     type;    // type (used in import grammar)
    Sym*      sym;     // symbol
    struct Val val;    // constant value
    int       i;      // integer (line numbers, etc.)
}
```

Each grammar symbol has a declared type from this union.

### Token Declarations (lines 39-48)

```yacc
%token <val>  LLITERAL          // literals carry a Val
%token <i>    LASOP LCOLAS      // carry an int (op code or line number)
%token <sym>  LBREAK LCASE ...  // keywords carry their Sym*
%token        LANDAND LBODY ... // pure tokens carry nothing
```

### Non-terminal Type Declarations (lines 50-94)

```yacc
%type <node>  stmt expr ...     // statements and expressions are Nodes
%type <list>  xdcl_list ...     // declaration lists are NodeLists
%type <type>  hidden_type ...   // import types are Type*
%type <sym>   sym packname ...  // names are Sym*
%type <i>     lbrace            // lbrace carries whether it was LBODY or '{'
```

---

## Operator Precedence

**Lines 96-119**

YACC uses `%left` and `%right` declarations to define operator
precedence and associativity. Lower in the file = higher precedence.

```yacc
%left   LCOMM                           // <- (lowest)
%left   LOROR                           // ||
%left   LANDAND                         // &&
%left   LEQ LNE LLE LGE LLT LGT       // == != <= >= < >
%left   '+' '-' '|' '^'                // + - | ^
%left   '*' '/' '%' '&' LLSH LRSH LANDNOT  // * / % & << >> &^  (highest)
```

This matches the **Go specification's operator precedence** exactly:

| Precedence | Operators |
|-----------|-----------|
| 5 (highest) | `*  /  %  <<  >>  &  &^` |
| 4 | `+  -  \|  ^` |
| 3 | `==  !=  <  <=  >  >=` |
| 2 | `&&` |
| 1 (lowest) | `\|\|` |

> **Reference**: Go Specification, Section "Operator precedence":
> https://go.dev/ref/spec#Operator_precedence

### Conflict Resolution Tokens (lines 104-119)

Additional pseudo-tokens resolve shift/reduce conflicts:

```yacc
%left   NotPackage    // lower than LPACKAGE
%left   LPACKAGE      // resolves: empty vs package statement
%left   NotParen      // lower than '('
%left   '('           // resolves: name vs name()
%left   ')'
%left   PreferToRightParen  // higher than ')'
```

These are **never produced by the lexer**. They exist only to give
precedence hints to grammar rules via `%prec`. For example:

```yacc
package:
    %prec NotPackage      // empty package → error
    { yyerror("package statement must be first"); }
|   LPACKAGE sym ';'      // actual package statement
```

The `%prec NotPackage` gives the empty rule lower precedence than
`LPACKAGE`, ensuring the parser prefers to shift `LPACKAGE` rather
than reduce the empty rule.

---

## Top-Level Structure

**Lines 122-279**

### file (line 123)

A Go source file is:

```yacc
file:
    loadsys         // load runtime definitions (implicit)
    package         // package declaration
    imports         // import declarations
    xdcl_list       // top-level declarations
    { xtop = concat(xtop, $4); }
```

`xtop` is the global list of all top-level declarations. It
accumulates across all source files in the package.

### loadsys (line 149)

Before any user code is parsed, the compiler loads **built-in
runtime definitions**. These come from a "canned" string
(`runtimeimport`) that defines runtime functions like `newobject`,
`makeslice`, `gopanic`, etc.

```yacc
loadsys:
    { importpkg = runtimepkg;
      cannedimports("runtime.builtin", runtimeimport); }
    import_package
    import_there
    { importpkg = nil; }
```

This is why Go code can call `make()`, `new()`, etc. — they're
pre-loaded as runtime functions before parsing begins.

### imports (line 165)

```yacc
imports:
    /* empty */
|   imports import ';'

import:
    LIMPORT import_stmt
|   LIMPORT '(' import_stmt_list osemi ')'
|   LIMPORT '(' ')'
```

This handles all three import syntaxes:
```go
import "fmt"
import (
    "fmt"
    "os"
)
import ()  // valid but useless
```

### import_here (line 225)

Three forms of import naming:

```yacc
import_here:
    LLITERAL                // import "fmt"
|   sym LLITERAL            // import f "fmt"
|   '.' LLITERAL            // import . "fmt" (dot import)
```

Each form calls `importfile()` in `lex.c` to locate and load
the package's export data.

---

## Declarations

**Lines 280-406**

### common_dcl (line 303)

Handles `var`, `const`, and `type` declarations, each in both
single and grouped form:

```yacc
common_dcl:
    LVAR vardcl                       // var x int
|   LVAR '(' vardcl_list osemi ')'    // var ( x int; y string )
|   LVAR '(' ')'                      // var ()
|   lconst constdcl                   // const x = 1
|   lconst '(' constdcl ... ')'       // const ( x = 1; y = 2 )
|   LTYPE typedcl                     // type T int
|   LTYPE '(' typedcl_list ... ')'    // type ( T int; U string )
```

### vardcl (line 358)

Three forms of variable declaration:

```yacc
vardcl:
    dcl_name_list ntype                // var x int
|   dcl_name_list ntype '=' expr_list  // var x int = 1
|   dcl_name_list '=' expr_list        // var x = 1  (type inferred)
```

Each calls `variter()` in `dcl.c` to create declaration nodes.

### constdcl (line 372)

```yacc
constdcl:
    dcl_name_list ntype '=' expr_list  // const x int = 1
|   dcl_name_list '=' expr_list        // const x = 1

constdcl1:
    constdcl
|   dcl_name_list ntype               // const x int  (reuse previous value)
|   dcl_name_list                     // const x       (reuse previous value)
```

`constdcl1` handles the Go feature where subsequent constants in a
`const()` block can omit the value and reuse the previous one (with
`iota` incremented). This is managed by `constiter()` in `dcl.c`.

### typedcl (line 402)

```yacc
typedclname:
    sym
    { $$ = typedcl0($1); }   // name becomes visible immediately

typedcl:
    typedclname ntype
    { $$ = typedcl1($1, $2, 1); }
```

**Important**: Type names become visible at the point of the `sym`,
not at the end of the declaration. This allows recursive types:
```go
type Node struct {
    Left *Node  // 'Node' is already visible here
}
```

---

## Function Declarations

**Lines 1306-1485**

### xfndcl (line 1310)

```yacc
xfndcl:
    LFUNC fndcl fnbody
    {
        $$ = $2;
        $$->nbody = $3;
        $$->noescape = noescape;   // //go:noescape pragma
        $$->nosplit = nosplit;      // //go:nosplit pragma
        funcbody($$);
    }
```

### fndcl (line 1325) — Two Forms

**Regular function:**
```yacc
fndcl:
    sym '(' oarg_type_list_ocomma ')' fnres
    {
        // func foo(args) returns
        t = nod(OTFUNC, N, N);
        t->list = $3;       // params
        t->rlist = $5;      // returns
        $$ = nod(ODCLFUNC, N, N);
        $$->nname = newname($1);
        declare($$->nname, PFUNC);
        funchdr($$);
    }
```

**Method:**
```yacc
|   '(' oarg_type_list_ocomma ')' sym '(' oarg_type_list_ocomma ')' fnres
    {
        // func (recv) method(args) returns
        rcvr = $2->n;
        t = nod(OTFUNC, rcvr, N);
        t->list = $6;       // params
        t->rlist = $8;      // returns
        $$ = nod(ODCLFUNC, N, N);
        $$->shortname = newname($4);
        $$->nname = methodname1($$->shortname, rcvr->right);
        declare($$->nname, PFUNC);
        funchdr($$);
    }
```

Special handling for `init` and `main`:
```c
if(strcmp($1->name, "init") == 0) {
    $1 = renameinit();   // each init func gets a unique name
    if($3 != nil || $5 != nil)
        yyerror("func init must have no arguments and no return values");
}
```

### fnliteral (line 1476) — Closure/Lambda

```yacc
fnliteral:
    fnlitdcl lbrace stmt_list '}'
    { $$ = closurebody($3); }

fnlitdcl:
    fntype
    { closurehdr($1); }    // opens closure scope
```

`closurehdr()` and `closurebody()` in `closure.c` handle the
creation of a closure function node, capturing variables from
the enclosing scope.

---

## Statements

**Lines 408-471, 1698-1780**

### simple_stmt (line 408)

```yacc
simple_stmt:
    expr                          // expression statement
|   expr LASOP expr               // x += y (compound assignment)
|   expr_list '=' expr_list       // x = y or x, y = a, b
|   expr_list LCOLAS expr_list    // x := y
|   expr LINC                     // x++
|   expr LDEC                     // x--
```

**Interesting detail** — `x++` and `x--` are **not** expressions in Go
(unlike C). They're statements:

```c
// x++ becomes OASOP with implicit=1
$$ = nod(OASOP, $1, nodintconst(1));
$$->implicit = 1;
$$->etype = OADD;
```

**Short variable declaration** (`:=`, line 444):

```c
// x := y
// But check: is the right side a type switch?
if($3->n->op == OTYPESW) {
    $$ = nod(OTYPESW, N, $3->n->right);
    $$->left = dclname($1->n->sym);
} else {
    $$ = colas($1, $3, $2);  // normal :=
}
```

### non_dcl_stmt (line 1716)

```yacc
non_dcl_stmt:
    simple_stmt
|   for_stmt
|   switch_stmt
|   select_stmt
|   if_stmt
|   labelname ':' stmt           // labeled statement
|   LFALL                        // fallthrough
|   LBREAK onew_name             // break [label]
|   LCONTINUE onew_name          // continue [label]
|   LGO pseudocall               // go f()
|   LDEFER pseudocall            // defer f()
|   LGOTO new_name               // goto label
|   LRETURN oexpr_list           // return [values]
```

**`go` and `defer` restrictions**: These can only take a function call
(`pseudocall`), not an arbitrary expression. The grammar enforces this
directly — you cannot write `go x + 1`.

### for_stmt (line 651)

```yacc
for_stmt:
    LFOR { markdcl(); } for_body { $$ = $3; popdcl(); }

for_header:
    osimple_stmt ';' osimple_stmt ';' osimple_stmt  // for init; cond; post
|   osimple_stmt                                     // for cond
|   range_stmt                                       // for range
```

Note `markdcl()` and `popdcl()` — these push/pop the declaration
scope, so variables declared in the for-init are scoped to the for body.

The parser enforces that `:=` cannot appear in the post-statement:
```c
if($5 != N && $5->colas != 0)
    yyerror("cannot declare in the for-increment");
```

### return statement (line 1764)

```yacc
LRETURN oexpr_list
{
    $$ = nod(ORETURN, N, N);
    $$->list = $2;
    // Check for shadowed named return values
    if($$->list == nil && curfn != N) {
        for(l=curfn->dcl; l; l=l->next) {
            if(l->n->class == PPARAMOUT)
                if(l->n->sym->def != l->n)
                    yyerror("%s is shadowed during return", l->n->sym->name);
        }
    }
}
```

This catches a common bug: named return values shadowed by local
variables during a bare `return`.

---

## Expressions

**Lines 780-908**

### Binary Expressions (line 783)

Go has **no ternary operator** and **no comma operator**. Binary
expressions are straightforward:

```yacc
expr:
    uexpr                          // unary expression
|   expr LOROR   expr  { $$ = nod(OOROR, $1, $3); }   // ||
|   expr LANDAND expr  { $$ = nod(OANDAND, $1, $3); }  // &&
|   expr LEQ     expr  { $$ = nod(OEQ, $1, $3); }     // ==
|   expr LNE     expr  { $$ = nod(ONE, $1, $3); }     // !=
|   expr LLT     expr  { $$ = nod(OLT, $1, $3); }     // <
    // ... all comparison and arithmetic operators
|   expr LCOMM   expr  { $$ = nod(OSEND, $1, $3); }   // <-
```

Precedence is resolved by the `%left` declarations, not by the
grammar structure. All binary operators are **left-associative**.

### Unary Expressions (line 867)

```yacc
uexpr:
    pexpr                         // primary expression
|   '*' uexpr  { $$ = nod(OIND, $2, N); }    // dereference
|   '&' uexpr  { ... }           // address-of (special case for &T{})
|   '+' uexpr  { $$ = nod(OPLUS, $2, N); }   // unary plus
|   '-' uexpr  { $$ = nod(OMINUS, $2, N); }  // unary minus
|   '!' uexpr  { $$ = nod(ONOT, $2, N); }    // logical NOT
|   '~' uexpr  { yyerror("the bitwise complement operator is ^"); }
|   '^' uexpr  { $$ = nod(OCOM, $2, N); }    // bitwise complement
|   LCOMM uexpr { $$ = nod(ORECV, $2, N); }  // channel receive
```

**Helpful error**: If you write `~x` (C-style complement), the compiler
says "the bitwise complement operator is ^" (line 898).

**`&T{...}` special case** (line 875):

```c
if($2->op == OCOMPLIT) {
    // Turn &T{...} into (*T){...} with implicit dereference
    $$ = $2;
    $$->right = nod(OIND, $$->right, N);
    $$->right->implicit = 1;
}
```

This transforms `&T{x: 1}` at the syntax level. The composite literal
gets a pointer type, and the allocation is handled later.

### Primary Expressions (line 931)

```yacc
pexpr_no_paren:
    LLITERAL              // 42, "hello", 3.14
|   name                  // x, fmt
|   pexpr '.' sym         // x.Field or pkg.Name
|   pexpr '.' '(' expr_or_type ')'  // x.(T) type assertion
|   pexpr '.' '(' LTYPE ')'         // x.(type) type switch
|   pexpr '[' expr ']'              // a[i] index
|   pexpr '[' oexpr ':' oexpr ']'   // a[i:j] 2-index slice
|   pexpr '[' oexpr ':' oexpr ':' oexpr ']'  // a[i:j:k] 3-index slice
|   pseudocall            // f() function call
|   convtype '(' expr ')' // T(x) conversion
|   comptype lbrace ... '}' // T{...} composite literal
|   fnliteral             // func() {...}
```

**Dot expressions** (line 937): Package-qualified names (`fmt.Println`)
are recognized here and resolved immediately:

```c
if($1->op == OPACK) {
    s = restrictlookup($3->name, $1->pkg);
    $1->used = 1;          // mark package as used
    $$ = oldname(s);        // look up the name in that package
}
```

**Three-index slices** (line 964): Go 1.2 added `a[i:j:k]` to control
slice capacity. The parser requires both `j` and `k`:

```c
if($5 == N) yyerror("middle index required in 3-index slice");
if($7 == N) yyerror("final index required in 3-index slice");
```

---

## Types

**Lines 1066-1305**

The type grammar is split to avoid parsing conflicts:

```yacc
ntype:            // general type
    recvchantype  // <-chan T
|   fntype        // func(...)...
|   othertype     // []T, [N]T, map[K]V, chan T, struct{}, interface{}
|   ptrtype       // *T
|   dotname       // T or pkg.T
|   '(' ntype ')' // (T) parenthesized
```

### Why Split?

Channel types are split into `recvchantype` and `non_recvchantype`
because `<-chan T` is ambiguous:

```go
chan<- T   // send-only channel of T
<-chan T   // receive-only channel of T
```

The lexer returns `<-` as `LCOMM`. Without the split, the parser
couldn't distinguish `<-` as channel direction vs. `<-` as receive.

### othertype (line 1240) — Array, Map, Chan, Struct, Interface

```yacc
othertype:
    '[' oexpr ']' ntype          // []T or [N]T
|   '[' LDDD ']' ntype           // [...]T
|   LCHAN non_recvchantype       // chan T (bidirectional)
|   LCHAN LCOMM ntype            // chan<- T (send-only)
|   LMAP '[' ntype ']' ntype     // map[K]V
|   structtype                   // struct { ... }
|   interfacetype                // interface { ... }
```

### structtype (line 1280)

```yacc
structtype:
    LSTRUCT lbrace structdcl_list osemi '}'
    { $$ = nod(OTSTRUCT, N, N); $$->list = $3; }
```

Struct fields (structdcl, line 1548) handle:
- Named fields: `x int`
- Embedded types: `io.Reader`
- Embedded pointers: `*io.Reader`
- Tagged fields: `x int \`json:"x"\``

---

## Composite Literals

**Lines 979-1044**

Composite literals (`T{...}`) are one of the trickiest parts of the
grammar because `{` is ambiguous (see the [loophack](02-lexer.md#the-loophack)).

```yacc
pexpr_no_paren:
    comptype lbrace start_complit braced_keyval_list '}'
    { $$ = $3; $$->right = $1; $$->list = $4; }
|   pexpr_no_paren '{' start_complit braced_keyval_list '}'
    { $$ = $3; $$->right = $1; $$->list = $4; }
```

Two forms:
1. `comptype { ... }` — type that's clearly a type (`[]int`, `map[K]V`)
2. `pexpr_no_paren { ... }` — type that looks like an expression (`T`)

`start_complit` (line 1001) creates the `OCOMPLIT` node early so it
gets the correct line number.

### Nested Composite Literals

```yacc
bare_complitexpr:
    expr                         // value
|   '{' start_complit braced_keyval_list '}'  // nested T{...}

complitexpr:
    expr
|   '{' start_complit braced_keyval_list '}'
```

`bare_complitexpr` adds `OPAREN` wrappers around simple names to
preserve line numbers, since composite literals commonly span
multiple lines.

---

## Import Data Grammar

**Lines 1917-2223**

The second half of `go.y` defines a **separate grammar** for reading
**export data** from imported packages. This is how the compiler learns
about types and functions from other packages without re-parsing their
source code.

Export data is stored in `.a` (archive) or `.6`/`.8`/`.5` object
files as text between `$$` markers:

```
$$
package fmt
import runtime "runtime"
type @"".Stringer interface { String() string }
func @"".Println(a ...interface{}) (n int, err error)
$$
```

The `hidden_*` grammar rules parse this format:

```yacc
hidden_import:
    LIMPORT LNAME LLITERAL ';'              // import "path"
|   LVAR hidden_pkg_importsym hidden_type   // var sym type
|   LCONST hidden_pkg_importsym '=' ...     // const sym = value
|   LTYPE hidden_pkgtype hidden_type        // type sym = typedef
|   LFUNC hidden_fndcl fnbody               // func sym(args) body
```

The `@""` syntax represents symbols qualified by package path.
`@"".Println` means `Println` in the current import package.

---

## Shift/Reduce Conflict Resolution

**Lines 104-119**

The grammar has several intentional shift/reduce conflicts resolved
by precedence annotations. The comment at line 104 explains:

```c
/*
 * manual override of shift/reduce conflicts.
 * the general form is that we assign a precedence
 * to the token being shifted and then introduce
 * NotToken with lower precedence or PreferToToken with higher
 * and annotate the reducing rule accordingly.
 */
```

### Example: Empty package (line 132)

```yacc
package:
    %prec NotPackage              // empty → error
    { yyerror("package statement must be first"); }
|   LPACKAGE sym ';'             // package foo
```

Without `%prec NotPackage`, the parser wouldn't know whether to
reduce the empty rule or shift `LPACKAGE`. The precedence annotation
forces it to prefer shifting.

### Example: Function return types (line 1455)

```yacc
fnres:
    %prec NotParen                // no return type
    { $$ = nil; }
|   fnret_type                    // single return type
|   '(' oarg_type_list_ocomma ')' // (multiple returns)
```

Without `%prec NotParen`, after parsing `func f(x int)`, the parser
can't decide if a following `(` starts return types or is a call.
`NotParen` has lower precedence than `(`, so the parser prefers to
shift `(` and parse it as return types.

---

## AST Construction Patterns

### nod() — Create a Node

Every grammar action uses `nod(op, left, right)` to create AST nodes:

```c
nod(OADD, $1, $3)     // $1 + $3
nod(OIF, N, N)         // if (fields filled later)
nod(ORETURN, N, N)     // return (values in ->list)
nod(OCOMPLIT, N, N)    // composite literal
```

### list() / list1() / concat() — Build Lists

```c
list1($1)              // create single-element list
list($1, $3)           // append $3 to list $1
concat($1, $3)         // concatenate two lists
```

Lists are **left-recursive** in the grammar (line 1487):
```yacc
/* left recursive to conserve yacc stack */
stmt_list:
    stmt            { $$ = list1($1); }
|   stmt_list ';' stmt  { $$ = list($1, $3); }
```

### markdcl() / popdcl() — Scope Management

Every block-opening rule calls `markdcl()` and the corresponding
close calls `popdcl()`:

```yacc
compound_stmt:
    '{'  { markdcl(); }
    stmt_list '}'
    { $$ = liststmt($3); popdcl(); }
```

This pushes/pops the declaration scope, ensuring variables declared
inside a block aren't visible outside.

---

## References

- Johnson, S.C. (1975). "Yacc: Yet Another Compiler-Compiler".
  Bell Laboratories Computing Science Technical Report No. 32.
  https://people.cs.pitt.edu/~mock/cs2210/yacc.pdf

- DeRemer, F.L. and Pennello, T.J. (1982). "Efficient Computation
  of LALR(1) Look-Ahead Sets". ACM TOPLAS, 4(4), pp. 615-649.
  DOI: 10.1145/69622.357187

- Aho, A.V., Sethi, R., Ullman, J.D. (1986). "Compilers: Principles,
  Techniques, and Tools" (Dragon Book), Chapter 4: Syntax Analysis.
  Addison-Wesley. ISBN: 0-201-10088-6.

- Aho, A.V., Lam, M.S., Sethi, R., Ullman, J.D. (2006).
  "Compilers: Principles, Techniques, and Tools", 2nd Edition.
  Chapter 4: Syntax Analysis. Addison-Wesley. ISBN: 0-321-48681-1.

- Go Specification: "Source file organization", "Declarations",
  "Expressions", "Statements".
  https://go.dev/ref/spec

- Knuth, D.E. (1965). "On the Translation of Languages from
  Left to Right". Information and Control, 8(6), pp. 607-639.
  DOI: 10.1016/S0019-9958(65)90426-2
  (The original LR parsing paper.)
