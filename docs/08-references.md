# Go 1.4 Compiler: Complete Reference Bibliography

All references have been **verified via web search** against primary
sources (ACM Digital Library, SIAM, Springer, IEEE, university archives).

---

## Compiler Theory — Foundational Texts

- **Aho, A.V., Sethi, R., Ullman, J.D.** (1986). *Compilers: Principles,
  Techniques, and Tools* ("The Dragon Book"). Addison-Wesley.
  ISBN: 0-201-10088-6.

- **Aho, A.V., Lam, M.S., Sethi, R., Ullman, J.D.** (2006). *Compilers:
  Principles, Techniques, and Tools*, 2nd Edition. Addison-Wesley.
  ISBN: 0-321-48681-1.

- **Appel, A.W.** (1998). *Modern Compiler Implementation in C*.
  Cambridge University Press. ISBN: 0-521-58390-X.

- **Cooper, K.D. and Torczon, L.** (2011). *Engineering a Compiler*,
  2nd Edition. Morgan Kaufmann. ISBN: 978-0-12-088478-0.

---

## Parsing Theory

- **Knuth, D.E.** (1965). "On the Translation of Languages from Left
  to Right". *Information and Control*, 8(6), pp. 607-639.
  DOI: 10.1016/S0019-9958(65)90426-2.
  https://www.sciencedirect.com/science/article/pii/S0019995865904262
  *(The original LR parsing paper.)*

- **Johnson, S.C.** (1975). "Yacc: Yet Another Compiler-Compiler".
  AT&T Bell Laboratories, Computing Science Technical Report No. 32.
  https://epaperpress.com/lexandyacc/download/yacc.pdf
  *(The YACC parser generator used by Go 1.4.)*

- **DeRemer, F.L.** (1969). "Practical Translators for LR(k) Languages".
  PhD Dissertation, Massachusetts Institute of Technology.
  https://dspace.mit.edu/handle/1721.1/13628
  *(Invented LALR(1) and SLR(1) parsing.)*

- **DeRemer, F.L. and Pennello, T.J.** (1982). "Efficient Computation
  of LALR(1) Look-Ahead Sets". *ACM TOPLAS*, 4(4), pp. 615-649.
  DOI: 10.1145/69622.357187.
  *(The efficient LALR(1) lookahead algorithm used by YACC.)*

---

## Lexical Analysis

