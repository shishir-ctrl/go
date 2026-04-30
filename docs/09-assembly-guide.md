# How the Go Compiler Produces Machine Code + Assembly Study Guide

## Your Assumption: Verified and Clarified

**Yes, the Go compiler converts Go source into architecture-specific
machine code.** But the exact pipeline has an important nuance:

```
Go Source (.go)
     │
     ▼
┌─────────────────────────────────────────────────────────────┐
│  6g compiler (src/cmd/6g/)                                  │
│                                                             │
│  Go AST ──► Plan 9 pseudo-assembly ──► machine code bytes   │
│              (internal representation)   (in same pass)     │
│                                                             │
│  Output: .6 object file (contains machine code + metadata)  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  6l linker (src/cmd/6l/)                                    │
│                                                             │
│  - Resolves cross-file references (patches addresses)       │
│  - Combines all .6 files + runtime into one binary          │
│  - Outputs: ELF (Linux), Mach-O (macOS), PE (Windows)       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
              Native executable (runs on CPU directly)
```

**Key point**: The compiler does NOT output a `.s` assembly text file
that gets fed to a separate assembler (`as` / `nasm`). Instead, `6g`
directly emits machine code bytes. The "assembly" is an **internal
representation** that you can view with the `-S` flag.

### Proof from the Code

The `add` function:
```go
func add(a, b int) int { return a + b }
```

Produces this **Plan 9 pseudo-assembly** (viewable with `-S`):
```asm
MOVQ    "".a+8(FP), BX     // load first argument
MOVQ    "".b+16(FP), BP    // load second argument  
ADDQ    BP, BX              // add them
MOVQ    BX, "".~r2+24(FP)  // store result
RET
```

And simultaneously these **raw x86-64 machine code bytes**:
```
48 8b 5c 24 08    = MOV RBX, [RSP+8]
48 8b 6c 24 10    = MOV RBP, [RSP+16]
48 01 eb          = ADD RBX, RBP
48 89 5c 24 18    = MOV [RSP+24], RBX
c3                = RET
```

Both representations are in the `.6` object file.

---

## Part 1: Plan 9 Assembly Syntax (What Go Uses)

Go's assembly uses **Plan 9 syntax**, which is different from both
Intel syntax and AT&T (GNU) syntax. You need to understand all three.

### Three Assembly Syntaxes Compared

The same instruction in three syntaxes:

| Plan 9 (Go) | AT&T (GNU/GCC) | Intel (NASM/MASM) |
|-------------|-----------------|-------------------|
| `MOVQ AX, BX` | `movq %rax, %rbx` | `mov rbx, rax` |
| `ADDQ $1, AX` | `addq $1, %rax` | `add rax, 1` |
| `MOVQ 8(SP), AX` | `movq 8(%rsp), %rax` | `mov rax, [rsp+8]` |
| `CMPQ AX, $42` | `cmpq $42, %rax` | `cmp rax, 42` |

### Key Differences

| Feature | Plan 9 | AT&T | Intel |
|---------|--------|------|-------|
| **Operand order** | `SRC, DST` | `SRC, DST` | `DST, SRC` |
| **Register prefix** | None (`AX`) | `%` (`%rax`) | None (`rax`) |
| **Immediate prefix** | `$` (`$42`) | `$` (`$42`) | None (`42`) |
| **Memory** | `8(SP)` | `8(%rsp)` | `[rsp+8]` |
| **Size suffix** | `Q/L/W/B` | `q/l/w/b` | Inferred |
| **Indirection** | `(AX)` | `(%rax)` | `[rax]` |

### Plan 9 Size Suffixes

| Suffix | Bits | Go Types | Example |
|--------|------|----------|---------|
| `B` | 8 | `byte`, `bool` | `MOVB $1, AX` |
| `W` | 16 | `int16` | `MOVW AX, BX` |
| `L` | 32 | `int32`, `float32` | `MOVL AX, BX` |
| `Q` | 64 | `int64`, `int`, `pointer` | `MOVQ AX, BX` |

### Plan 9 Register Names

| Plan 9 | x86-64 | Purpose |
|--------|--------|---------|
| `AX` | `RAX` | Accumulator (return value in C ABI) |
| `BX` | `RBX` | Base register (callee-saved) |
| `CX` | `RCX` | Counter (loop, shift) |
| `DX` | `RDX` | Data (multiply/divide high bits) |
| `SI` | `RSI` | Source index |
| `DI` | `RDI` | Destination index |
| `SP` | `RSP` | Stack pointer |
| `BP` | `RBP` | Base pointer (frame pointer) |
| `R8`-`R15` | `R8`-`R15` | Extended registers |
| `X0`-`X15` | `XMM0`-`XMM15` | SSE registers (float) |
| `FP` | *(virtual)* | Frame pointer (arguments) |
| `SB` | *(virtual)* | Static base (global symbols) |
| `PC` | `RIP` | Program counter |
| `TLS` | `FS:` | Thread-local storage |

