---
title: "Compiler Performance and LLVM"
date: 2019-03-09T07:50:45+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Performance"
tags:
  - "Cone"
---

*Note: You can read Russian translation 
[here](http://softdroid.net/proizvoditelnost-kompilyatora-i-llvm).*

I have always wanted the Cone compiler to be fast.
Faster build times make it easier and more pleasant for the programmer
to iterate and experiment quickly.
Compilers built for speed, such as Turbo Pascal, D and Go, win praise and loyalty.
Languages that struggle more with this, like C++ or Rust, get regular complaints and requests
to speed up build times.

Compiler speed is also important to Jonathan Blow.
In his videos, he demonstrates the ability to rebuild a large Jai program in seconds
and claims the Jai compiler can build 100kloc in a second.
I would like Cone to match that velocity, so long as it does not interfere in other priorities.

To that end, I designed the Cone compiler for performance,
following the principles enumerated in [Premature Optimization](/post/premature-optimization).
In this post, let me share with you the coding choices I made
and some recent performance measurements that evaluate whether those choices were sound.
These numbers lead us inevitably to some fascinating questions about the
elephant in the room:  LLVM's performance profile.

## Architecting for Performance ##

Let's begin with the Cone compiler design choices made for performance:

* **C**. Choosing any systems language (C++, Rust, D) will likely result in measurably faster programs.
  From my standpoint, choosing C improves the odds, as it makes it easier to get
  closer to the metal. To manage performance even tighter,
  the compiler uses only one external library.
  Although some might argue C has slowed my productivity,
  bloated the code base, and made it susceptible to all manner of unsafety bugs,
  that has not been my experience. My only regret
  is I wish I could have self-hosted it in Cone instead.
  
* **Arena memory management**. Compilers allocate many tiny pieces of data,
  largely for AST/IR nodes and symbol tables. To nearly eliminate the non-trivial runtime cost
  of malloc(), free() and liveness tracking for each data piece,
  the Cone compiler uses a very simple bump-pointer arena allocator.
  Allocation takes just a few instructions to extract the next bite out of a large, malloced arena. 
  References are never tracked nor freed, and yet are 100% safe to use.
  The theoretical downside for this strategy is that memory consumption grows non-stop.
  However, in practice, I don't fear the compile ever running out of memory.
  On average, the Cone compiler consumes about 35 bytes of memory for every source code byte.
  How big does your source code need to be before you would consume all memory?
  
* **Data is rarely copied**.
  Nearly all data structures, including the IR and global name table, are mutable and reusable.
  Furthermore, the design of the IR means that most IR nodes survive largely intact
  all the way from parse to gen. Even temporary, working data stacks are allocated once and reused.
  Additionally, string copies of loaded source file tokens happen at most once,
  when interning names and when capturing string literals.
  Compilers that rewrite their AST/IR nodes or use immutable data structures
  will incur a measurable allocation and copying performance cost for doing so.

* **Names are interned**.
  The lexer is responsible for interning all names into the global name table.
  As a result, the rest of the compiler (including parsing and name resolution)
  is never slowed down by any string comparison or hash table look-up.
  Correlating a name to an IR node is nearly as simple as dereferencing a pointer.
  Between lexing and generation, the compiler pays no attention to strings in any way,
  other than for error message generation.
  
* **Ad hoc, built-in heuristics**.
  Semantic analysis traverses the full IR only three times.
  No "solvers" are used for type inference or borrow-checking,
  as cascading chains of conditionals suffice.
  Additionally, none of the semantic analysis happens in abstraction layers
  (e.g., via core library templates or generics).
  
I make no claim to any unimpeachable evidence that these strategies are indeed the best
performing choices I could make.
If any vetted research exists that validates these choices, I am unaware of it.
I can point to Walter Bright's article [Increasing Compiler Speed By Over 75%](http://www.drdobbs.com/cpp/increasing-compiler-speed-by-over-75/240158941)
which quantifies the performance benefit of arena-based memory allocation.
I welcome any other hard data and/or recommendations regarding these
(or alternative) performance-based design choices.

## Cone Compiler Frontend Performance ##

The Cone compiler outputs a message summarizing elapsed time and memory usage for every compile.
Running on my server, as part of the Playground, small programs typically compile in
4-11 milliseconds. 

Although content with that, I recently got curious about what's happening under the covers.
I ran the Visual Studio profiler.
One surprising result was that most compilation steps (like
parsing and semantic analysis) were not even showing up on the log.

My hopeful guess was that these stages were happening more quickly than the 
milli-second probe speed of the profiler could notice.
To validate that guess, I instrumented the compiler code to use QueryPerformanceCounter,
which measures elapsed time at a precision of a tenth of a microsecond.

For a small program of 200 LOC (3800 bytes), here are the average elapsed-times on my laptop
for each front-end compiler stage:

μSec | Compiler Stage
-----|-----------------
 300 | **Load.** Completely load source program(s) from SSD disk to memory
 100 | **Lex.** Transform source UTF-8 characters to ready-to-use, interned tokens
 300 | **Parse.** Convert tokens into high-level IR nodes
 100 | **Analysis.** All 3 semantic passes: name resolve, type check, data flow

One should not read too much into these numbers.
Several stages will almost certainly slow down over time,
particularly when metaprogramming support is added to Cone.
Furthermore, we don't know how these results will scale to larger programs.
Even if the nature of the logic suggests scaling will likely be linear,
tests will need to be performed that confirm this.
The tests were run on my laptop competing with other programs for the CPU,
and the results varied ±50% from one run to another.

All caveats aside, these results are breathtaking.
The untuned, front-end compiler is chewing through 250Kloc per second,
despite nearly 40% of the time being spent on disk i/o.
These better-than-expected results
improve my confidence that Cone's upfront design choices were largely the right ones.

I do wonder why parsing is slower than lexing and analysis.
Two possibilities come to mind: every token is likely to be trial-matched
by the recursive descent parser many times across multiple functions
(which is less true of the lexer's handling of source code characters).
More intriguing is that this may have more do with the cost of building the Cone IR nodes.
As compared to lexing and analysis, parsing is by far doing the most memory mutation.
Would extensive memory mutation slow things down this much (e.g., because of cache invalidation)?

An astute reader will have noticed that I did not give a number for the code generation stage.
Here, unfortunately, the performance news is much less stellar.
Generating an optimized object file from the analyzed IR takes about 115,500 μSec.
That's 99.3% of the total compile time!
To understand this disappointing result, we must talk about LLVM.

## LLVM Backend Performance ##

Like many recent compilers, Cone uses LLVM to generate optimized native code.
Essentially, the compiler's code generation stage builds binary LLVM IR from the Cone IR,
asks LLVM to verify and optimize it, and then has LLVM generate an object
file for the appropriate target CPU architecture.

Overall, my experience using LLVM has been extremely positive.
I doubt I would have tackled building a compiler for Cone without the existence of LLVM.
My misgivings about LLVM's performance (and its large runtime/build footprint)
are the *only* significant concerns I have about using LLVM in Cone.

Similar to the front-end benchmarks shown above, I measured the 
elapsed-time for various LLVM-based backend stages,
processing the *same* small Cone program:

μSec    | LLVM Stage
--------|-----------------
  2,500 | **Setup.**
 11,000 | **Gen LLVM IR.** Build LLVM IR by traversing the Cone IR nodes
  3,000 | **Verify LLVMIR.** Verify the build LLVM IR
 15,000 | **Optimize.** Optimize the LLVM IR
 84,000 | **Gen obj.** Convert LLVM IR to an object file and save it

Before we can talk about what these numbers mean,
let's sketch out the high-level algorithmic work each stage performs:
 
* **Setup**. This gets LLVM ready to support code generation for the
  specified compile target (out of 15 supported target architectures).
  It initializes information about the target machine, data layout,
  and the global context.
  Should it really require 2.5msecs
  (3x longer than loading and parsing a small Cone program)
  to accomplish this work?
  It does feel potentially excessive.
  However, fortunately, this elapsed time is likely constant,
  and won't grow as the source programs get larger and more complex.

* **Gen LLVM IR**. This is the only stage that intermixes Cone
  and LLVM code. The Cone-side logic traverses the Cone IR similarly
  to the semantic analysis passes, issuing a lot of individual LLVM
  API calls which build the corresponding LLVM IR nodes, 
  effectively lowering Cone IR to LLVM IR.
  Since the Cone-side logic would likely not take longer than 100 μSecs
  (the cost of 3-passes of semantic analysis),
  building 1,000 LLVM IR SSA nodes evidently takes more than 100x longer.
  The algorithmic complexity here is likely O(n) for the most part,
  scaling roughly linearly to the quantity of Cone IR nodes.
  The process of lowering Cone IR types to LLVM types
  might not be O(n), as LLVM normalizes type info for uniqueness.
  
* **Verify LLVMIR**. This step is effectively semantic analysis for LLVM IR.
  It ensures the module's IR is well-formed, including type checking.
  This stage could probably be omitted in a production compiler that knows
  it only generates valid LLVM IR. 
  Although much quicker than other backend stages, it still takes 30x longer
  than all three of Cone's semantic passes.
  Its algorithmic complexity is likely O(n), scaling linearly to the
  number of LLVM IR nodes.
  
* **Optimize LLVMIR**. This performs 6 LLVM optimization passes:
  converting stack variables to registers, function inlining,
  simple peephole/bit-twiddling, expression re-association, common subexpression elimination,
  and control-flow graph simplification.
  These passes scan and digest the LLVM IR on every pass and likely rewrite the LLVM IR each time.
  Some of the optimization passes might be O(n), but others
  are almost certainly exponentially more complex.
  
* **Gen Obj.** This step lowers LLVM IR to the target's machine language and
  stores it out to disk as a target-specific object file.
  This final LLVM stage takes up 73% of LLVM's running time.
  We would expect this lowering step to take more time than the one before,
  partly because the input->outut data is larger (800 LOC LLVM IR turns into 1900 LOC of assembler),
  and partly because the lowering process is far more complex
  due to target-specific register allocation and other optimization algorithms.

## Does LLVM need to be this slow? ##

I want to be very careful when talking about LLVM and performance.
Although my intent is to offer helpful feedback and intelligence on what to expect from LLVM,
precedence suggests my words could be misconstrued as rude or unfair to the many people
that have invested 100s of man-years effort contributing to this amazing tool.
It would be a shame if people felt a stronger need to vigorously (and needlessly) defend LLVM
rather than engaging in a insightful dialogue that explores what we can learn.

However, when LLVM's performance plays a dominant role on compiler build times (99.3%!),
it is worth asking whether a backend truly needs to
consume almost 200x more of the CPU than the front-end.
If we want rapid compile times (and I do), we need to explore
ways to speed up the backend.

It is no secret that LLVM is big (1.2mloc of C++ code) and chews through many CPU cycles.
Every compiler that uses it (e.g., Clang, Rust, Swift) has a reputation
for slower-than-desired compile times.
By contrast, compilers with reputations for much faster compile times
(e.g., D, Go, tcc and Jai) do not.
Creating a nimbler, smaller code-generation backend library has been a key driver for a
number of LLVM alternatives,
such as the Rust-based [Cranelift](https://github.com/CraneStation/cranelift),
[libFirm](https://pp.ipd.kit.edu/firm/index.html), 
and [QBE](https://c9x.me/compile/).
This concern also drove the WebKit team to replace LLVM with their own
[FTL JIT backend](https://webkit.org/blog/5852/introducing-the-b3-jit-compiler/).

Can we glean any additional insights into LLVM's performance (and how to improve it)
based on the elapsed time results given above?
Certainly no definitive conclusions can be drawn,
given that these numbers are based on compiling a small, simple program.
We can make no sure predictions of how these results would change
after increasing program size or complexity.
A growth in program size alone would likely make the backend even
worse compared to the frontend. However, a complex program which
depends on many external packages or heavily uses metaprogramming 
might well be weighted more heavily towards frontend processing
(as [these Clang flame-chart profiler diagrams](https://aras-p.info/blog/2019/01/16/time-trace-timeline-flame-chart-profiler-for-Clang/) 
vividly demonstrate).

Given these caveats, let's explore potential explanations for why
LLVM often dominates the CPU and what might be done about it.

### Optimization Impact ###

A defender of LLVM's performance will claim the difference
between LLVM and its competitors lies in the quality of its optimization work.
It's a trade-off with diminishing returns:  invest a lot more time doing optimization
work to eek out somewhat faster-running programs.

There is truth here. Some forms of optimization **are** computationally expensive,
responding exponentially to complexity.
LLVM offers superior optimization,
noticeably better than tcc, for example, at a cost of much slower compilation (5-9x).

Does the cost have to be that steep? We can't say for sure, but surely it means something
that gcc and clang are very similar in both compile time and the
quality of optimizations. Perhaps a steep compile-time penalty 
is an unfortunate, but necessary cost for optimal execution speed.
That said, one such example is *not* proof that optimization work completely explains LLVM's speed.

There is a bit of good news, however, in the way LLVM support optimization.
LLVM makes it possible to improve compile time by performing fewer optimization passes.
For debug builds, this is an attractive option.
Based on my numbers, eliminating all optimization passes
would restore at most only 15% of LLVM's CPU use. That's underwhelming.<sup>1</sup>

### Using LLVM more Efficiently ###

Another explanation for LLVM's performance is that Cone might not be
setting LLVM up for success. Although there is a published guideline
called ["Performance Tips for Frontend Authors"](https://llvm.org/docs/Frontend/PerformanceTips.html),
it focuses on how to help the optimizer.
It offers no suggestions on how to make LLVM run faster.

In addition to eliminating optimization passes and IR verification,
could there be other ways for the Cone compiler to speed up LLVM processing?

* **Pre-optimized IR generation.** The smaller the IR being generated,
  the less work LLVM has to expend processing and optimizing it.

* **Normalized types.** Certain Cone types (e.g., references) are
  repetitively lowered to LLVM types, which are then normalized for uniqueness.
  If the Cone compiler were to pre-normalize these types, this would
  avoid the potentially more expensive cost for LLVM to do this work.

* **Faster LLVM IR building.** I read an article on Stack Overflow
  which suggested that LLVM can ingest a text-based LLVM IR file
  much faster than some compilers can build the IR using the API.
  It explained this counterintuitive outcome results from improved memory locality.
  It would be interesting to research whether the LLVM utilities have a backdoor
  way to build instructions more quickly, different from 
  the one-at-a-time instruction builder provided by the C++ (or C) API.
  
I have no reason to expect that these or other Cone-based changes would
result in huge speed ups to LLVM, but they are likely worth exploring.

### Back-end Architecture Improvements ###

Is it possible that the slowness of LLVM might also result from its architectural choices?
An imperfect comparison between several of the benchmark numbers opens up this possibility:

* The **LLVM IR verification** stage took 30x more time than 3 semantic analysis passes for Cone.
  The nature of the work being performed by both stages is vaguely similar. 
  Both have linear O(n) complexity.
  LLVM verification is typically handling more IR nodes than Cone's semantic analysis,
  but in both cases the type checks are similarly simple.
  It is not unreasonable to wonder whether LLVM *needs* to take 30x more time than Cone
  to perform its IR verification.
  Might it take less time if LLVM were built on different architectural choices?

* The **gen** stage, which lowers Cone IR to build LLVM IR,
  takes almost 40x longer than Cone's parsing, which lowers tokens to Cone IR.
  They both have largely linear O(n) complexity and the primary responsibility for
  both stages is to  build up an IR graph.
  The resulting LLVM IR graph will have more nodes than the Cone IR graph,
  but would that still require 40x more effort if more performant design choices were made?

A 30-40x performance gap is sizeable enough to warrant research
into whether there is some excisable fat in LLVM's design
not only in these stages, but quite possibly also spilling over into
the later, more time-consuming stages that also consume the same LLVM IR data structures.
If this excess fat is not due to LLVM's optimization and target architecture flexibility,
what else could be a significant factor?

[LuaJIT 2.0's new IR](http://wiki.luajit.org/SSA-IR-2.0#introduction)
offers inspiration for one promising line of attack.
This low-level IR design cleverly packs every SSA node into a 64-bit value.
Node cross-references take up only 16-bits.
All IR nodes for a function are packed together tightly.
Such a scheme is not just cache friendly, but also register friendly,
which would accrue huge performance benefits.
Packing nodes together also reduces memory allocation/free runtime overhead.

I am not suggesting that LLVM adopt LuaJIT's IR scheme, especially since
LuaJIT's IR is designed to support a simpler, dynamically-typed language.
Nodes for a more versatile LLVM-like IR would almost certainly require
more than 64-bits.
Rather, I am wondering what sort of performance gains might result
from redesigning the backend's low-level IR (and memory management)
along similar high-performance lines.<sup>2</sup>
Even if these improvements only ensure faster debug builds,
this is much better than no improvement at all.

Another promising avenue of attack is to parallelize as much of the
IR processing as possible. 
For example, many optimization passes and code generation steps
operate at the level of individual functions, allowing each function's work to
be independently scheduled across different threads.

## Next Steps ##

It is impossible to reach any firm conclusions about Cone's or LLVM's 
performance architecture based purely on the the data and analysis shared in this post.
The only way to reach more definitive conclusions involves careful benchmarks
that compare performance of different design approaches that accomplish the
exact same task.

From my perspective, some information is still better than none.
On the basis of this analysis so far,
I am taking away several useful lessons to guide my future choices:

* Designing for performance upfront can work out really well, contrary to the warnings
  from Knuth-inspired skeptics.
  Good decisions for performance can be made that improve programmer productivity
  and result in understandable code.
  I want to keep getting better at this.
  I also want to design the Cone language to make this easier for its programmers.
* I really don't need to focus on performance tuning the Cone compiler's front-end.
  I am really quite pleased with its current speed for now.
  After adding macro and generics support, it would probably be valuable to revisit these numbers.
* LLVM is going to be a boat anchor to rapid Cone build times.
  Near term, I might explore ways to speed up my use of LLVM,
  along the lines expressed above (pre-optimizing LLVM IR generation and type normalization).
* Longer term, I should probably explore alternatives to LLVM, particularly
  for doing debug builds where the generation of optimized runtimes is not a priority.
  Since I am not eager to tackle the work of building a custom back-end,
  I will certainly take a closer look at ways to exploit other back-ends,
  as demonstrated by compilers
  that boast much faster code generation (e.g., tcc or Jai).

As always, I am interested to learn from other people's experiences and insights
regarding compiler and LLVM/backend performance,
whether on your own projects or based on results you have seen published by other
teams that must surely be wrestling with similar challenges.

-------------------

<sup>1</sup>Thanks to u/sanxlyn for pointing out FastISel,
a fast mode for LLVM code generation, an option I had forgotten about.
When I dialed codegen optimization from "aggressive" to "none",
it cut code gen times in half.

<sup>2</sup>It turns out, fortunately, others are way ahead of me on this insight:
As pcwalton explains, the Rust-based "Cranelift is in some ways an effort
to rearchitect LLVM for faster compilation speed, written by some longtime LLVM contributors. 
For example, Cranelift has only one IR, which lets it avoid rebuilding trees over and over, 
and the IR is stored tightly packed instead of scattered throughout memory."
Furthermore, LLVM 9 evidently plans to replace SelectionDag and FastISel
with a [largely rewritten GlobalISel module](https://llvm.org/docs/GlobalISel.html).
Performance is a key goal, achieved in part by unifying and simplifying
the IR data structures being used during codegen.