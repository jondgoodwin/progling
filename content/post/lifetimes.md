---
title: "The Power of Lifetimes"
date: 2019-04-16T07:20:05+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "References"
  - "Memory management"
tags:
  - "Rust"
  - "Cone"
  - "Cyclone"
---

The execution of a program unfolds over some interval of time.
The lifetime of every temporary resource (e.g., variable or object)
is the time span between that resource's "creation" and "destruction".
This lifetime is wholly contained within the typically-longer lifetime of the program.

The goal of this post is to explore how versatile lifetime analysis
has increasingly become in managing memory efficiently, safely and with better performance.
By the end of this post, we will explore exciting new ways to apply lifetime analysis,
beyond their current support in Rust.
However, let's begin by demonstrating the explicit relationship between
lifetimes and automatic memory management.

## Lifetimes and Garbage Collection ##

Automatic memory management is all about lifetime analysis.
Here's why: the useful lifetime of an object depends entirely on
the lifetimes of all references to that object.
The object's life begins with creating the first reference to it.
It needs to stay alive as long as any strong<sup>1</sup> reference points to it.
The allocated object becomes a useless waste of space
once the last strong reference to it expires, making it ripe for memory reclamation.

The purpose of all forms of automatic memory management is to track
the lifetimes of all strong references to that object.
As soon as we know when the lifetimes of these references have all expired,
we can automatically destroy the object and reclaim its memory for re-use.
What distinguishes the various automatic memory management strategies
is **how** each determines when all strong references to an object have expired.

- **Tracing garbage collection** relies on periodically probing all references.
  It traces all strong references emanating from the root set, essentially
  marking all objects whose reference lifetimes have not yet expired.
  It can then iterate across a list of all allocated objects to reclaim
  the memory used by all unmarked objects.
  
- **Reference counting** keeps a running count of how
  many strong (and weak) references there are to an object.
  In effect, the reference count is incremented whenever a new reference to the object
  is created and decremented when the lifetime of that reference expires.
  The object's memory can be reclaimed when the count goes to 0.
  
- **Single-owner** uses compile-time analysis to ensure the first
  reference created to an allocated object is also the last to survive.
  This works because aliasing is done
  using borrowed references whose lifetimes are guaranteed to be
  shorter than the original reference they borrowed from.
  Thus, when the original reference expires, the object's memory is safely reclaimed.
  
- **Arena** ensures that all references to objects allocated within an arena
  have a lifetime shorter than the arena itself. Thus, when the
  arena expires, it is safe to reclaim all memory held by the arena,
  since we know no references to any of its objects exist.
  
Notice how all these techniques, except tracing garbage collection,
involve the compiler doing lifetime analysis: either knowing when a lifetime
expires or comparing lifetimes. This compile-time analysis relies on the fact
that most lifetimes are structured by scopes.

## Structured lifetimes ##