### Go's Virtual Registers

Go adds **pseudo-registers** that don't exist in hardware:

- **`FP`** (Frame Pointer): Points to the **caller's** stack frame.
  Used to access function arguments: `arg+0(FP)`, `arg+8(FP)`

- **`SB`** (Static Base): Used for global/package-level symbols:
  `runtime.newobject(SB)`, `"".myGlobal(SB)`

- **`SP`** (Stack Pointer): In Go assembly, `SP` refers to the
  **bottom** of the current frame (top of allocated locals), not
  the hardware RSP. When used without a name prefix, it means the
  actual hardware SP.

```
    ┌───────────────────────┐ high addresses
    │ caller's frame        │
    ├───────────────────────┤ ← FP (virtual)
    │ arg2 (+16)            │
    │ arg1 (+8)             │
    │ return address (+0)   │
    ├───────────────────────┤ ← hardware SP on entry
    │ local vars            │
    │ ...                   │
    ├───────────────────────┤ ← SP (after SUBQ $size, SP)
    │ outgoing args         │
    └───────────────────────┘ low addresses
```

---

## Part 2: x86-64 Instruction Set — What You Need to Know

You do NOT need to learn all 699 instructions defined in `6.out.h`.
The Go compiler uses a **small subset** regularly. Here are the
categories you need, ordered by importance:

### Category 1: Data Movement (MOST IMPORTANT)

These appear in ~60% of compiler output.

| Instruction | Meaning | Example |
|-------------|---------|---------|
| `MOVQ src, dst` | Move 64-bit value | `MOVQ AX, BX` |
| `MOVL src, dst` | Move 32-bit value | `MOVL AX, BX` |
| `MOVB src, dst` | Move 8-bit value | `MOVB $1, AX` |
| `MOVBQZX src, dst` | Move byte, zero-extend to 64-bit | `MOVBQZX (AX), BX` |
| `MOVBQSX src, dst` | Move byte, sign-extend to 64-bit | |
| `MOVLQZX src, dst` | Move 32-bit, zero-extend | |
| `MOVLQSX src, dst` | Move 32-bit, sign-extend | |
| `LEAQ src, dst` | Load effective address | `LEAQ 8(AX), BX` |

**Key concept**: `MOVQ` is by far the most common instruction. It
copies data between registers, memory, and immediates.

`LEAQ` is special: it computes an address without loading from it.
`LEAQ 8(AX)(BX*4), CX` means `CX = AX + BX*4 + 8` — it's often
used for **arithmetic**, not just addresses.

### Category 2: Arithmetic

| Instruction | Meaning | Example |
|-------------|---------|---------|
| `ADDQ src, dst` | `dst += src` | `ADDQ BX, AX` |
| `SUBQ src, dst` | `dst -= src` | `SUBQ $8, SP` |
| `IMULQ src, dst` | `dst *= src` (signed) | `IMULQ BX, AX` |
| `IDIVQ src` | `AX = DX:AX / src` | `IDIVQ BX` |
| `INCQ dst` | `dst++` | `INCQ AX` |
| `DECQ dst` | `dst--` | `DECQ AX` |
| `NEGQ dst` | `dst = -dst` | `NEGQ AX` |
| `SHLQ n, dst` | Left shift | `SHLQ $3, AX` |
| `SHRQ n, dst` | Logical right shift | `SHRQ $1, AX` |
| `SARQ n, dst` | Arithmetic right shift | `SARQ $1, AX` |

**Division is weird**: `IDIVQ BX` divides the 128-bit value
`DX:AX` by `BX`, storing quotient in `AX` and remainder in `DX`.

### Category 3: Comparison and Branching

| Instruction | Meaning | Example |
|-------------|---------|---------|
| `CMPQ src, dst` | Compare (sets flags) | `CMPQ AX, $0` |
| `TESTQ src, dst` | Bitwise AND (sets flags, discards result) | `TESTQ AX, AX` |
| `JMP target` | Unconditional jump | `JMP label` |
| `JEQ target` | Jump if equal (ZF=1) | `JEQ done` |
| `JNE target` | Jump if not equal | `JNE loop` |
| `JGT target` | Jump if greater (signed) | |
| `JGE target` | Jump if greater or equal | |
| `JLT target` | Jump if less (signed) | |
| `JLE target` | Jump if less or equal | |
| `JHI target` | Jump if higher (unsigned) | |
| `JLS target` | Jump if lower or same (unsigned) | |

