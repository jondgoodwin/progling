---
title: "The Fascinating Influence of Cyclone"
date: 2019-09-10T07:29:57+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory management"
  - "Permissions"
tags:
  - "Rust"
  - "Cone"
  - "Pony"
  - "Midori"
  - "Cyclone"
  - "C++"
---

In 2001, Trevor Jim (AT&T Research) and Greg Morrisett (Cornell) launched
a joint project to develop a safe dialect of the C programming language,
an outgrowth of earlier work on
[Typed Assembly Language](https://www.cs.cornell.edu/talc/).
After five years of hard work and some published
[papers](https://cyclone.thelanguage.org/wiki/Papers/),
the team (including Dan Grossman, Michael Hicks, Nik Swamy, and
[others](https://cyclone.thelanguage.org/wiki/People/))
released [Cyclone 1.0](https://cyclone.thelanguage.org/).
And then the developers moved on to other things.

Few have heard of Cyclone and almost no one has used it.
And yet, when you throw the right rock into a receptive pond, 
waves of influence ripple outwards.
Cyclone was a large, well-fashioned stone; the ripples of its zeitgeist,
as well as the notable innovations it distilled and pioneered, 
continue to spread in fascinating ways.

Before telling this story, here is my warning:
Innovation and influence is a complicated social process.
Nothing new happens in a vacuum.
Thousands of people throw interesting rocks into the pond every year,
influenced by rocks tossed earlier.
No single post can do justice to all these chaotic ripples.

To keep this story coherent and digestible, I focus on certain themes:
progress in the evolution of 
static safety and the application of linearity to memory management and concurrence,
as manifested in a few chosen mainstream ("imperative") programming languages.
This means tons of relevant and important detail about influential academic languages
and research will receive insufficient attention.
Don't let my editorial shortcomings mislead you into a simplistic understanding of cause and effect.
Instead, be curious, explore the historical record,
and appreciate the contributions of so many pioneers.

## Cyclone ##

At the end of the 20th century, systems software was usually built
using C (or pre-"modern" C++).
Because their semantics closely mirror the way CPUs are used,
these languages produce lean, high-performance systems.
The price one pays for this efficiency is the risk of safety bugs,
such as buffer overflows,
making our critical systems vulnerable to malicious actors.

The goal of the Cyclone team was
to build a C-compatible language that was at least as fast and lean,
but much, much safer. Their notion of safety was radical for its time:
unsafe programs should be hard to write, impossible to compile,
and panic when safety violations are encountered at runtime.

Vulnerabilities targeted for extinction included: buffer overflow,
null pointer de-referencing, use-after-free, dangling pointers, double free,
format string attacks, uninitialized variable use, unsafe casts, 
indefinite returns, cross-scope gotos, and undiscriminated unions.

The primary weapon of choice for improving safety (and versatility)
came from strengthening C's notoriously
weak type systems through the skillful application of
prior art from ML, Haskell, and published research, particularly: 

* **Algebraic data types**. Although C supports product types
  with `struct`, use of `union` for sum types can be unsafe. Instead,
  Cyclone introduces fixed-size [tagged unions](https://cyclone.thelanguage.org/wiki/Tagged%20Unions/),
  variable-sized [datatypes](https://cyclone.thelanguage.org/wiki/Datatypes/),
  and [pattern matching](https://cyclone.thelanguage.org/wiki/Pattern%20Matching/).
  Cyclone also supports tuples, enabling functions to return multiple values.

* **Quantified types**. Cyclone supports 
  [parametric polymorphism](https://cyclone.thelanguage.org/wiki/Cyclone%20for%20C%20Programmers/#PolymorphicFunctions)
  (generics) on functions and types. 
  Despite being constrained to word-sized types and unable to monomorphize,
  this feature is sufficient for supporting type-generic boxed collections.
  Cyclone also supports abstract, 
  [existential types](https://cyclone.thelanguage.org/wiki/Existential%20types/), 
  with similar limitations.
  Similar to traits, these allow use of method-like functions across
  differently-implemented concrete types.
  
* **Region-based memory management**. 
  Cyclone was heavily inspired by 
  [Tofte and Talpin](https://en.wikipedia.org/wiki/Region-based_memory_management#Region_inference)'s
  seminal work on regional inference in the mid-1990s. As implemented in 
  [ML Kit](https://www.researchgate.net/publication/220606837_A_Retrospective_on_Region-Based_Memory_Management)
  (with Birkedal and others),
  whole program inference made it possible to replace the use of 
  garbage-collected (GC) memory with faster, scope-nested memory regions (arenas).
  Related work by Aiken applied arenas and ref-counted memory management to C.
  The Cyclone team improved on these techniques,
  replacing cross-functional inference with explicit 
  [region](https://cyclone.thelanguage.org/wiki/Memory%20Management%20Via%20Regions/) annotations.
  Importantly, they enriched this scheme to
  support an unheard-of variety of safe, automatic memory management strategies:
  arena regions (including first-class arenas), reference-counted,
  tracing GC (via Boehm), and something they call the
  *unique* region.
  
* **Linear/affine types**. Cyclone's 
  [*unique* region](https://cyclone.thelanguage.org/wiki/Pointers%20with%20Restricted%20Aliasing/#UniquePointers)
  is a useful application of the
  [linear logic](https://en.wikipedia.org/wiki/Linear_logic)
  work from the late 1980s by Girard, refined later by Wadler and then Walker.
  By guaranteeing that allocated memory will only ever have
  one owner (reference), we get safe, deterministic memory management
  without run-time GC bookkeeping costs. Reference-counted memory management
  is also more efficient when built using linear logic.
  That said, linear logic (and regions) adds complexity to a language in terms
  of move semantics and quantified types, challenges the Cyclone team
  had to work through.

[Pointers](https://cyclone.thelanguage.org/wiki/Cyclone%20for%20C%20Programmers/#Pointers)
required the most work to make them safe:

* **Null pointers**.
  Cyclone addressed Tony Hoare's so-called "billion dollar mistake" by offering a choice.
  You may define pointers as non-nullable (e.g., `int @x`) and use them freely and safely.
  Or, if you do need a nullable pointer, the compiler will not let you de-reference it
  until you demonstrate first that it is not null.
  
* **Fat and bounded pointers**.
  To protect against buffer overflows, Cyclone offers "fat" pointers (`char ?`),
  which bake in accurate bounds data next to the pointer.
  Pointer arithmetic is allowed, but any attempt to
  access elements outside the bounds triggers a runtime error.
  Bounded pointers offer similar capability in a somewhat different way.

* **Memory-safe pointers**.
  To ensure pointers can only access valid, live data,
  a pointer type can be annotated with the region its object acquired memory from.
  This annotation ensures that the object
  is freed only when the last usable pointer to that object expires.
  Using data flow analysis, pointers to stack allocated data are also kept safe.

* **Polymorphic pointers**.
  With so many type annotations on pointers, type safety could
  force us to duplicate code (or use generics) for every possible
  pointer variation being passed to functions.
  Cyclone overcame this usability challenge by supporting 
  [polymorphic pointer types](https://cyclone.thelanguage.org/wiki/Pointer%20Subtyping/),
  which accommodate a wide variety of pointer types safely.

More could be said about Cyclone's safety extensions
(e.g., [exception handling](https://cyclone.thelanguage.org/wiki/Cyclone%20for%20C%20Programmers/#Exceptions) and
[definite assignment](https://cyclone.thelanguage.org/wiki/Definite%20Assignment/)),
but I think you get the point.
The Cyclone team was diligent and thorough about hunting down
and mitigating safety vulnerabilities.
They even worked through a static type strategy for
making [multi-threading safe](https://homes.cs.washington.edu/~djg/papers/cycthreads.pdf),
using thread locks and thread-local data.

Remarkably, all these safety and versatility improvements were designed to preserve
as much backward compatibility with C as possible.
For understandable reasons, they wanted to make the path to safety
as painless as possible for existing C programmers.
Their papers show helpful examples where C code was ported to Cyclone,
showing benchmark results that illustrate that safety need not
incur a steep performance penalty.

## C++, Ownership and Aliasing ##

Before going forward in time, I want to contrast Cyclone's 
safe memory management journey with that of C++.
C++'s memory management rests on two key features available before 1990:
templates (more versatile and complex than typical generics) and
Resource Acquisition Is Initialization (RAII).
Using RAII, a program can acquire and use some resource local to a block,
and expect the compiler to automatically
destroy that resource at the end of that block.
However, RAII does not work with objects dynamically allocated using `new`.

To address the potential for leaks due to forgotten `delete`s,
the 1997 standard introduced `auto_ptr`, the first "smart" pointer.
This template-defined feature acts like a pointer, while still
empowering RAII to ensure automatic deletion of the owned object.
Even better, `auto_ptr` was linear-like<sup>1</sup>: 
Only one variable owned the pointed-at resource.
Assignment would transfer ownership.

However, `auto_ptr` had a fatal design flaw, limiting its usefulness.
In 2002, inspired by Andrew Koenig from Bell Labs, Howard Hinnant authored
["A proposal to add move semantics to C++"](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2002/n1377.htm#move_ptr%20Example).
A key motivation was performance:
copying a pointer non-destructively is far cheaper than any sort of deep copy.
The standards process resulted with `unique_ptr` replacing `auto_ptr`
in 2005's technical report (TR1) and ultimately C++11.
The same standard also introduced `shared_ptr`, a ref-counted smart pointer,
based on practices dating back to the late 1990s.

Thus, by 2006, divergent influences had resulted in both Cyclone and C++
supporting the same two forms of automatic memory management: 
single owner (linear) and reference counted.
That said, Cyclone's region-based memory management was more
versatile (as it also supported arenas and tracing GC) and far safer.

When it came to rigorous memory safety, the Cyclone team had a deeper well of academic research
and prior practice to leverage.
Besides the influences I mentioned earlier, they drew on
alias types (Smith and Walker, 2000, as implemented in Typed Assembly Language),
Objective C's use of reference counting,
as well as the influential [separation logic](https://cacm.acm.org/magazines/2019/2/234356-separation-logic/fulltext)
(Reynolds, O'Hearn, Pym, 2000-2002).
As Greg Morrisett says: "Cyclone’s contribution was to find a common framework to put a bunch of different things in."

The handling of "borrowed" references illustrates the safety gap
between Cyclone and C++.
Using `get()` on a C++ smart pointers returns a aliased pointer.
Since C++ does no tracking to ensure pointer aliases are always safe to use,
it is problematically easy to use a pointer after the object it points to has been freed.

Cyclone also has a way to create a polymorphic pointer borrowed from any region-based pointer.
In contrast to C++, Cyclone carefully tracks the scope lifetime of every
borrowed pointer, often implicitly. Sophisticated compile-time analysis of
region lifetimes (including region subtyping)
makes it impossible for any borrowed
pointer to outlive the lifetime of its owning pointer.
Cyclone's borrowed pointers are thus always safe to use.


## Rust ##

In 2006, the same year that the Cyclone team closed up shop,
Graydon Hoare (then a Mozilla employee)
began work on the Rust programming language as a private project.
Three years later, corporate sponsorship, funding and staffing kicked in,
culminating with a 1.0 stable release in 2015.

If you know Rust, my summary of Cyclone's safety focus and features 
must sound eerily familiar.
Rust is a robust, real-world systems programming language that delivers
on almost all of Cyclone's goals. 

Rust cites Cyclone, C++, and SML/OCaml among its many
[influences](https://doc.rust-lang.org/reference/influences.html).
Several core team members, such as Niko
Matsakis, Aaron Turon and Dave Herman, also brought a wealth
of [academic research](https://rust-lang.github.io/rustc-guide/appendix/bibliography.html) 
to bear on their design choices.

Rust is as thoroughly safe as Cyclone, in all the same ways and often
utilizing largely similar techniques. That said,
differences between the languages are easily spotted:

* **Rust's syntax** is C/C++-familiar rather than compatible.

* **Generics and traits** are more flexible, akin to ML-family languages.
  Their use is central to the language's core capabilities
  (e.g., Option, Result, and smart pointer wrappers like Rc and Mutex).

* **Borrowed References** are more versatile, supporting features
  like static mutability exclusion (e.g., `&mut`), non-lexical lifetimes,
  use of borrowing to obtain block-scoped locks, and lifetime-elision sugar.
  That said, it is intriguing to notice that Cyclone's polymorphic
  variables `` `r`` (often used as region annotations) resemble Rust's lifetime annotation `'a`.

* **Memory Management**. Due to Rust's embrace of ownership as a defining feature of the language,
  its support for multiple memory management strategies is more limited than Cyclone's.
  As with C++, single-owner (Box) and ref-counted (Rc) are the dominant techniques.
  Although Rust packages exist for tracing GC and arenas,
  they are limited when compared to Cyclone's support for these regions,
  particularly with regard to safe
  [first-class arenas](https://cyclone.thelanguage.org/wiki/Pointers%20with%20Restricted%20Aliasing/#DynamicRegions).

* **`unsafe`**. The Rust team wisely realized they needed to offer an escape
  hatch that allows programmers to write safe, performant logic which the compiler cannot
  verify as safe. A number of important libraries rely on this essential feature.

Importantly, Rust went all-in on linearity, going well beyond
single-owner memory management.
They wanted to wrestle to the ground issues
that arise from allowing shared, mutable access to values.

With most mainstream languages, programs 
can easily have many references to the same object, 
each able to change (mutate) its state.
This can trigger problems. Debugging and maintenance
are significantly more difficult when you don't know where changes arise,
especially when invariants are unexpectedly broken.
Unconstrained, shared mutability opens the door to data races
not only for multi-threaded programs, but also sometimes in 
[single-threaded contexts](https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/).
Properly used, locks (as Cyclone intended) can cure data races, 
but they often bear a significant throughput cost.

Languages have looked for better mutable, aliasing solutions for decades.
Ada's limited types, C's `restrict`, C++'s strict aliasing rules, 
Fortran's argument non-aliasing restrictions,
[Flexible Alias Protection](http://janvitek.org/pubs/ecoop98.pdf) for Java,
functional language restrictions on mutability,
and academic work on fractional permissions and (again) separation logic.

As it turns out, linear logic is not just
a valuable memory management strategy. It is also a valuable aliasing and data race strategy.
Clean introduced uniqueness types in 1992 for IO handling and destructive updates.
Later languages, like ATS, Alms, and Mezzo, explored this idea in different ways.
The Rust team was in touch with these teams,
and stayed abreast of a wealth of [research](http://www.cs.cmu.edu/~carsten/linearbib/llb.html)
underway at the same time.

Rust's resulting ownership-based model is largely based on the
mutual exclusion idea that one either has a single mutable reference to an object,
or possibly-multiple immutable references to that object.
Abiding by this restriction eliminates the problems cited earlier,
and makes it safe to transfer mutable data locklessly from one thread to another.
Rust's extensive approach to [fearless concurrency](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)
relies on much more than this,
but this aspect is an important contributor.

And, if you really do want shared, mutable, that too is possible,
using lock-based synchronization mechanisms (e.g., `Mutex`) or even the
lockless, but somewhat restricted `Cell`.
  
## Midori's M# ##

[Midori](http://joeduffyblog.com/2015/11/03/blogging-about-midori/)
was a Microsoft research/incubation project that lasted from 2007-2014.
Its purpose was to commercially implement
[Singularity](https://www.microsoft.com/en-us/research/project/singularity/),
an operating system built on more reliable foundations than mountains of unsafe C and C++ code.

To achieve their concurrency, safety and performance goals, the team
created a new dialect of the C# programming language called M#. 
Later-released C# features, such as slices and async/await, had their genesis in M#.
Despite some persistent longing,
they did not follow in Cyclone's (and Rust's) footsteps
with regard to memory management; M# remained 
a garbage-collected language (although extensive changes were made there too).

As with Cyclone and Rust, the work on Midori and M# was richly
inspired by prior art and research: C++ const, alias analysis, linear types, monads,
effect types, regions, separation logic, uniqueness types,
model checking, modern C++, D, Go, and Rust.
Several on the Midori team knew about Cyclone.
Colin Gordon, aware of Dan Grossman's work on Cyclone, followed him to the University of Washington
as a graduate/doctoral student,
where he made important contributions as an intern on the Midori team.
Manuel Fähndrich, whose [earlier work](https://www.microsoft.com/en-us/research/wp-content/uploads/2002/05/pldi02.pdf?from=http%3A%2F%2Fresearch.microsoft.com%2F%7Emaf%2Fpapers%2Fpldi02.pdf)
inspired Cyclone's borrowing, was a key contributor to the Singularity project.
David Tarditi, another Singularity/Midori contributor,
is now working with Michael Hicks (Cyclone team) and others
on Microsoft's [Checked C](https://www.microsoft.com/en-us/research/project/checked-c/?from=http%3A%2F%2Fresearch.microsoft.com%2Fen-us%2Fprojects%2Fcheckedc%2Fdefault.aspx),
whose goals resonate with Cyclone's.

The Midori team was every bit as focused on 
the ["three safeties"](http://joeduffyblog.com/2015/11/03/a-tale-of-three-safeties/)
as Cyclone and Rust.
They used similar safety mechanisms, such as slices (to address bounds-safety),
discriminated unions and borrowed references.
They also battle-tested additional mechanisms, including
[software isolated processes](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/EuroSys2007_SealedProcesses.pdf),
[object capabilities](http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/),
[contracts](http://joeduffyblog.com/2015/12/19/safe-native-code/),
[exception handling](http://joeduffyblog.com/2016/02/07/the-error-model/),
and [reference capabilities](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/msr-tr-2012-79.pdf).

M#'s reference capabilities enrich the usefulness of unique references.
M# marked all references with one of four lockless 
reference capabilities (permission qualifiers): isolated, writable, immutable, and readable.
Even though all are data race safe, each enables (and constrains)
the capabilities of its reference in different ways.
The constraint imposed by `isolated` is linearity: there can be only one.

For all their benefits, unique (linear) references have a significant limitation.
Using only unique references, only hierarchical (tree) data structures can be built.
Nearly all data structures with cycles need to allow multiple
references to the same object, which unique references cannot do.
This limitation, as well as the disruptive constraints of move semantics,
should stay top-of-mind when touting the virtues of
single-owner memory management and unique references.

In light of this limitation, imagine, then, we have an isolated (unique) reference 
to some cyclic data structure,
implemented internally using aliasable references. Is it data race safe to move
this entire data structure from one thread to another?
Definitely yes, if the aliasable references point to immutable values.
However, if the aliased references are mutable (`writable`), we must be more careful.
Moving is then only safe if *all* writable references to every object in that data structure 
are essentially moved over together. If we cannot guarantee the movable data structure is
["externally isolated"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.109.1645&rep=rep1&type=pdf),
we put data race safety at risk,
since multiple threads might end up with mutable references to the same objects.

M# is able to guarantee external isolation at compile-time
though the use of two reference capability mechanisms.
The first is a form of viewpoint adaptation, where the permission of
some field's reference may be downgraded based on the permission of the
reference used to access that field. For example, an immutable reference
may not be used to alter the value pointed at by its field's reference,
even though that field's reference says you can.

The second mechanism is M#'s ability to coerce and then "recover" capabilities.
Recovery enables an isolated reference to be temporarily coerced to writable (or immutable)
for some limited scope, after which we can safely recover its original isolated status.
While it is writable, we can mutate and enrich the pointed-at data structure
with additional objects.
To preserve external isolation, logic within the recovery scope is constrained 
so that it may only use `immutable` or `isolated` references from
outside the scope (and not writable or readonly).

In addition to enabling cross-thread moves of mutable, cyclic data structures,
M#'s coercion and recovery mechanism offers other benefits.
It can also be used to safely (and mutably) build cyclic data structures
that end up transitioning to immutable.
The end result is that Midori's reference capabilities
give multi-threaded programs the ability to safely extract additional performance and 
data structure versatility benefits
through the use of unique references.

## Pony ##

In 2014-5, Sylvan Clebsch and his team designed and built a compiler for the actor-based
[Pony language](https://www.ponylang.io/).
Here too many related influences were brought to bear,
including Erlang's prior work on actor processes.
Once again, the design choices demonstrate that performance
and [capabilities-secure safety](https://tutorial.ponylang.io/) need not be enemies.

Pony is full of fascinating features and architectural decisions, such as:
actors and behaviors, implicit message passing, interfaces vs. traits, 
and an unusual hybrid garbage collection approach that uses
both reference counting and tracing.
Like M#, Pony supports object and reference capabilities.
Pony refined, enriched, and renamed its set of reference capabilities,
further clarifying what is allowed and denied.
Pony also made it possible to instantiate generics with a unique reference.

## Cone ##

Although I had no role in the stories I have told so far,
they are of great personal interest to me, due to my work
on the [Cone](http://cone.jondgoodwin.com/) language.

Two years ago, I studied 
Rust and Pony carefully in hopes of learning innovative strategies for
managing memory and concurrency safely and efficiently.
I was excited by what I discovered, and ended up documenting
my thoughts on how to extend their techniques
for [gradual memory management](http://jondgoodwin.com/pling/gmm.pdf).
In particular, I want to marry Rust's approach to owned and borrowed
references with Pony's reference capabilities (permissions),
in support of a more versatile, safe, performant use of memory.

At that time I had no idea how much relevant pioneering work Cyclone had done
before Rust and Pony.
That changed a few months back, when Aaron Weiss sent me links to a pair
of interesting academic articles. One thing led to another, and before long
I found myself immersed in the history of Cyclone and related work in great detail.

I now understand that Cone is pursuing similar goals to Cyclone,
not just in terms of safety, concurrence and performance,
but also [memory management versatility](http://cone.jondgoodwin.com/memory.html).
It has long been one of my goals to make it easy for a programmer
to choose whether to allocate and manage memory using single-owner, ref-counted, tracing GC, arenas,
or pools, as best meets a program's performance and data structure requirements.
Now, inspired by Cyclone having accomplished this 15 years earlier, 
I am looking into adding first-class arenas to that list.
I am no less grateful to be able to
leverage many of the innovative, improved features pioneered in later languages.

## Summary ##

At the end of Philip Wadler's 1990 paper 
[*Linear types can change the world!*](https://pdfs.semanticscholar.org/24c8/50390fba27fc6f3241cb34ce7bc6f3765627.pdf),
he concluded: "more practical experience is required before we can evaluate the likelihood that
linear types *will* change the world."
So many language have given us that practical experience.
And *that* experience is undoubtedly changing the way the programming world
designs programs for safety and performance.

Now that languages focusing on safety, performance, and linearity are becoming
more accepted and successful, we are increasingly
seeing existing (and new) languages adopting similar features.
The D programming language is planning to support
[ownership and borrowing](https://dlang.org/blog/2019/07/15/ownership-and-borrowing-in-d/).
Nim is exploring [something similar](https://github.com/nim-lang/RFCs/issues/144),
citing a Google/IBM paper that bases much of its technique on Cyclone.

This will not be the end of it, I am sure.
And for that, I give thanks to so many people,
including the Cyclone team that did so much to help get this ball rolling.

------------------------

I am grateful to Greg Morrisett,
Graydon Hoare, Michael Hicks, Colin Gordon and Dan Grossman
for their invaluable feedback on an earlier version of this post,
opening my eyes to how many people have contributed seminal ideas that
have gone into making these languages possible.

<sup>1</sup>C++ was not the first.
Five years earlier (1992), [Linear Lisp](http://home.pipeline.com/~hbaker1/LinearLisp.html)
demonstrated use of linear logic as a solution to garbage collection.