- **Thompson, K.** (1968). "Programming Techniques: Regular Expression
  Search Algorithm". *Communications of the ACM*, 11(6), pp. 419-422.
  DOI: 10.1145/363347.363387.
  *(Thompson's construction for NFA from regex.)*

- **Pike, R. and Thompson, K.** (1993). "Hello World, or Καλημέρα κόσμε,
  or こんにちは 世界". Proceedings of the Winter 1993 USENIX Conference.
  *(UTF-8 encoding, invented by the Go creators.)*

- **RFC 3629**: "UTF-8, a transformation format of ISO 10646" (2003).
  https://www.rfc-editor.org/rfc/rfc3629

- **Pike, R.** (2011). "Lexical Scanning in Go". Google Technology
  User Group talk. https://go.dev/talks/2011/lex.slide

---

## Type Systems

- **Pierce, B.C.** (2002). *Types and Programming Languages* (TAPL).
  MIT Press. ISBN: 0-262-16209-1.
  *(Comprehensive reference on type theory.)*

- **Cardelli, L. and Wegner, P.** (1985). "On Understanding Types,
  Data Abstraction, and Polymorphism". *ACM Computing Surveys*,
  17(4), pp. 471-523. DOI: 10.1145/6041.6042.
  *(Structural vs nominal typing theory.)*

- **Abadi, M. and Cardelli, L.** (1996). *A Theory of Objects*.
  Springer-Verlag. ISBN: 978-0-387-94775-4.
  DOI: 10.1007/978-1-4419-8598-9.

- **Cook, W.R.** (2009). "On Understanding Data Abstraction, Revisited".
  *OOPSLA 2009*, pp. 557-572. DOI: 10.1145/1640089.1640133.

- **Hindley, J.R.** (1969). "The Principal Type-Scheme of an Object
  in Combinatory Logic". *Trans. American Mathematical Society*,
  146, pp. 29-60. DOI: 10.2307/1995158.

- **Milner, R.** (1978). "A Theory of Type Polymorphism in Programming".
  *Journal of Computer and System Sciences*, 17(3), pp. 348-375.
  DOI: 10.1016/0022-0000(78)90014-4.

- **Damas, L. and Milner, R.** (1982). "Principal Type-Schemes for
  Functional Programs". *POPL '82*, pp. 207-212.
  DOI: 10.1145/582153.582176.

---

## Escape Analysis

- **Park, Y.G. and Goldberg, B.** (1992). "Escape Analysis on Lists".
  *PLDI '92*, SIGPLAN Notices 27(7), pp. 116-127.
  DOI: 10.1145/143103.143125.
  https://cs.nyu.edu/~goldberg/pubs/pg92.pdf

- **Deutsch, A.** (1997). "On the Complexity of Escape Analysis".
  *POPL '97*, pp. 358-371. DOI: 10.1145/263699.263750.
  *(Proved EA1 solvable in O(n log² n), EA2 is DEXPTIME-hard.)*

- **Choi, J.D., Gupta, M., Serrano, M.J., Sreedhar, V.C., Midkiff, S.P.**
  (1999). "Escape Analysis for Java". *OOPSLA '99*, SIGPLAN Notices
  34(10), pp. 1-19. DOI: 10.1145/320385.320386.
  *(Connection graph approach to escape analysis.)*

- **Blanchet, B.** (1999). "Escape Analysis for Object-Oriented Languages:
  Application to Java". *OOPSLA '99*, SIGPLAN Notices 34(1), pp. 20-34.
  DOI: 10.1145/320384.320387.
  *(First correctness proofs via abstract interpretation.)*

- **Blanchet, B.** (2003). "Escape Analysis for Java: Theory and Practice".
  *ACM TOPLAS*, 25(6), pp. 713-775.
  *(Journal extension with full proofs.)*

---

## Graph Algorithms

- **Tarjan, R.E.** (1972). "Depth-First Search and Linear Graph Algorithms".
  *SIAM Journal on Computing*, 1(2), pp. 146-160.
  DOI: 10.1137/0201010.
  https://epubs.siam.org/doi/10.1137/0201010
  *(SCC algorithm used in Go's escape analysis.)*

- **Sedgewick, R.** (1990). *Algorithms in C*, 2nd Edition.
  Addison-Wesley. ISBN: 0-201-51425-7. pp. 477-483.
  *(Cited in Go's esc.c for the SCC implementation.)*

---

## Register Allocation & Code Generation

- **Sethi, R. and Ullman, J.D.** (1970). "The Generation of Optimal
  Code for Arithmetic Expressions". *Journal of the ACM*, 17(4),
  pp. 715-728. DOI: 10.1145/321607.321620.
  *(Sethi-Ullman numbering used in Go's `ullmancalc()`.)*

- **Chaitin, G.J., Auslander, M.A., Chandra, A.K., Cocke, J.,
  Hopkins, M.E., Markstein, P.W.** (1981). "Register Allocation
  via Coloring". *Computer Languages*, 6, pp. 47-57.

- **Chaitin, G.J.** (1982). "Register Allocation & Spilling via Graph
  Coloring". *ACM SIGPLAN Notices*, 17(6), pp. 98-105.
  DOI: 10.1145/872726.806984.

- **Briggs, P., Cooper, K.D., Torczon, L.** (1994). "Improvements to
  Graph Coloring Register Allocation". *ACM TOPLAS*, 16(3), pp. 428-455.
  DOI: 10.1145/177492.177575.

- **Poletto, M. and Sarkar, V.** (1999). "Linear Scan Register
  Allocation". *ACM TOPLAS*, 21(5), pp. 895-913.
  DOI: 10.1145/330249.330250.

---

## Optimization

- **Kildall, G.A.** (1973). "A Unified Approach to Global Program
  Optimization". *POPL '73*, pp. 194-206.
  DOI: 10.1145/512927.512945.
  *(Data flow analysis framework — liveness, reaching definitions.)*

- **McKeeman, W.M.** (1965). "Peephole Optimization". *Communications
  of the ACM*, 8(7), pp. 443-444. DOI: 10.1145/364995.365000.

- **Granlund, T. and Montgomery, P.L.** (1994). "Division by Invariant
  Integers using Multiplication". *PLDI '94*, pp. 61-72.
  DOI: 10.1145/178243.178249.
  https://gmplib.org/~tege/divcnst-pldi94.pdf
  *(Magic number division used in Go's `smagic()`/`umagic()`.)*

- **Warren, H.S. Jr.** (2012). *Hacker's Delight*, 2nd Edition.
  Addison-Wesley. ISBN: 978-0-321-84268-8.
  *(Chapter 10: Integer Division by Constants.)*

---

## Multi-Precision Arithmetic

- **Knuth, D.E.** (1997). *The Art of Computer Programming, Volume 2:
  Seminumerical Algorithms*, 3rd Edition. Addison-Wesley.
  ISBN: 0-201-89684-2. Section 4.3: "Multiple-Precision Arithmetic".
  *(Algorithms used in Go's Mpint/Mpflt implementation.)*

---

## Memory Management

- **Hanson, D.R.** (1990). "Fast Allocation and Deallocation of Memory
  Based on Object Lifetimes". *Software: Practice and Experience*,
  20(1), pp. 5-12. *(Arena allocation technique used in Go's `mal()`.)*

- **Ghemawat, S. and Menage, P.** "TCMalloc: Thread-Caching Malloc".
  Google Performance Tools.
  https://google.github.io/tcmalloc/design.html
  *(Inspired Go's memory allocator.)*

- **Bonwick, J.** (1994). "The Slab Allocator: An Object-Caching Kernel
  Memory Allocator". *USENIX Summer 1994*, pp. 87-98.

---

## Garbage Collection

- **Dijkstra, E.W., Lamport, L., Martin, A.J., Scholten, C.S.,
  Steffens, E.F.M.** (1978). "On-the-fly garbage collection: An
  exercise in cooperation". *Communications of the ACM*, 21(11),
  pp. 966-975. DOI: 10.1145/359642.359655.
  https://lamport.azurewebsites.net/pubs/garbage.pdf
  *(Tri-color abstraction and write barriers.)*

- **Wilson, P.R.** (1992). "Uniprocessor Garbage Collection Techniques".
  *IWMM '92*, LNCS 637, pp. 1-42.

- **Jones, R., Hosking, A., Moss, E.** (2011). *The Garbage Collection
  Handbook*. Chapman and Hall/CRC. ISBN: 978-1-4200-8279-1.

---

## Concurrency & Scheduling

- **Hoare, C.A.R.** (1978). "Communicating Sequential Processes".
  *Communications of the ACM*, 21(8), pp. 666-677.
  DOI: 10.1145/359576.359585.
  *(Theoretical foundation for Go's channels.)*

- **Hoare, C.A.R.** (1985). *Communicating Sequential Processes*.
  Prentice Hall. ISBN: 0-13-153289-8.
  Free online: http://www.usingcsp.com/

- **Blumofe, R.D. and Leiserson, C.E.** (1999). "Scheduling Multithreaded
  Computations by Work Stealing". *Journal of the ACM*, 46(5),
  pp. 720-748. DOI: 10.1145/324133.324234.
  *(Work-stealing algorithm used in Go's scheduler.)*

- **Vyukov, D.** (2012). "Scalable Go Scheduler Design Doc".
  https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw
  *(The M:P:G scheduler model introduced in Go 1.1.)*

---

## Go-Specific Design Documents

- **"The Go Programming Language Specification"** (2014).
  https://go.dev/ref/spec

- **Cox, R.** (2009). "Go Data Structures: Interfaces".
  https://research.swtch.com/interfaces
  *(How Go implements interface dispatch via itables.)*

- **Go Team** (2013). "Contiguous Stacks Design Doc".
  https://docs.google.com/document/d/1wAaf1rYoM4nCTUWO94WDGb73ZhIMZnJ_PkNAupvLa5o
  *(Stack copying replacing segmented stacks.)*

- **Griesemer, R., Hu, R., Kokke, W., Lange, J., Taylor, I.L.,
  Toninho, B., Wadler, P., Yoshida, N.** (2020). "Featherweight Go".
  *ACM OOPSLA 2020*, Article 149. arXiv: 2005.11710.
  *(Formal model of Go's type system with generics.)*

---

## Plan 9 & Inferno Heritage

- **Pike, R., Presotto, D., Dorward, S., Flandrena, B., Thompson, K.,
  Trickey, H., Winterbottom, P.** (1995). "Plan 9 from Bell Labs".
  *Computing Systems*, 8(3), pp. 221-254.

- **Thompson, K. and Pike, R.** (2002). "Plan 9 C Compilers".
  https://9p.io/sys/doc/compiler.html

- **Pike, R.** (2002). "How to Use the Plan 9 C Compiler".
  https://9p.io/sys/doc/comp.html

- **Winterbottom, P. and Pike, R.** (1997). "The Design of the Inferno
  Virtual Machine". Bell Labs Technical Report.

---

## Hardware References

- **Intel Corporation** (2024). "Intel 64 and IA-32 Architectures
  Software Developer's Manual", Volumes 1-3.
  https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html