**`TESTQ AX, AX` vs `CMPQ AX, $0`**: Both check if AX is zero,
but `TESTQ` is shorter (no immediate operand).

### Category 4: Stack and Calling

| Instruction | Meaning | Example |
|-------------|---------|---------|
| `SUBQ $n, SP` | Allocate stack frame | `SUBQ $24, SP` |
| `ADDQ $n, SP` | Deallocate stack frame | `ADDQ $24, SP` |
| `CALL target` | Call function | `CALL runtime.newobject(SB)` |
| `RET` | Return from function | `RET` |

**Go doesn't use PUSH/POP** for arguments (unlike C). Everything
goes through explicit `MOVQ` to/from `SP`-relative addresses.

### Category 5: Bitwise Operations

| Instruction | Meaning | Go equivalent |
|-------------|---------|--------------|
| `ANDQ src, dst` | `dst &= src` | `x & y` |
| `ORQ src, dst` | `dst \|= src` | `x \| y` |
| `XORQ src, dst` | `dst ^= src` | `x ^ y` |
| `NOTQ dst` | `dst = ^dst` | `^x` |

**`XORQ AX, AX`**: Common idiom to zero a register (shorter than
`MOVQ $0, AX`).

### Category 6: Go-Specific Pseudo-Instructions

| Instruction | Meaning |
|-------------|---------|
| `TEXT name(SB), flags, $framesize-argsize` | Function definition |
| `FUNCDATA $n, sym(SB)` | GC metadata pointer |
| `PCDATA $n, $value` | PC-dependent data (for GC, stack maps) |
| `NOP` | No operation (alignment/placeholder) |

---

## Part 3: Reading Go Compiler Output

### How to Generate Assembly

```bash
# Method 1: Compile with -S flag
./bin/go tool 6g -S myfile.go

# Method 2: Build binary, then disassemble
./bin/go build -o binary myfile.go
objdump -d binary    # AT&T syntax
```

### Anatomy of a Go Function in Assembly

```asm
"".fib t=1 size=128 value=0 args=0x10 locals=0x18
```
- `"".fib`: Function name (package `""` = current package)
- `t=1`: Symbol type (1 = text/code)
- `size=128`: Function code size in bytes
- `args=0x10`: Argument size (16 bytes = two int64s)
- `locals=0x18`: Local variable size (24 bytes)

```asm
TEXT  "".fib+0(SB), $24-16
```
- `TEXT`: Declares a function
- `$24`: Stack frame size (24 bytes for locals)
- `-16`: Argument size (16 bytes)

### The Stack Check Prologue

Every function (except `nosplit`) starts with:

```asm
MOVQ    (TLS), CX           // Load goroutine struct pointer
CMPQ    SP, 16(CX)          // Compare SP with stack guard
JHI     ,22                  // If SP > guard, stack is ok
CALL    runtime.morestack    // Otherwise, grow the stack
JMP     ,0                   // Retry from the beginning
```

This is how Go implements **growable stacks**. If the function
would overflow the stack, `morestack` allocates a bigger one,
copies the old stack, and retries.

### Reading a Real Example: Fibonacci

```go
func fib(n int) int {
    if n <= 1 { return n }
    return fib(n-1) + fib(n-2)
}
```

Becomes:
```asm
// --- Prologue: stack check ---
MOVQ    (TLS), CX           // CX = &goroutine
CMPQ    SP, 16(CX)          // SP vs stack guard
JHI     22                   // skip if enough stack
CALL    runtime.morestack    // grow stack
JMP     0                    // retry

// --- Frame setup ---
SUBQ    $24, SP              // allocate 24 bytes on stack

// --- Load argument ---
MOVQ    "".n+32(FP), AX     // AX = n (argument, at FP+32)
                              // +32 because: return addr(8) + frame(24) = 32

// --- if n <= 1 { return n } ---
CMPQ    AX, $1              // compare n with 1
JGT     47                   // if n > 1, skip to recursive case
MOVQ    AX, "".~r1+40(FP)   // store return value = n
ADDQ    $24, SP              // deallocate frame
RET                          // return

// --- fib(n-1) ---
MOVQ    AX, BX              // BX = n
DECQ    BX                   // BX = n - 1
MOVQ    BX, (SP)            // push argument for fib(n-1)
CALL    "".fib               // call fib(n-1)
MOVQ    8(SP), BX           // BX = fib(n-1) result

// --- Save intermediate result ---
MOVQ    BX, 16(SP)          // save fib(n-1) to local var

// --- fib(n-2) ---
MOVQ    "".n+32(FP), BX     // reload n (may have moved during call)
SUBQ    $2, BX              // BX = n - 2
MOVQ    BX, (SP)            // push argument for fib(n-2)
CALL    "".fib               // call fib(n-2)
MOVQ    8(SP), AX           // AX = fib(n-2) result

// --- Add and return ---
MOVQ    16(SP), BX          // BX = fib(n-1) (saved earlier)
ADDQ    AX, BX              // BX = fib(n-1) + fib(n-2)
MOVQ    BX, "".~r1+40(FP)  // store return value
ADDQ    $24, SP              // deallocate frame
RET                          // return
```