In 1968, Edsger Dijkstra's provocative letter,
["Go To Statement Considered Harmful"](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf),
became an influential catalyst towards the ascent of
[structured programming](https://en.wikipedia.org/wiki/Structured_programming).
Structured programming replaces "spaghetti code" with modularized
control flow structures (e.g., "if" and "while blocks) that
have a single point-of-entry and a single point-of-exit.
This allows control flow to be modeled using a
[reducible flow graph](https://en.wikipedia.org/wiki/Interval_(graph_theory)).

One of the beneficial side-effects of our now-dominant, block-based
style of coding, is that we can conveniently scope the lifetimes of
our local variables to the inner-most block they are declared within.
These variables are "allocated" near the beginning of that block
and are automatically "freed" at block's end.
We can say that local variables have a structured lifetime,
one that correlates with a single-entry/single-exit control flow block.

Compile-time lifetime analysis for memory management is made easier when lifetimes are structured,
especially since analysis only cares when a lifetime expires.
We know that a local variable's lifetime expires no later than the end of its scope.
Similarly, we know that a local variable in an outer block
has a longer lifetime than any local variable scoped within an inner block,
as the inner block always completes before the outer block does.

Structured lifetime analysis can be reduced, in many cases, to simple arithmetic
that can be performed at compile-time.
Assign all blocks an integer number in a LIFO manner:
Global scope is 0, a function's parameters are 1, the outermost block's variables are 2, etc.
You increment the lifetime number as you enter a block and decrement when you leave.
To compare the lifetimes of two variables, compare their number;
a greater number corresponds to a shorter lifetime.

If you look back at the description of the four memory management strategies,
you will see how most of them depend only on knowing when lifetimes expire
and the ability to compare lifetimes. The more aggressively we can do this analysis lexically
at compile-time, the more we can excise performance-sapping runtime bookkeeping costs
from our programs. Even reference-counting can be optimized significantly
by using static lifetime analysis, allowing us to avoid the cost of altering
the reference counter when we know a reference has escaped from one scope to another.

## Borrowed References ##

Rust is rightly the programming language that comes to mind first when thinking about lifetimes.
However, it is worth noticing that Rust's lifetime mechanisms improve on prior art.
For example, Rust's static, single-owner memory management is an enrichment to
C++'s [Resource Acquisition is Initialization (RAII)](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization).
RAII with move semantics ensures an allocated resource is automatically destroyed at
end of its last scope. 

Rust's lifetime-constrained borrowed references improve on this by allowing safe aliasing.
The inspiration for safe borrowed references comes in two ways from
the [Cyclone programming language](https://en.wikipedia.org/wiki/Cyclone_(programming_language)),
a joint development effort by AT&T and Cornell University.
The notion of a borrowed reference is adapted from Cyclone's
polymorphic [restricted alias pointers](https://cyclone.thelanguage.org/wiki/Pointers%20with%20Restricted%20Aliasing/).
Additionally, Cyclone's safe memory management focused on regions (which are usually arenas).
In order to ensure memory safety, Cyclone pointers could be annotated
with a named region qualifier preceded with a back-tick
(making them look very much like Rust's apostrophe-based lifetime annotations).

Niko Matsakis's key insight in 2012 was to
[annotate borrowed references](http://smallcultfollowing.com/babysteps/blog/2012/07/17/borrowed-pointer-tutorial/) 
with lifetimes rather than a memory-allocation region (arena).
This makes possible two key benefits:

* Borrowed references are isolated from having to know anything
  about how the object's memory was allocated (including the stack) to be useful to program logic. 
  A borrowed reference can be borrowed from any kind of allocated pointer, or even from
  the interior of an allocated object. This makes borrowed references
  a perfect fit for polymorphic use of references.
  
* Safety of the borrowed reference can be easily guaranteed at compile-time
  by ensuring that the lifetime of any object pointed at by the borrowed reference's lifetime
  lasts longer than the lifetime of the data structure that holds that reference.

Every use of lifetime-constrained borrowed references doesn't just make
reference-based polymorphism possible.
It also reduces the runtime bookkeeping costs when handling using RC-allocated objects,
since borrowed references never increment or decrement the reference counter.

Checking the lifetimes of borrowed references is the job performed by Rust's borrow checker.
People sometimes get confused and think the borrow checker is the name for Rust's
single-owner memory management strategy. In truth, they are separate mechanisms that
synergistically work well together. The sole job of the borrow checker is
to ensure that references borrowed from any stack-, Box-, or Rc- allocated object
are always safe to dereference. It is mostly a lifetime checker.

## Lifetime Inference and Annotations ##

To the compiler, all borrowed references have lifetimes.
However, they are often not visibly annotated on most references.
This is because a lot of attention was paid to making Rust lifetimes as usable as possible,
largely by allowing lifetime annotations (qualifiers) to be elided from references
wherever they can be accurately inferred. 
Lifetime inference is easy-to-accomplish in most cases.

Rust requires lifetime annotations in two places: on certain
function signatures and on struct fields bearing borrowed references.
For the most part, lifetime annotations are only needed 
to simplify lifetime analysis for these potentially ambiguous situations:

* When a function returns a borrowed reference whose lifetime is unclear,
  given that the function accepts multiple borrowed references 
  (each with possibly different lifetimes) as parameters.
* When a function will store one a borrowed reference originating from
  one parameter into a data structure originating from another parameter's borrowed reference.
* When a function returns a borrowed reference to a longer-lived field
  from a shorter-lived data structure pointed at by a parameter's borrowed reference.

## Non-lexical lifetimes ##

Rust 2018 Edition improves on borrowed reference lifetimes
by adding support for "non-lexical lifetimes".
A more descriptive name might be "sub-lexical lifetimes" or "sub-block lifetimes",
as the improvement allows the lifetime of a borrowed reference
to end sooner than the end of the lexical block scope it is declared within.
As usual, Niko does a great job of 
[summarizing key situations](http://smallcultfollowing.com/babysteps/blog/2016/04/27/non-lexical-lifetimes-introduction/)
where support for non-lexical lifetimes can be useful.

Before Rust 2018, the borrow checker would put a usage lock on whatever it
borrowed from, for the full lifetime of the enclosing block.
This lock prevents unsafe use of the original resource while the borrowed reference is in play.
The lock would expire at the end of the block.
With non-lexical lifetimes, Rust can detect that use of the borrowed reference
finishes before the end of the block, enabling it to expire the usage lock earlier
and allow subsequent, safe use of the original resource within that block.

Part of the benefit lies in making it more convenient to handle
these situations without requiring awkward workarounds.
More importantly, support for non-lexical lifetimes makes the borrow
checker smarter and less likely to yell at the programmer,
as it can now notice that the code's use of
borrowed references is essentially safe.

## Extending the Power of Lifetimes ##

At the beginning of this post, I promised to sketch out several 
exciting ways we can extend the versatility and usability of lifetimes
yet further. Let's get to that!

### Make lifetime annotations easier ###

Rust describes [three rules](https://doc.rust-lang.org/1.9.0/book/lifetimes.html)
for determining when lifetime annotations can be elided from function signatures.
These begin with the assumption that the lifetime for each borrowed reference on
a function signature is different. The subsequent two rules reverse this assumption
under certain conditions.

One possible improvement might be to make the opposite assumption: that these
lifetimes are all the same. Simplifying the annotation elision rules in this way
would make it easier for a programmer to decide when a lifetime annotation must be specified.
It might also reduce how many times lifetime annotations are even required.

### Enable thread support for borrowed references ###

Rust does not allow borrowed references to be passed from one unbound thread to another.<sup>2</sup>
This restriction is due to memory (and race) safety.
Put simply, it is not possible to compare lifetimes between one unbound thread and another.
Without that ability, the borrow checker cannot do its job and guarantee that the
lifetime constraints of all borrowed references will be honored.

Remember how Dijsktra's diatribe against GOTO paved
the way towards structured control flow blocks and then structured lifetimes?
A year ago, Nathaniel Smith articulated a similar attack on the dominant use of unbound concurrency
(e.g., Go threads).
He proposed a switch to
[structured concurrency](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)
instead.

The basic idea is that threads are spawned within a concurrency nursery,
a different kind of structured control flow block.
At the end of the block, all spawned threads are joined back together.
This proposal is less restrictive than it may sound, which his post explores.
The post also enumerates the benefits, including having better control
over the ability to cancel an in-process thread.

So what does all this have to do with lifetimes and borrowed references?
Well, in unbound concurrency, we have no idea when a thread is going to expire.
For all we know, the lifetime of any spawned thread could be as long as the program itself.
Aha! I can hear you exclaim:  That's the simple explanation for why the
borrow checker refuses to move a lifetime-constrained borrowed reference 
to a (potentially) longer-lived thread.
This would violate the lifetime constraint placed on that reference!

However, a language's support for structured concurrency and nursery blocks
makes it safe and viable to support passing borrowed references to scoped threads,
as demonstrated by Rust's Rayon and Crossbeam packages.
This is safe because the lifetime of these scoped threads is guaranteed
to be smaller than any borrowed reference passed to it.
Allowing threads (and channels) to have lexical lifetimes opens up so many interesting doors.

### Support lifetimes on allocated references ###

Remember Cyclone and its safe support for regions (arenas)?
Regions are first-class in Cyclone, which means you can not only
define them in some lexical scope, you can dynamically create them and pass them around.
This creates some interesting safety challenges.

With allocation techniques like ref-counting or tracing GC,
one typically assumes that the allocation area is global, with a lifetime
as long as the running program itself.
But with first-class regions, the allocation area's lifetime is bounded.

This post is already too long for me to offer a detailed explanation 
of how a language like Cyclone might
guarantee the safety of first-class arenas.
But the heart of it is that the arena itself has a lifetime, which
the compiler can ensure is longer than every object allocated
within it and every reference to those objects.
Inferred, and sometimes annotated, lifetimes are needed on all these references,
even when they are not borrowed references.
The same sort of rules would hold true for tracing GC references, should one
want to support multiple dynamically-allocated GC pools, much as Pony
does for each spawned actor.

### Support lifetimes as movable regions ###

There is one more wrinkle on lifetimes for allocated references that
is worth mentioning. A reference with the 'uni' permission ensures that
only one reference exists to its object. A key benefit of a 'uni'
reference is how it allows a mutable object to be safely moved from
one thread to another. However, this can raise race safety concerns.

If the uni reference points to a data structure that includes shared, mutable
references to certain objects and *all* of them move over together, that's fine.
But if it ends up splitting up the references so that
two threads have unlocked, mutable access to the same object, that's not okay.
The common solution to this problem, viewpoint adaptation, basically
prohibits shared, mutable references from the moved data graph.
The downside is that the moved data cannot hold a mutable cyclic graph.

The use of a special kind of lifetime annotation, where the lifetime has to match exactly<sup>3</sup>,
can be used to get around this limitation. With this improvement,
we are able to guarantee that all shared-mutable references to any object
are all found in the same lifetime (effectively like an abstract region),
and therefore will all move over together from one thread to the next safely.
To get safe, movable, mutable cycles, the sacrifice becomes an added, explicit lifetime annotation.

## Summary ##

The notion of lifetimes has been around for a long time, if not by that name
(usually we use the term 'scope'). It turns out that formalizing
the concept into static type qualifiers, and building additional analysis around it 
as Rust does, we can offer a bunch of rich, compile-time memory and resource management
mechanisms that can improve performance, 
while also ensuring memory and race safety.
With good inference, we can minimize its human factors impact on the programmer.

Rust offers an invaluable waypoint on that journey.
Cyclone shows that the future of lifetimes still beckons.
I intend to incorporate these additional improvements as part of Cone's design.

------------

<sup>1</sup> I want to be accurate here without getting into the weeds.
With garbage collection, not all references are treated equally.
For example, strong references (the default) keep their object alive,
but references explicitly designated as [weak](https://en.wikipedia.org/wiki/Weak_reference) do not.
Tracing GCs may need to have their own private (untraced) reference
to all objects in order for the sweep phase to work correctly.
Lastly, references can certainly exist in a program that are not traceable
from the [root set](https://en.wikipedia.org/wiki/Tracing_garbage_collection)
(typically the execution stack and global variables); these too should not keep an object alive.
For the purpose of this post, we only care about the lifetimes
of a program's in-play references whose existence requires that
the object remain in memory and accessible. (Thanks /u/raiph)

<sup>2</sup> In the original version of this post, I was unaware that Rust already supports
structured concurrency with Rayon and Crossbeam. 
I have revised this section accordingly. 
Rust threads are normally marked as 'static, but Crossbeam is able to spawn nested, 
[scoped threads](https://docs.rs/crossbeam/0.7.1/crossbeam/thread/index.html). 
"Unsafe" magic then
[resets a passed closure's lifetime to 'static](https://docs.rs/crossbeam-utils/0.6.5/src/crossbeam_utils/thread.rs.html#427).
Although a cleaner architectural approach might be preferred, this definitely gets the job done.
(thanks u/sociopath_in_me, u/burntsushi and u/peterjoel)

<sup>3</sup> Evidently Rust supports this capability using invariant lifetimes.
Alexis Beingessner (Gankro) describes how to define and use these
in section 6.3 of ["You can't spell trust without Rust"](https://raw.githubusercontent.com/Gankro/thesis/master/thesis.pdf).
Catherine West's [Lester library](https://github.com/kyren/luster) (a reimplementation of Lua) 
uses them to provide a safe interface to a garbage collected heap.
(thanks u/rusky)