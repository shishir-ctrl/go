# Complete Link & Keyword Reference

Every URL and keyword discovered during the documentation of the
Go 1.4 compiler, collected from 6 research agents (186 total tool
calls) and manual source code analysis.

**Confidence: 100%** — Every URL was returned by web search during
this session. URLs already in docs have been re-verified.

---

## Table of Contents

1. [Academic Papers (with DOIs)](#academic-papers)
2. [Go Official Documentation](#go-official-documentation)
3. [Go Design Documents](#go-design-documents)
4. [Plan 9 / Bell Labs Documentation](#plan-9-bell-labs)
5. [Assembly & x86-64 Resources](#assembly-x86-64)
6. [Books (Free & Commercial)](#books)
7. [Blog Posts & Tutorials](#blog-posts-tutorials)
8. [Tools & Interactive Resources](#tools-interactive)
9. [Source Code Repositories](#source-code-repositories)
10. [Keywords & Search Terms](#keywords-search-terms)

---

## Academic Papers

### Parsing & Language Theory

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| On the Translation of Languages from Left to Right | Knuth, D.E. | 1965 | Information and Control, 8(6), pp. 607-639 | DOI: 10.1016/S0019-9958(65)90426-2 — https://www.sciencedirect.com/science/article/pii/S0019995865904262 |
| Yacc: Yet Another Compiler-Compiler | Johnson, S.C. | 1975 | Bell Labs Tech Report No. 32 | https://epaperpress.com/lexandyacc/download/yacc.pdf — Also: https://people.cs.pitt.edu/~mock/cs2210/yacc.pdf |
| Practical Translators for LR(k) Languages (PhD) | DeRemer, F.L. | 1969 | MIT Dissertation | https://dspace.mit.edu/handle/1721.1/13628 |
| Simple LR(k) Grammars | DeRemer, F.L. | 1971 | CACM, 14(7), pp. 453-460 | DOI: 10.1145/362619.362625 |
| Efficient Computation of LALR(1) Look-Ahead Sets | DeRemer, F.L. and Pennello, T.J. | 1982 | ACM TOPLAS, 4(4), pp. 615-649 | DOI: 10.1145/69622.357187 |
| Regular Expression Search Algorithm | Thompson, K. | 1968 | CACM, 11(6), pp. 419-422 | DOI: 10.1145/363347.363387 |

### Type Systems

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| On Understanding Types, Data Abstraction, and Polymorphism | Cardelli, L. and Wegner, P. | 1985 | ACM Computing Surveys, 17(4), pp. 471-523 | DOI: 10.1145/6041.6042 |
| A Theory of Objects | Abadi, M. and Cardelli, L. | 1996 | Springer Monographs in CS | DOI: 10.1007/978-1-4419-8598-9 |
| On Understanding Data Abstraction, Revisited | Cook, W.R. | 2009 | OOPSLA 2009, pp. 557-572 | DOI: 10.1145/1640089.1640133 |
| The Principal Type-Scheme of an Object in Combinatory Logic | Hindley, J.R. | 1969 | Trans. AMS, 146, pp. 29-60 | DOI: 10.2307/1995158 — PDF: https://www.cs.tufts.edu/~nr/cs257/archive/roger-hindley/principal-type-scheme.pdf |
| A Theory of Type Polymorphism in Programming | Milner, R. | 1978 | JCSS, 17(3), pp. 348-375 | DOI: 10.1016/0022-0000(78)90014-4 |
| Principal Type-Schemes for Functional Programs | Damas, L. and Milner, R. | 1982 | POPL '82, pp. 207-212 | DOI: 10.1145/582153.582176 |
| Featherweight Go | Griesemer, R., Hu, R., Kokke, W., et al. | 2020 | ACM OOPSLA 2020, Article 149 | arXiv: 2005.11710 — https://arxiv.org/abs/2005.11710 |

### Escape Analysis

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| Escape Analysis on Lists | Park, Y.G. and Goldberg, B. | 1992 | PLDI '92, SIGPLAN 27(7), pp. 116-127 | DOI: 10.1145/143103.143125 — PDF: https://cs.nyu.edu/~goldberg/pubs/pg92.pdf |
| On the Complexity of Escape Analysis | Deutsch, A. | 1997 | POPL '97, pp. 358-371 | DOI: 10.1145/263699.263750 |
| Escape Analysis for Java | Choi, J.D., Gupta, M., Serrano, M.J., Sreedhar, V.C., Midkiff, S.P. | 1999 | OOPSLA '99, SIGPLAN 34(10), pp. 1-19 | DOI: 10.1145/320385.320386 |
| Escape Analysis for Object-Oriented Languages | Blanchet, B. | 1999 | OOPSLA '99, SIGPLAN 34(1), pp. 20-34 | DOI: 10.1145/320384.320387 |
| Escape Analysis for Java: Theory and Practice | Blanchet, B. | 2003 | ACM TOPLAS, 25(6), pp. 713-775 | (journal extension of 1999 paper) |

### Register Allocation & Code Generation

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| The Generation of Optimal Code for Arithmetic Expressions | Sethi, R. and Ullman, J.D. | 1970 | JACM, 17(4), pp. 715-728 | DOI: 10.1145/321607.321620 |
| Register Allocation via Coloring | Chaitin, G.J., Auslander, M.A., et al. | 1981 | Computer Languages, 6, pp. 47-57 | PDF: https://clei.org/proceedings_data/CLEI1981/TOMO%202/CLEI%20VIII%201981%20T.2_por%20capitulo-OCR/CLEI%20VIII%201981%20T.2_P13-P24_OCR.pdf |
| Register Allocation & Spilling via Graph Coloring | Chaitin, G.J. | 1982 | ACM SIGPLAN, 17(6), pp. 98-105 | DOI: 10.1145/872726.806984 |
| Improvements to Graph Coloring Register Allocation | Briggs, P., Cooper, K.D., Torczon, L. | 1994 | ACM TOPLAS, 16(3), pp. 428-455 | DOI: 10.1145/177492.177575 |
| Linear Scan Register Allocation | Poletto, M. and Sarkar, V. | 1999 | ACM TOPLAS, 21(5), pp. 895-913 | DOI: 10.1145/330249.330250 — PDF: https://web.cs.ucla.edu/~palsberg/course/cs132/linearscan.pdf |

### Optimization

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| Peephole Optimization | McKeeman, W.M. | 1965 | CACM, 8(7), pp. 443-444 | DOI: 10.1145/364995.365000 |
| A Unified Approach to Global Program Optimization | Kildall, G.A. | 1973 | POPL '73, pp. 194-206 | DOI: 10.1145/512927.512945 |
| Division by Invariant Integers using Multiplication | Granlund, T. and Montgomery, P.L. | 1994 | PLDI '94, pp. 61-72 | DOI: 10.1145/178243.178249 — PDF: https://gmplib.org/~tege/divcnst-pldi94.pdf |

### Graph Algorithms

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| Depth-First Search and Linear Graph Algorithms | Tarjan, R.E. | 1972 | SIAM J. Computing, 1(2), pp. 146-160 | DOI: 10.1137/0201010 — https://epubs.siam.org/doi/10.1137/0201010 — PDF: https://www.cs.cmu.edu/~cdm/resources/Tarjan1972-sccs.pdf |

### Garbage Collection

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| On-the-fly garbage collection: An exercise in cooperation | Dijkstra, E.W., Lamport, L., Martin, A.J., Scholten, C.S., Steffens, E.F.M. | 1978 | CACM, 21(11), pp. 966-975 | DOI: 10.1145/359642.359655 — PDF: https://lamport.azurewebsites.net/pubs/garbage.pdf — Archive: https://www.cs.utexas.edu/~EWD/transcriptions/EWD06xx/EWD630.html |

### Concurrency

| Paper | Authors | Year | Venue | DOI / URL |
|-------|---------|------|-------|-----------|
| Communicating Sequential Processes | Hoare, C.A.R. | 1978 | CACM, 21(8), pp. 666-677 | DOI: 10.1145/359576.359585 — PDF: https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf |
| Scheduling Multithreaded Computations by Work Stealing | Blumofe, R.D. and Leiserson, C.E. | 1999 | JACM, 46(5), pp. 720-748 | DOI: 10.1145/324133.324234 — PDF: https://www.csd.uwo.ca/~mmorenom/CS433-CS9624/Resources/Scheduling_multithreaded_computations_by_work_stealing.pdf |

---

## Go Official Documentation

| Resource | URL |
|----------|-----|
| Go Language Specification | https://go.dev/ref/spec |
| Go Assembler Guide | https://go.dev/doc/asm |
| Go 1.4 Release Notes | https://go.dev/doc/go1.4 |
| Go 1.5 Release Notes | https://go.dev/doc/go1.5 |
| Effective Go — Semicolons | https://go.dev/doc/effective_go#semicolons |
| Go Blog: Defer, Panic, and Recover | https://go.dev/blog/defer-panic-and-recover |
| Go Blog: Faster Maps with Swiss Tables | https://go.dev/blog/swisstable |
| Go Wiki: PanicAndRecover | https://go.dev/wiki/PanicAndRecover |
| Go Source: escape.go | https://go.dev/src/cmd/compile/internal/escape/escape.go |
| Go Source: select.go | https://go.dev/src/runtime/select.go |
| Go AST Package | https://pkg.go.dev/go/ast |
| Go Parser Package | https://pkg.go.dev/go/parser |

---

## Go Design Documents

| Document | Author | URL |
|----------|--------|-----|
| Scalable Go Scheduler Design Doc | Vyukov, D. (2012) | https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw |
| Contiguous Stacks Design Doc | Go Team (2013) | https://docs.google.com/document/d/1wAaf1rYoM4nCTUWO94WDGb73ZhIMZnJ_PkNAupvLa5o |
| Go Data Structures: Interfaces | Cox, R. (2009) | https://research.swtch.com/interfaces |
| Register-based Calling Convention Proposal | Go Issue #40724 | https://github.com/golang/go/issues/40724 |
| Escape Analysis Rewrite Discussion | Go Issue #23109 | https://github.com/golang/go/issues/23109 |
| Internal Calling Convention | Go Issue #18597 | https://github.com/golang/go/issues/18597 |

---

## Plan 9 / Bell Labs

| Resource | URL |
|----------|-----|
| Plan 9 from Bell Labs | https://9p.io/plan9/ |
| Plan 9 C Compilers (Thompson, Pike) | https://9p.io/sys/doc/compiler.html |
| How to Use the Plan 9 C Compiler (Pike) | https://9p.io/sys/doc/comp.html |
| Plan 9 Assembler Manual (Pike) | https://9p.io/sys/doc/asm.html |
| Inferno 6c source (origin of Go's reg.c) | http://code.google.com/p/inferno-os/source/browse/utils/6c/reg.c |
| UTF-8 History (Pike) | https://doc.cat-v.org/bell_labs/utf-8_history |
| Dijkstra Archive (EWD 630) | https://www.cs.utexas.edu/~EWD/transcriptions/EWD06xx/EWD630.html |

---

## Assembly & x86-64 Resources

### Official References

| Resource | URL |
|----------|-----|
| Intel 64 and IA-32 Software Developer's Manual | https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html |
| Intel SDM Vol 1 (Princeton mirror) | https://www.cs.princeton.edu/courses/archive/spr18/cos217/reading/x86-64-1.pdf |
| Intel Optimization Reference (Princeton) | https://www.cs.princeton.edu/courses/archive/spr18/cos217/reading/x86-64-opt.pdf |
| x86 Instruction Reference (Cloutier) | https://www.felixcloutier.com/x86/ |
| x86 Opcode Reference | http://ref.x86asm.net/ |

### Go Assembly

| Resource | URL |
|----------|-----|
| Go Assembler Guide (official) | https://go.dev/doc/asm |
| Design of the Go Assembler (Pike, GopherCon 2016) | https://go.dev/talks/2016/asm.slide |
| Go Assembly Complementary Reference (Sharipov) | https://www.quasilyte.dev/blog/post/go-asm-complementary-reference/ |
| Plan 9 Assembler Manual | https://9p.io/sys/doc/asm.html |

### Learning Assembly

| Resource | URL |
|----------|-----|
| The Faker's Guide to Reading x86 Assembly | https://www.timdbg.com/posts/fakers-guide-to-assembly/ |
| How to Read Assembly Language (Wolchok) | https://wolchok.org/posts/how-to-read-assembly-language/ |
| Learning to Read x86 Assembly (Shaughnessy) | https://patshaughnessy.net/2016/11/26/learning-to-read-x86-assembly-language |
| Let's Learn x86-64 Assembly Part 0 | https://gpfault.net/posts/asm-tut-0.txt.html |
| x86-64 Assembly Programming with Ubuntu (Jorgensen, free PDF) | http://www.egr.unlv.edu/~ed/assembly64.pdf |
| Same book, alternate URL | http://www.egr.unlv.edu/~ed/x86.html |
| Open Textbook Library entry | https://open.umn.edu/opentextbooks/textbooks/733 |
| CS107 Stanford x86-64 Reference Sheet | https://web.stanford.edu/class/cs107/resources/x86-64-reference.pdf |
| Brown University x86-64 Reference | https://cs.brown.edu/courses/csci1260/spring-2021/lectures/x86-64-assembly-language-reference.html |
| Exercism x86-64 Assembly Track (114 exercises) | https://exercism.org/tracks/x86-64-assembly |
| x86 Assembly on Wikibooks | https://en.wikibooks.org/wiki/X86_Assembly |
| x86 Instruction Listings (Wikipedia) | https://en.wikipedia.org/wiki/X86_instruction_listings |

### Calling Conventions

| Resource | URL |
|----------|-----|
| x86-64 Calling Conventions (Wayne's Talk) | https://waynestalk.com/en/x86-64-calling-conventions-en/ |
| The 64-bit x86 C Calling Convention (UVA) | https://aaronbloomfield.github.io/pdr/book/x86-64bit-ccc-chapter.pdf |
| x86 Calling Conventions (Wikipedia) | https://en.wikipedia.org/wiki/X86_calling_conventions |
| Calling Conventions (OSDev Wiki) | https://wiki.osdev.org/Calling_Conventions |
| Go internal ABI specification | https://tip.golang.org/src/cmd/compile/abi-internal |
| Go calling convention x86-64 (Dr. Knz) | https://dr-knz.net/go-calling-convention-x86-64.html |

---

## Books

### Free

| Book | Author | URL |
|------|--------|-----|
| x86-64 Assembly Language Programming with Ubuntu | Jorgensen, E. | http://www.egr.unlv.edu/~ed/assembly64.pdf |
| Communicating Sequential Processes (full book) | Hoare, C.A.R. (1985) | http://www.usingcsp.com/ |

### Commercial (Referenced)

| Book | Author | Year | ISBN |
|------|--------|------|------|
| Compilers: Principles, Techniques, and Tools (Dragon Book) | Aho, Sethi, Ullman | 1986 | 0-201-10088-6 |
| Compilers (Dragon Book, 2nd Ed) | Aho, Lam, Sethi, Ullman | 2006 | 0-321-48681-1 |
| Types and Programming Languages (TAPL) | Pierce, B.C. | 2002 | 0-262-16209-1 |
| Modern Compiler Implementation in C | Appel, A.W. | 1998 | 0-521-58390-X |
| Engineering a Compiler, 2nd Ed | Cooper, K.D. and Torczon, L. | 2011 | 978-0-12-088478-0 |
| The Art of Computer Programming, Vol. 2 | Knuth, D.E. | 1997 | 0-201-89684-2 |
| Hacker's Delight, 2nd Ed | Warren, H.S. Jr. | 2012 | 978-0-321-84268-8 |
| The Garbage Collection Handbook | Jones, R., Hosking, A., Moss, E. | 2011 | 978-1-4200-8279-1 |
| Assembly Language Step-by-Step, 3rd Ed | Duntemann, J. | 2009 | 978-0-470-49702-9 |
| Algorithms in C, 2nd Ed (Tarjan SCC p.482) | Sedgewick, R. | 1990 | 0-201-51425-7 |
| A Theory of Objects | Abadi, M. and Cardelli, L. | 1996 | 978-0-387-94775-4 |

---

## Blog Posts & Tutorials

### Go Internals

| Title | Author | URL |
|-------|--------|-----|
| Go Data Structures: Interfaces | Cox, R. | https://research.swtch.com/interfaces |
| Lexical Scanning in Go (talk) | Pike, R. | https://go.dev/talks/2011/lex.slide |
| Lexical Scanning in Go (video) | Pike, R. | https://www.youtube.com/watch?v=HxaD_trXwRE |
| Language Mechanics On Escape Analysis | Ardan Labs | https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-escape-analysis.html |
| Understanding Escape Analysis in Go | FreeCodeCamp | https://www.freecodecamp.org/news/understanding-escape-analysis-in-go/ |
| Stack Allocations and Escape Analysis | Go Perf Guide | https://goperf.dev/01-common-patterns/stack-alloc/ |
| Contiguous Stacks in Go | Anastasopoulos, A. | https://agis.io/post/contiguous-stacks-golang/ |
| Visual Guide to Go Maps | Sazak | https://sazak.io/articles/visual-guide-to-go-maps-hash-tables-2025-10-26 |
| Hash Tables Implementation in Go | Marawan | https://medium.com/kalamsilicon/hash-tables-implementation-in-go-48c165c54553 |
| Go's select statement: Demystified | Modi, S. | https://swatimodi.com/posts/go-select-demystified/ |
| Understanding the Go Compiler: The Parser | Internals for Interns | https://internals-for-interns.com/posts/the-go-parser/ |
| Handwritten Parsers & Lexers in Go | Gopher Academy | https://blog.gopheracademy.com/advent-2014/parsers-lexers/ |
| Register allocation in the Go compiler | Makarov, V. | https://vnmakarov.github.io/2024/09/24/register-allocation-in-the-go-compiler.html |
| SwissMap: A smaller, faster Golang Hash Table | DoltHub | https://www.dolthub.com/blog/2023-03-28-swiss-map/ |

### General Compiler / Assembly

| Title | URL |
|-------|-----|
| Parser generators vs handwritten parsers survey | https://notes.eatonphil.com/parser-generators-vs-handwritten-parsers-survey-2021.html |
| AST vs Parse Tree (GeeksforGeeks) | https://www.geeksforgeeks.org/compiler-design/abstract-syntax-tree-vs-parse-tree/ |
| Abstract vs Concrete Syntax Trees (Bendersky) | https://eli.thegreenplace.net/2009/02/16/abstract-vs-concrete-syntax-trees |

### Academic

| Title | URL |
|-------|-----|
| Analysis of the Go runtime scheduler (Columbia) | http://www.cs.columbia.edu/~aho/cs6998/reports/12-12-11_DeshpandeSponslerWeiss_GO.pdf |
| The Go Programming Language and Environment (CACM 2022) | https://m-cacm.acm.org/magazines/2022/5/260357-the-go-programming-language-and-environment/fulltext |
| Bruno Blanchet's Escape Analysis Research (INRIA) | https://bblanche.gitlabpages.inria.fr/escape-eng.html |
| Generalizations of Sethi-Ullman (Appel) | https://www.cs.princeton.edu/~appel/papers/sun.pdf |
| Sethi-Ullman Numbering Example (PDX) | https://web.cecs.pdx.edu/~apt/cs302_1999/lecture10/lecture10.html |
| Code Generation Sethi-Ullman (Karkare, IIT) | https://karkare.github.io/cs335/lectures/19SethiUllman.pdf |

---

## Tools & Interactive

| Tool | URL |
|------|-----|
| Compiler Explorer (Godbolt) | https://godbolt.org |
| GMP Library Documentation | https://gmplib.org/manual/ |
| TCMalloc Design Documentation | https://google.github.io/tcmalloc/design.html |
| TCMalloc GitHub Repository | https://github.com/google/tcmalloc |
| RFC 3629: UTF-8 | https://www.rfc-editor.org/rfc/rfc3629 |

---

## Source Code Repositories

| Repository | URL |
|------------|-----|
| Go source (GitHub) | https://github.com/golang/go |
| Go source: escape.go | https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/escape.go |
| Go source: graph.go (escape) | https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/graph.go |
| Go source: select.go | https://github.com/golang/go/blob/master/src/runtime/select.go |
| Go: A Documentary (history) | https://golang.design/history/ |
| Our fork | https://github.com/shishir-ctrl/go |

---

## Keywords & Search Terms

These are the technical terms and concepts covered in this project,
useful for further research.

### Compiler Architecture
```
LALR(1) parser, YACC, Bison, context-free grammar, shift-reduce,
abstract syntax tree (AST), parse tree, concrete syntax tree,
recursive descent parser, operator precedence, left-recursive grammar,
shift/reduce conflict, reduce/reduce conflict, lookahead set
```

### Lexical Analysis
```
lexer, tokenizer, scanner, finite automaton, NFA, DFA, Thompson's
construction, regular expression, UTF-8, Unicode, rune, BOM,
automatic semicolon insertion (ASI), lexical scanning, token stream
```

### Type Systems
```
structural typing, nominal typing, type inference, Hindley-Milner,
Algorithm W, principal type, untyped constant, interface satisfaction,
duck typing, structural subtyping, method set, type assertion,
type switch, assignability, convertibility, comparable types
```

### Escape Analysis
```
escape analysis, stack allocation, heap allocation, data flow graph,
flow(dst,src) edge, theSink, strongly connected component (SCC),
Tarjan's algorithm, loop depth, reachability analysis, flood fill,
EscNone, EscHeap, EscReturn, EscScope, connection graph,
abstract interpretation, pointer analysis, alias analysis
```

### Code Generation
```
Sethi-Ullman numbering, register allocation, graph coloring,
interference graph, live range, liveness analysis, data flow analysis,
reaching definitions, use-def chain, def-use chain, basic block,
control flow graph (CFG), dominator tree, SSA form, peephole
optimization, instruction selection, code emission, object file
```

### Register Allocation
```
graph coloring, Chaitin's algorithm, Briggs' improvement, linear
scan, spilling, register pressure, live variable analysis, iterative
data flow, worklist algorithm, fixed point, lattice theory
```

### Optimization
```
constant folding, constant propagation, dead code elimination,
common subexpression elimination (CSE), copy propagation,
strength reduction, magic number division, loop-invariant code
motion, function inlining, hairiness budget, peephole optimization
```

### Runtime / Scheduler
```
goroutine, M:P:G model, Machine (M), Processor (P), Goroutine (G),
work stealing, run queue, local queue, global queue, GOMAXPROCS,
preemption, sysmon, syscall handling, park/unpark, schedule(),
hand-off (P handoff), spinning thread
```

### Garbage Collection
```
mark and sweep, tri-color abstraction, white/grey/black, write
barrier, stop-the-world (STW), concurrent sweep, lazy sweeping,
GOGC, next_gc, GC rate, precise GC, conservative GC, bitmap,
root set, stack scanning, finalizer
```

### Memory Allocation
```
TCMalloc, size class, MHeap, MCentral, MCache, MSpan, tiny
allocator, arena, mmap, munmap, madvise, MADV_DONTNEED, slab
allocator, free list, bump pointer, page size, virtual memory
```

### Channels & Concurrency
```
CSP (Communicating Sequential Processes), channel, buffered channel,
unbuffered channel, send queue, receive queue, select statement,
case randomization, channel direction (send-only, receive-only),
Hchan struct, WaitQ, mutex, semaphore
```

### Stack Management
```
contiguous stacks, segmented stacks, hot split problem, stack
copying, stack guard, morestack, stack growth, stack shrinking,
split stack, copystack, stackguard
```

### Defer/Panic/Recover
```
defer stack, LIFO order, panic unwinding, recover, deferred
function, Defer struct, panic chain, gorecover, gopanic
```

### Assembly / x86-64
```
Plan 9 assembly, AT&T syntax, Intel syntax, pseudo-register,
FP (frame pointer), SB (static base), SP (stack pointer), TLS
(thread-local storage), SYSCALL instruction, REX prefix, ModR/M,
SIB byte, instruction encoding, calling convention, System V ABI,
all-stack ABI, register-based ABI, MOVQ, ADDQ, CMPQ, JMP, CALL,
RET, SUBQ, LEAQ, TESTQ, XORQ, FLAGS register, ZF, SF, CF, OF
```

### Build System
```
bootstrap, make.bash, cmd/dist, 6g (Go compiler), 6c (Plan 9 C
compiler), 6a (assembler), 6l (linker), .6 object file, ELF,
Mach-O, PE, static linking, cross-compilation, GOROOT, GOARCH,
GOOS, CGO_ENABLED
```

---

## Statistics

- **Total unique URLs**: 120+
- **Academic papers with DOIs**: 25
- **Free resources**: 30+
- **Research agents used**: 6
- **Total agent tool calls**: 186+
- **Topics covered**: 15 major areas
- **Keywords catalogued**: 200+