**Key observations:**
1. Arguments come from `FP` (frame pointer), results go to `FP`
2. Stack is explicitly managed with `SUBQ`/`ADDQ` on `SP`
3. Local variables are at `SP`-relative offsets
4. After a `CALL`, registers may be clobbered — the compiler
   reloads `n` from the stack after calling `fib(n-1)`

---

## Part 4: Go's Calling Convention (Go 1.4)

Go 1.4 uses an **all-stack** calling convention:

```
┌─────────────────────────────────┐
│ Return value 2  (+offset)       │  ← caller writes space,
│ Return value 1  (+offset)       │     callee fills values
│ Argument 3      (+offset)       │  ← caller writes values
│ Argument 2      (+offset)       │
│ Argument 1      (+offset)       │
│ Return address  (+0)            │  ← CALL pushes this
├─────────────────────────────────┤
│ Callee's local variables        │  ← SUBQ allocates these
│ ...                             │
│ Outgoing arguments for subcalls │
└─────────────────────────────────┘
```

**No register arguments**: Everything on the stack. This is
different from C's System V ABI (which uses `RDI, RSI, RDX, RCX,
R8, R9` for the first 6 arguments).

**Why?** Stack-based calling makes it easy for the garbage
collector to find all pointers, and enables goroutine stack
copying (all pointers are at known offsets).

---

## Part 5: Study Plan for Learning Assembly

### Phase 1: Learn to READ Assembly First (1 week)

Start by **reading** compiler output, not writing assembly from scratch.
This is the fastest path to understanding the Go compiler.

1. **"The Faker's Guide to Reading (x86) Assembly Language"**
   by Tim Debug — **Start here**
   https://www.timdbg.com/posts/fakers-guide-to-assembly/
   *(Teaches reading compiler output using Compiler Explorer.)*

2. **"How to Read Assembly Language"** by Scott Wolchok
   https://wolchok.org/posts/how-to-read-assembly-language/
   *(Practical approach to instruction patterns and program flow.)*

3. **"Learning to Read x86 Assembly Language"** by Pat Shaughnessy
   https://patshaughnessy.net/2016/11/26/learning-to-read-x86-assembly-language
   *(Real code examples, explains AT&T suffixes b/w/l/q.)*

**Practice**: Use https://godbolt.org (Compiler Explorer) to see
how C code maps to assembly. Compare with Go output.

### Phase 2: x86-64 Fundamentals (1-2 weeks)

4. **"x86-64 Assembly Language Programming with Ubuntu"**
   by Ed Jorgensen, Ph.D. (free PDF textbook, 24 chapters)
   http://www.egr.unlv.edu/~ed/assembly64.pdf
   Also at: https://open.umn.edu/opentextbooks/textbooks/733

5. **CS107 Stanford x86-64 Reference Sheet** (one-page cheat sheet)
   https://web.stanford.edu/class/cs107/resources/x86-64-reference.pdf

6. **Exercism x86-64 Assembly Track** (114 free exercises)
   https://exercism.org/tracks/x86-64-assembly

7. **Intel x86-64 Instruction Reference** (bookmark this):
   https://www.felixcloutier.com/x86/
   *(Searchable HTML version of Intel's manual.)*

### Phase 3: Plan 9 / Go Assembly Specifics (1 week)

8. **"A Quick Guide to Go's Assembler"** (official, essential):
   https://go.dev/doc/asm

9. **Rob Pike, "The Design of the Go Assembler"** (GopherCon 2016):
   https://go.dev/talks/2016/asm.slide

10. **Rob Pike, "A Manual for the Plan 9 Assembler"** (original):
    https://9p.io/sys/doc/asm.html

11. **"Go assembly language complementary reference"** by Iskander Sharipov
    https://www.quasilyte.dev/blog/post/go-asm-complementary-reference/
    *(Detailed tables: Go vs AT&T/Intel, instruction suffixes,
    register naming across architectures.)*

### Phase 4: Reading Go Compiler Output (ongoing)

12. **Practice with every Go construct**:
    ```bash
    # See assembly for any Go code:
    ./bin/go tool 6g -S yourfile.go
    ```

13. **Study these constructs in order:**
    - Simple arithmetic (`a + b`)
    - If/else (conditional jumps)
    - For loops (backward jumps)
    - Function calls (CALL/RET)
    - Struct field access (memory offsets)
    - Slice operations (3-word header)
    - Interface method calls (itab dispatch)
    - Goroutine creation (runtime.newproc)
    - Channel operations (runtime.chansend)

### Phase 5: x86-64 Machine Encoding (advanced)

14. **"Intel 64 and IA-32 Architectures Software Developer's Manual"**
    Volume 2: Instruction Set Reference
    https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html

15. **x86 Opcode and Instruction Reference** (precise encoding):
    http://ref.x86asm.net/

16. **Understanding x86-64 instruction encoding**:
    ```
    48 8b 5c 24 08
    │  │  │  │  └── displacement: 0x08 (offset 8)
    │  │  │  └──── ModR/M + SIB: [RSP+disp8]
    │  │  └────── ModR/M: mod=01, reg=011(RBX), rm=100(SIB)
    │  └──────── opcode: 8B = MOV r64, r/m64
    └────────── REX prefix: 48 = REX.W (64-bit operand)
    ```

### Phase 6: Calling Conventions (for understanding function calls)

17. **"x86-64 Calling Conventions"** at Wayne's Talk
    https://waynestalk.com/en/x86-64-calling-conventions-en/

18. **"The 64-bit x86 C Calling Convention"** by Aaron Bloomfield (UVA)
    https://aaronbloomfield.github.io/pdr/book/x86-64bit-ccc-chapter.pdf

*(Note: Go 1.4 does NOT use the System V ABI — it passes everything
on the stack. But understanding the C convention helps when reading
runtime code that interfaces with the OS.)*

---

## Part 6: Essential x86-64 Concepts

### Registers You Must Know

```
┌──────────────────────────────────────────┐
│  64-bit    32-bit   16-bit   8-bit       │
│  RAX       EAX      AX       AL          │
│  RBX       EBX      BX       BL          │
│  RCX       ECX      CX       CL          │
│  RDX       EDX      DX       DL          │
│  RSI       ESI      SI       SIL         │
│  RDI       EDI      DI       DIL         │
│  RSP       ESP      SP       SPL         │
│  RBP       EBP      BP       BPL         │
│  R8-R15    R8D-R15D R8W-R15W R8B-R15B    │
└──────────────────────────────────────────┘
```

### The FLAGS Register

After `CMPQ` or `TESTQ`, these flags are set:

| Flag | Name | Set When |
|------|------|----------|
| ZF | Zero | Result is 0 |
| SF | Sign | Result is negative |
| CF | Carry | Unsigned overflow |
| OF | Overflow | Signed overflow |

Conditional jumps test these flags:
- `JEQ` = jump if ZF=1
- `JNE` = jump if ZF=0
- `JGT` = jump if ZF=0 and SF=OF (signed greater)
- `JLT` = jump if SF≠OF (signed less)
- `JHI` = jump if CF=0 and ZF=0 (unsigned above)

### Memory Addressing Modes

```
Mode                   Plan 9          x86-64
─────────────────────────────────────────────
Register direct        AX              rax
Immediate              $42             42
Register indirect      (AX)            [rax]
Base + displacement    8(AX)           [rax+8]
Base + index           (AX)(BX*1)      [rax+rbx]
Base + scaled index    (AX)(BX*8)      [rax+rbx*8]
Full form              16(AX)(BX*4)    [rax+rbx*4+16]
PC-relative            sym(SB)         [rip+sym]
```

---

## References

- **Go Team**. "A Quick Guide to Go's Assembler".
  https://go.dev/doc/asm

- **Pike, R.** "A Manual for the Plan 9 Assembler".
  https://9p.io/sys/doc/asm.html

- **Jorgensen, E.** "x86-64 Assembly Language Programming with Ubuntu".
  http://www.egr.unlv.edu/~ed/x86.html

- **Cloutier, F.** "x86 and amd64 instruction reference".
  https://www.felixcloutier.com/x86/

- **Intel Corporation** (2024). "Intel 64 and IA-32 Architectures
  Software Developer's Manual", Volumes 1-3.
  https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html

- **Duntemann, J.** (2009). "Assembly Language Step-by-Step:
  Programming with Linux", 3rd Edition. Wiley.
  ISBN: 978-0-470-49702-9.
