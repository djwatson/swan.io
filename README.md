# swan.io
Swan scheme - let's implement a scheme!

# Motivation

Everyone probably writes a scheme at one point.  Probably everyone's
dog has a scheme.  Scheme is great because it is one of the few
langauges set up to be easy to implement, while still being relatively
easy to use.

# Constraints

Swan exists solely to win at the [R7RS Benchmarks](https://github.com/ecraven/r7rs-benchmarks), maintained by @ecraven.

To get up to speed quickly, LLVM is used as backend.  In order to
ensure top performance, we *don't* keep a shadow stack of variables,
instead we use conservative garbage collection and use the normal C
stack.

Linux supports unlimited stack size, so we use that, although most
benchmarks don't stress the stack size much.
[[clang::musttail]] is used for tail calls:  We *do* use a lookaside
stack after 6 variables: Because clang requires the param count of
caller and callee must match.  musttail requires clang 13.

For call/cc, I was going to implement something along the lines of
[Exceptional Continuations in
Javascript](http://www2.ift.ulaval.ca/~dadub100/sfp2007/procPaper4.pdf),
but it turns out none of the benchmarks actually needs full
continuations.  

# The benchmarks

Lots of things are tested, including dynamic typing, vectors, all
sorts of closures, nested loops, etc, etc.  

But a few things aren't tested at all: Only simple single call jump-up
call/cc is tested.  No libraries, dynamic-wind, eval, cyclic type
reading/writing, or macros are tested at all.

# Passes

Current passes in swan scheme are:

* Lexer, using re2c. 
* A simple handwritten parser to parse to s-expressions. 

(They say scheme is easy to parse - and it is! but only to s-exp.
The actual code is *very* hard).

*  Sexp to bytecode.  Currently bytecode is very similar to llvm (and
  other) SSA form, with the major deviation being nested lambdas.


*  This handles, and then removes, *all* symbol resolution, including
   from macros.  After this pass, symbols that point to differing
   in-memory symbol objects are different, even if they have the same
   name.


* A (bad and full of bugs) syntax-rules macro expander is
    included. None of the benchmarks actually exercise a macro
    expander, so I haven't spent much time on this.


* Quasiquote expansion is also handled here, since we need symbol
    information.

* Assignment conversion by boxing.

* Optimize direct calls: 


```scheme
  ((lambda (x) x) 10)
  10
```

* Copy propagation.  Once assignment conversion is done, we no longer
  need to track `let`s directly.

* Closure conversion.  This happens pretty early in the pipeline - and
  was probably a mistake in some ways to do so, since some
  optimizations are easier before closure conversion.


* Global function gathering and optimization.  For any globally
  `define`ed function, if it is not set or defined more than once, we
  can link directly to it instead of calling it using late-binding.
  This also allows inlining. This pass is also called 'known function'
  optimization.


* Inlining and tail-recursion optimization.  Inlining is pretty dumb:
  We don't track anything about the function except length.  As long
  as inlining won't add more than X number of instructions, it is
  inlined.  It is depth first: If we can't inline all the
  sub-functions in a function, then we residualize a call instead of
  inlining sub-calls.


  Tail recursion elimination here is also simple: If any function or
  inlined function calls itself in a tail-recursive fashion, we make
  it a loop call instead. 

* Sparse conditional constant propagation.  Pretty much straight from
  the paper.  We track variable types, as well as concrete constants.
  Currently only per-function, there is no global information.


* Strong dead code elimination. Any code that can be proven to never
  run is dropped.  This pass also drops any allocations that are set
  but never used:  This is currently the only form of escape analysis
  implemented.


* Store/load forwarding.  More of a hack around closure creation
  happening so early: If a function is inlined in to the function that
  created its closure, we can forward the references to the original
  variables instead (and probably drop the closure entirely)


* Recursion modulo cons: This is a simple pass that converts
  tail-recursion-modulo-cons in to normal tail recursion with
  accumulator.  Only cons in cdr position is currently implemented.
  This is used to great effect in the 'divrec' and 'primes'
  benchmarks - we can run these in constant stack space, and run
  almost entirely in L3 cache.


* Run to a fixpoint:


  * SSI conditionals: Turn any conditional information known in to
    separate SSA variables for each branch


  * This is then used in SCCP pass


  * And in a jump threading pass.


  * We then remove the SSI'd conditionals
 

  * And run DCE and SSA cleanup passes.


  * The final pass is 'register allocatin', which in our case just means
  a pretty form of getting out of SSA form.  


* Finally, we emit C++ code with a template based emitter.  It's
  pretty simple.


# GC

Many of the results depend heavily on the strength of the GC.  Swan uses Conservative RCImmix, with minor variations.

* Conservative: We scan the stack conservatively. Interior pointers
  are supported.  To do this easily, we allocate not from a single 
  block / line, but from size-segregated lines/sizes, similar to D or
  the boehm GC.


* The nursury size, or when we choose to scan next, is somewhat
  dynamic: Some tests can run entirely in L3, while others allocate up
  to a Gigabyte before deallocating anything.  So if not much memory
  is freed up on each run, we choose to run longer before running the
  collector again. (see dynamic, and paraffins)


* Defragmentation, both proactive and reactive, aren't implemented.  A
  couple benchmarks might use less space with this, but it's unlikely
  to impact runtimes in a big way. ('graphs' test)


* There is no cycle collection.  Conform does create cycles, but with
  a large enough collection time, most of them are discarded without
  ever being seen.  


* Other than the above issues, RcImmix has been <em>great</em>.  It slays on
  gcbench for example, running nearly 2x faster than anything else,
  while still using less memory.  Coalescing reference counts is
  essential, but has little performance penalty once implemented.

# Dynamic types

* 61-bit fixnums, that overflow to flonums.

* 61-bit flonums. Note that many of the flonum benchmarks are substantially faster than chez, because chez boxes all flonums.

* Everything else is boxed, with a header word for GC (Needs a reference count field).

# Safety

* All types are typechecked.  SCCP pass removes most of these typechecks.

* Currently there is no vector or string range checking.  SCCP doesn't track ranges.

* Symbols, either data or function calls, aren't checked at runtime
  for unset values.  (Although most of these checks could probably be
  optimized away).
  
* Fixnums to overflow correctly.  Actually this is a huge area of
  performance penalty: Things like fib could go quite a bit faster
  with more type hints - we can guarantee fibfp is always a flonum,
  but not that fib is always a fixnum, since it may overflow.

# Benchmark results v0.2

| benchmark  | swan  | chez  | diff        | analysis                                                 |
|------------|-------|-------|-------------|----------------------------------------------------------|
| browse     | 1.00  | 1.295 | -29.5       |                                                          |
| deriv      | 1.583 | 1.574 | 0.56854075  |                                                          |
| destruc    | 1.896 | 2.166 | -14.240506  |                                                          |
| diviter    | 1.332 | 1.492 | -12.012012  |                                                          |
| divrec     | 1.405 | 2.207 | -57.081851  |                                                          |
| puzzle     | 2.49  | 1.88  | 24.497992   | Needs direct array indexing                              |
| triangl    | 2.393 | 1.921 | 19.724196   | Needs inlined symbols, instead of indirection at runtime |
| tak        | 2.166 | 2.946 | -36.011080  |                                                          |
| takl       | 2.489 | 4.78  | -92.044998  |                                                          |
| ntakl      | 2.471 | 4.779 | -93.403480  |                                                          |
| cpstak     | 5.22  | 4.141 | 20.670498   | ???                                                      |
| ctak       | 1.132 | .984  | 13.074205   | Needs faster stack allocated closures                    |
| fib        | 3.551 | 5.799 | -63.306111  |                                                          |
| fibc       | .780  | .981  | -25.769231  |                                                          |
| fibfp      | 1.012 | 4.13  | -308.10277  |                                                          |
| sum        | 2.68  | 3.874 | -44.552239  |                                                          |
| sumfp      | 1.882 | 4.477 | -137.88523  |                                                          |
| fft        | .503  | 2.873 | -471.17296  |                                                          |
| mbrot      | 1.108 | 7.151 | -545.39711  |                                                          |
| mrotZ      |       | 6.877 | -687.7 / 0  | complex unimplemented                                    |
| nucleic    | .929  | 4.562 | -391.06566  |                                                          |
| pi         |       | 99999 | -9999900./0 | bignums unimplemented                                    |
| pnpoly     | 2.167 | 4.906 | -126.39594  |                                                          |
| ray        | 1.341 | 4.862 | -262.56525  |                                                          |
| simplex    | 1.376 | 3.804 | -176.45349  |                                                          |
| ack        | 2.367 | 2.688 | -13.561470  |                                                          |
| array1     | 5.67  | 6.663 | -17.513228  |                                                          |
| string     | .49   | 4.478 | -813.87755  |                                                          |
| sum1       | 1.12  | 1.488 | -32.857143  | needs 'round to odd' for 61-bit flonums                  |
| cat        | 2.124 | 2.199 | -3.5310734  |                                                          |
| tail       | 3.44  | 99999 | -2906847.7  |                                                          |
| wc         | 1.93  | 1.372 | 28.911917   | input/output needs buffering                             |
| read1      | .661  | .978  | -47.957640  |                                                          |
| compiler   | 3.13  | 99999 | -3194756.2  |                                                          |
| conform    | 2.899 | 4.375 | -50.914108  |                                                          |
| dynamic    | 2.383 | 2.547 | -6.8820814  |                                                          |
| earley     | 2.819 | 4.503 | -59.737496  |                                                          |
| graphs     | 4.00  | 2.337 | 41.575      | needs lambda-lifing for triple-nested loops              |
| lattice    | 3.113 | 3.874 | -24.445872  |                                                          |
| matrix     | 1.439 | 1.663 | -15.566366  |                                                          |
| maze       | .770  | 1.598 | -107.53247  |                                                          |
| mazefun    | 2.151 | 3.071 | -42.770804  |                                                          |
| nqueens    | 3.846 | 4.377 | -13.806552  |                                                          |
| paraffins  | 3.83  | 6.138 | -60.261097  |                                                          |
| parsing    | 2.084 | 3     | -43.953935  |                                                          |
| peval      | 1.920 | 2.37  | -23.4375    |                                                          |
| primes     | .782  | .997  | -27.493606  |                                                          |
| quicksort  | 3.519 | 99999 | -2841588.0  |                                                          |
| scheme     | 2.91  | 2.97  | -2.0618557  |                                                          |
| slatex     | 5.894 | 3.59  | 39.090601   | input/output needs buffering                             |
| chudnovsky |       | .328  | -32.8 / 0   | bignums unimplemented                                    |
| nboyer     | 2.016 | 2.67  | -32.440476  |                                                          |
| sboyer     | 1.151 | 1.24  | -7.7324066  |                                                          |
| gcbench    | .535  | .992  | -85.420561  |                                                          |
| mperm      | 9.603 | 11.16 | -12.693123  |                                                          |
| equal      |       | .67   | -67./0      | cycle equality hash checking unimplemented               |
| bv2string  |       | 1.29  | -129./0     | bytevector unimplemented                                 |
|            |       |       |             |                                                          |

