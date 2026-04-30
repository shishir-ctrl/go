# Go 1.4 Compiler: Lexer (lex.c)

**Source file**: `src/cmd/gc/lex.c` (2414 lines)

The lexer is both the **tokenizer** and the **entry point** (`main()`)
for the entire compiler. It converts raw UTF-8 source text into a
stream of tokens consumed by the YACC-generated parser.

---

## Table of Contents

1. [main() — The Compiler Entry Point](#main-the-compiler-entry-point)
2. [Compilation Phases in main()](#compilation-phases-in-main)
3. [The Lexer (yylex / _yylex)](#the-lexer)
4. [Automatic Semicolon Insertion](#automatic-semicolon-insertion)
5. [The "Loophack"](#the-loophack)
6. [Character Reading (getc, getr)](#character-reading)
7. [Number Literal Parsing](#number-literal-parsing)
8. [String Literal Parsing](#string-literal-parsing)
9. [Keyword and Builtin Initialization](#keyword-and-builtin-initialization)
10. [Easter Eggs](#easter-eggs)
11. [Compiler Pragmas](#compiler-pragmas)

---

## main() — The Compiler Entry Point

**Lines 199-518**

The `main()` function orchestrates the entire compilation of one package.
Here is the execution flow:

```c
int main(int argc, char *argv[])
{
    // 1. Signal handling (SIGBUS, SIGSEGV → fault handler)
    signal(SIGBUS, fault);
    signal(SIGSEGV, fault);

    // 2. Architecture validation
    //    Ensure GOARCH matches this compiler (e.g., "amd64" for 6g)
    goarch = getgoarch();

    // 3. Linker context initialization
    linkarchinit();
    ctxt = linknew(thelinkarch);

    // 4. Package setup
    localpkg = mkpkg(strlit(""));       // package being compiled
    builtinpkg = mkpkg(strlit("go.builtin"));
    unsafepkg = mkpkg(strlit("unsafe"));
    runtimepkg = mkpkg(strlit("runtime"));
    // ... plus several pseudo-packages for symbol namespacing

    // 5. Command-line flag parsing (lines 276-322)
    // 6. Lexer/type system initialization (lines 369-377)
    lexinit();    // register keywords and built-in types
    typeinit();   // initialize type sizes and alignments
    lexinit1();   // register error, byte, rune types
    yytinit();    // initialize YACC token name table

    // 7. Parse each source file (lines 384-414)
    for(i=0; i<argc; i++) {
        curio.bin = Bopen(argv[i], OREAD);
        yyparse();   // YACC parser consumes tokens from yylex()
    }

    // 8. Type checking in 3 phases (lines 423-450)
    // 9. Inlining (lines 458-481)
    // 10. Escape analysis (line 489)
    // 11. Function compilation (lines 496-498)
    // 12. Object file output (line 511)
    dumpobj();
}
```

### Command-Line Flags (lines 276-322)

The compiler accepts many debugging flags. Key ones:

| Flag | Purpose |
|------|---------|
| `-N` | Disable optimizations |
| `-l` | Toggle inlining (default: on) |
| `-S` | Print assembly listing |
| `-m` | Print optimization decisions |
| `-e` | No limit on error count |
| `-W` | Debug: print AST after type checking |
| `-x` | Debug: trace lexer |
| `-race` | Enable race detector instrumentation |
| `-wb` | Enable write barrier (for GC) |
| `-live` | Debug liveness analysis |

The inlining flag has a subtle toggle (lines 356-357):
```c
// -l: disable inlining (debug['l'] becomes 0)
// no flag: inlining on (debug['l'] becomes 1)
// -ll: inlining on + extra debug (debug['l'] becomes 2)
if(debug['l'] <= 1)
    debug['l'] = 1 - debug['l'];
```

---

## Compilation Phases in main()

**Lines 423-518**

After parsing, the compiler runs 7 phases on the AST. The phases
are executed sequentially on `xtop` — the linked list of all
top-level declarations.

```
Phase 1 (line 429): typecheck consts, types, func signatures
Phase 2 (line 435): typecheck variable assignments
Phase 3 (line 441): typecheck function bodies
Phase 4 (line 458): function inlining (caninl + inlcalls)
Phase 5 (line 489): escape analysis
Phase 6 (line 496): compile each function (order→walk→compile)
Phase 7 (line 511): write object file
```

See [00-architecture.md](00-architecture.md) for details on why these
phases are ordered this way.

---

## The Lexer

**yylex() at line 1607, _yylex() at line 870**

The lexer has two layers:

1. **`_yylex()`** (line 870): The actual tokenizer. Reads characters
   and returns token codes.
2. **`yylex()`** (line 1607): Wrapper that implements **automatic
   semicolon insertion**.

### _yylex() Token Recognition (line 870)

The tokenizer uses a **hand-written** lexer (not generated from a
regular expression specification). This is typical for production
compilers — hand-written lexers are faster and give better error
messages than generated ones.

The main loop at line 900:
```c
l0:
    c = getc();                // read one byte
    if(yy_isspace(c)) {
        if(c == '\n' && curio.nlsemi) {
            ungetc(c);
            return ';';        // automatic semicolon insertion!
        }
        goto l0;               // skip whitespace
    }

    if(c >= Runeself)          // multibyte UTF-8 → identifier
        goto talph;
    if(yy_isalpha(c))          // ASCII letter → identifier/keyword
        goto talph;
    if(yy_isdigit(c))          // digit → number literal
        goto tnum;

    switch(c) {                // operators and punctuation
        case '"': ...          // string literal
        case '`': ...          // raw string literal
        case '\'': ...         // rune literal
        case '/': ...          // comment or division
        // ... all operators
    }
```

### Token Types

The lexer returns these token categories:

| Return Value | Examples | Notes |
|-------------|----------|-------|
| ASCII char | `+`, `-`, `{`, etc. | Single-char tokens |
| `LNAME` | `x`, `fmt`, `int` | Identifiers and keywords |
| `LLITERAL` | `42`, `"hello"`, `3.14` | All literals |
| `LASOP` | `+=`, `-=`, `*=` | Compound assignment |
| `LCOLAS` | `:=` | Short variable declaration |
| `LINC` / `LDEC` | `++` / `--` | Increment/decrement |
| `LLSH` / `LRSH` | `<<` / `>>` | Shifts |
| `LCOMM` | `<-` | Channel operator |
| `LANDAND` / `LOROR` | `&&` / `\|\|` | Logical operators |
| `LEQ` / `LNE` | `==` / `!=` | Equality |
| `LLE` / `LGE` / `LLT` / `LGT` | `<=` `>=` `<` `>` | Comparison |
| `LDDD` | `...` | Variadic |
| `LBODY` | `{` | Block-opening brace (see loophack) |
| Keyword tokens | `LIF`, `LFOR`, `LRETURN`, etc. | |

### Identifier Recognition (talph, line 1301)

Identifiers are scanned character by character. The identifier string
is built in `lexbuf`, then looked up in the **symbol table**:

```c
talph:
    for(;;) {
        if(c >= Runeself) {        // Unicode character
            rune = getr();
            if(!isalpharune(rune) && !isdigitrune(rune))
                error;
            cp += runetochar(cp, &rune);
        } else if(!yy_isalnum(c) && c != '_')
            break;
        else
            *cp++ = c;
        c = getc();
    }
    s = lookup(lexbuf);            // look up in symbol table
    return s->lexical;             // return token type
```

If the identifier is a **keyword** (like `if`, `for`, `return`), the
symbol table lookup returns the keyword's token type (e.g., `LIF`).
Otherwise it returns `LNAME`.

> **Key insight**: Keywords are not special-cased in the lexer. They
> are regular identifiers that were pre-registered in the symbol table
> with their token types during `lexinit()`. This is a clean design
> pattern used in many compilers.

---

## Automatic Semicolon Insertion

**Lines 1607-1643**

Go uses **automatic semicolon insertion** — the programmer rarely
types semicolons, but the grammar needs them. The lexer inserts
them automatically after certain tokens.

```c
int32 yylex(void) {
    lx = _yylex();

    // EOF after a semicolon-eligible token → insert semicolon
    if(curio.nlsemi && lx == EOF)
        lx = ';';

    // After these tokens, a newline should be treated as semicolon
    switch(lx) {
    case LNAME:      // identifier
    case LLITERAL:   // literal
    case LBREAK:     // break
    case LCONTINUE:  // continue
    case LFALL:      // fallthrough
    case LRETURN:    // return
    case LINC:       // ++
    case LDEC:       // --
    case ')':
    case '}':
    case ']':
        curio.nlsemi = 1;   // next newline → semicolon
        break;
    default:
        curio.nlsemi = 0;
        break;
    }
}
```

When `curio.nlsemi` is set and the next character is `\n`, the `_yylex()`
function at line 903 returns `';'` instead of skipping the whitespace.

**This rule comes directly from the Go specification:**

> "When the input is broken into tokens, a semicolon is automatically
>  inserted into the token stream immediately after a line's final
>  token if that token is: an identifier, an integer/float/imaginary/
>  rune/string literal, one of the keywords break/continue/fallthrough/
>  return, one of the operators ++/--, or a closing )/]/}."
>
> — Go Specification, Section "Semicolons"
> https://go.dev/ref/spec#Semicolons

### Why Automatic Semicolons?

This design decision has several benefits:

1. **Cleaner code**: No semicolons cluttering source files.
2. **Enforced formatting**: The brace style `{` must be on the same
   line as `if`/`for`/`func`, because a newline before `{` would
   insert a semicolon and break the syntax.
3. **Simple implementation**: Just 40 lines of code.

> **Historical context**: JavaScript also has automatic semicolon
> insertion (ASI), but its rules are more complex and error-prone.
> Go's ASI is much simpler because it only triggers after specific
> token types, not based on grammatical context.

---

## The "Loophack"

**Lines 1226-1274**

This is one of the most delightful comments in the codebase (line 1227):

```c
/*
 * clumsy dance:
 * to implement rule that disallows
 *      if T{1}[0] { ... }
 * but allows
 *      if (T{1}[0]) { ... }
 * the block bodies for if/for/switch/select
 * begin with an LBODY token, not '{'.
 *
 * i said it was clumsy.
 */
```

**The problem**: In Go, `{` has two meanings:
1. Start of a block body (`if x { ... }`)
2. Start of a composite literal (`T{1, 2, 3}`)

Without the loophack, `if T{` would be ambiguous — is `T{` a
composite literal or the start of the if-body?

**The solution**: After keywords `if`, `for`, `switch`, `select`,
the lexer sets `loophack = 1`. The next `{` that appears at
loophack level 1 is returned as `LBODY` instead of `{`.

Parentheses `(` push the loophack counter onto a stack and reset
it to 0, so `if (T{1}[0]) {` correctly recognizes the `{` inside
parens as a composite literal and the final `{` as LBODY.

> **Note**: This hack was removed in Go 1.5 when the compiler was
> rewritten in Go, as the new parser uses a different strategy.

---

## Character Reading

**Lines 1645-1736**

### getc() — Single Byte (line 1645)

Reads one byte from the source, handling:
- **Peek buffer**: Two characters of lookahead (`peekc`, `peekc1`)
- **Canned imports**: Can read from a string instead of a file
- **BOM detection**: Warns about UTF-8 BOM in middle of file
- **NUL byte**: Reports error (Go source must not contain NUL)
- **EOF**: Inserts a `\n` before EOF if the file doesn't end with one
- **Line counting**: Increments `lexlineno` on `\n`

### getr() — Unicode Rune (line 1707)

Reads a complete UTF-8 character (rune):

```c
static int32 getr(void) {
    c = getc();
    if(c < Runeself)           // ASCII (0-127)
        return c;

    str[i++] = c;
    while(!fullrune(str, i))   // accumulate UTF-8 bytes
        str[i++] = getc();

    chartorune(&rune, str);    // decode UTF-8 → rune
    if(rune == Runeerror)
        yyerror("illegal UTF-8 sequence");
    return rune;
}
```

> **Reference**: UTF-8 was invented by Ken Thompson and Rob Pike
> (the Go creators) in 1992 for Plan 9. The encoding is defined in:
> - Pike, R. and Thompson, K. (1993). "Hello World, or Καλημέρα
>   κόσμε, or こんにちは 世界". Proceedings of the Winter 1993 USENIX
>   Technical Conference, pp. langley-1.
> - RFC 3629: "UTF-8, a transformation format of ISO 10646" (2003).
>   https://www.rfc-editor.org/rfc/rfc3629

### ungetc() — Push Back (line 1698)

Pushes a character back onto the input. Supports two levels of
pushback (`peekc` and `peekc1`), which is needed for tokens like
`...` (three characters of lookahead).

---

## Number Literal Parsing

**Lines 1344-1501**

The lexer handles all Go numeric literal formats:

### Integer Literals (tnum, line 1344)

```
Decimal:     123
Octal:       0777
Hexadecimal: 0xFF, 0XFF
```

The parsing strategy:
1. If starts with `0x`/`0X` → hex
2. If starts with `0` → octal (with check for invalid digits 8, 9)
3. Otherwise → decimal
4. Result parsed into `Mpint` via `mpatofix()` (arbitrary precision)

### Float Literals (casedot/caseep, lines 1434-1501)

```
1.5, .5, 1., 1e10, 1.5e-3, 0x1p10
```

After the decimal point or exponent marker, digits are accumulated
and parsed into `Mpflt` via `mpatoflt()`.

### Imaginary Literals (casei, line 1471)

```
1i, 1.5i, 1e10i
```

Any numeric literal followed by `i` becomes the imaginary part of
a complex constant. The real part is set to 0.

### Rune Literals (line 1021)

```
'a', '\n', '\x0A', '\u00E9', '\U0001F600'
```

Parsed by `escchar()` which handles all Go escape sequences:
- `\a \b \f \n \r \t \v \\ \'`
- `\x##` (2 hex digits → byte)
- `\u####` (4 hex digits → Unicode)
- `\U########` (8 hex digits → Unicode)
- `\###` (3 octal digits → byte)

---

## String Literal Parsing

**Lines 960-1019**

### Interpreted Strings (`"..."`, line 960)

Process escape sequences. Characters are accumulated into a
dynamically-growing buffer via `mal()` / `remal()`. UTF-8 encoding
is preserved — the string bytes are stored exactly as they'll appear
at runtime.

### Raw Strings (`` `...` ``, line 985)

No escape processing. Carriage returns (`\r`) are silently dropped
(per the Go spec). Everything else is literal.

### String Storage Format

Both end at `strlit:` (line 1010) where the length is stored as a
4-byte prefix:

```c
*(int32*)cp = clen - sizeof(int32);    // store length
cp[clen++] = 0;                        // null terminate
// align to MAXALIGN
```

The result is a `Strlit*` stored in `yylval.val.u.sval`.

---

## Keyword and Builtin Initialization

**lexinit() at line 1934**

Keywords and built-in functions are registered in a single table
`syms[]` (line 1853):

### Keywords (25 total)

```
break    case     chan     const    continue
default  defer    else    fallthrough for
func     go       goto    if       import
interface map     package  range   return
select   struct   switch  type     var
```

Each keyword is stored as a symbol with its token type:
```c
{"if", LIF, Txxx, OXXX}     // 'if' returns LIF token
{"for", LFOR, Txxx, OXXX}   // 'for' returns LFOR token
```

### Built-in Types (16 total)

```
int8  int16  int32  int64  uint8  uint16  uint32  uint64
float32  float64  complex64  complex128
bool  string  any
```

These are pre-registered as type symbols:
```c
{"int32", LNAME, TINT32, OXXX}   // 'int32' is a name of type TINT32
```

### Built-in Functions (15 total)

```
append  cap    close   complex  copy   delete  imag
len     make   new     panic    print  println real  recover
```

These are registered as names with operation codes:
```c
{"len", LNAME, Txxx, OLEN}      // 'len' is a name with op=OLEN
{"make", LNAME, Txxx, OMAKE}    // 'make' has op=OMAKE
```

### Special Initializations (lexinit1, line 2022)

After the table-driven init, special types are constructed:

1. **`error` interface**: Built manually as `interface { Error() string }`
2. **`byte`**: Aliased to `uint8`
3. **`rune`**: Aliased to `int32`
4. **`true`/`false`**: Registered as boolean constants
5. **`nil`**: Registered as nil literal
6. **`_` (blank)**: Registered with block=-100 (always in scope)
7. **`iota`**: Registered for const blocks

---

## Easter Eggs

**Lines 1927-1931**

Hidden in the keyword table are five "ignored" words:

```c
{"notwithstanding",      LIGNORE, Txxx, OXXX},
{"thetruthofthematter",  LIGNORE, Txxx, OXXX},
{"despiteallobjections", LIGNORE, Txxx, OXXX},
{"whereas",              LIGNORE, Txxx, OXXX},
{"insofaras",            LIGNORE, Txxx, OXXX},
```

These are **legal legalese** words that the compiler silently
ignores. They were inserted as a joke by the Go team — you can
write `notwithstanding` anywhere in your Go code and it will be
silently accepted. This was a prank from the early days of Go
development.

The `LIGNORE` token type causes the lexer to skip the word entirely
(line 1329):
```c
case LIGNORE:
    goto l0;    // just skip it
```

> **Note**: These were removed in later versions of Go when the
> compiler was rewritten.

---

## Compiler Pragmas

**getlinepragma() at line 1510**

The lexer recognizes special `//go:` comments as compiler directives:

### `//line file:line`

Changes the reported source file and line number. Used by code
generators (like `go generate`, `cgo`, `yacc`) to map errors back
to original source files.

### `//go:noescape`

Marks the next function declaration as having arguments that do
not escape to the heap. Used for assembly-implemented functions
where escape analysis can't see the implementation.

### `//go:nosplit`

Marks the next function as not needing a stack split check. Used
for functions that must not be preempted (e.g., in the runtime
scheduler).

### `//go:nointerface`

(When `fieldtrack` experiment is enabled.) Prevents interface
satisfaction checking for this method.

---

## References

- **Lexical Analysis Theory**:
  - Aho, A.V., Sethi, R., Ullman, J.D. (1986). "Compilers: Principles,
    Techniques, and Tools" (The Dragon Book). Chapter 3: Lexical Analysis.
    Addison-Wesley. ISBN: 0-201-10088-6.

- **UTF-8**:
  - Pike, R. and Thompson, K. (1993). "Hello World, or Καλημέρα κόσμε,
    or こんにちは 世界". Proceedings of the Winter 1993 USENIX Conference.
  - RFC 3629: "UTF-8, a transformation format of ISO 10646".
    https://www.rfc-editor.org/rfc/rfc3629

- **Go Specification**:
  - "The Go Programming Language Specification" (2014).
    Sections: Semicolons, Tokens, String Literals, Integer Literals.
    https://go.dev/ref/spec

- **Automatic Semicolon Insertion**:
  - Cox, R. (2009). "Semicolons in Go".
    https://go.dev/doc/effective_go#semicolons
