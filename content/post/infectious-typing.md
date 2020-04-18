---
title: "Infectious Typing"
date: 2019-11-03T07:21:16+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Types"
tags:
  - "Cone"
---

Type theory has many kinds of typing mechanisms: product, sum, top, bottom, recursive,
universal, existential, subtypes, linear, refinement, dependent, and so on.
I have run across something different from all those, 
which I am calling "infectious typing".
If anyone knows a formal name and/or treatment for this mechanism, 
I would love to hear about it.

Type composition allows us to create new types out of existing ones.
A struct or class product types is composed of several fields, each with its own type.
Likewise, a sum type (discriminated union) establishes that its value
can be one of several types.

Infectious typing is the observation that any composed type as a whole 
is constrained infectiously by constraints held by its constituent types.
Often, but not necessarily, these result from constraints 
that safety considerations impose on reference types.

Let's illustrate this abstract idea with some interesting examples.

## Move Semantics ##

Sometimes (often for safety), we want to forbid the copying of certain values.
When we know that only one copy exists of a value, we can safely close that resource
or free the object's memory, avoiding the risk of use-after-free.
Similarly, unique references allow us to move a mutable data structure
efficiently and safely from one thread to another without risk of data races.

Even though we want to forbid the existence of copies, 
we still want to freely move such unique (or owned) values around the program.
Languages like C++ and Rust accomplish this through the use of move semantics.
In Rust's case, each type determines whether its values are handled using copy vs. move semantics.

In the presence of type composition, move semantics are infectious:
Generally speaking, if the type of a struct's field is subject to move semantics,
the composed type should also be subject to move semantics.
Intuitively, this makes sense. If we are not allowed to make a copy of that
field's value, we should also not be allowed to make a copy of any record
containing that field. Similar logic would apply to sum type values.

Intriguingly, a type's methods can also influence whether a type should
enforce move vs. copy semantics. If it implements a finalizer, 
move semantics may be necessary to avoid use-after-free.
However, if the type specifically implements a method for safely
creating a copy of any value (despite the presence of unique fields or a finalizer),
then copy semantics may make more sense.

## Lifetimes ##

In Rust, borrowed references deliver invaluable versatility when working
with single-owned objects.
To ensure they are always safe to use, borrowed references are lifetime-constrained.
The compiler's borrow checker ensures, at compile-time, that borrowed references
can never escape to a greater scope than the block scope they were created in.

Lifetime constraints are also infectious across type composition.
If a struct record contains a borrowed reference, then that aggregate record
must also comply to the lifetime constraint of its borrowed reference.
Again, this makes sense intuitively. If we do not constrain the lifetime of the record,
it might be possible for the borrowed reference within that record
to live longer than it should.

Lifetime infectiousness has its own interesting wrinkles which surface when a record
holds multiple fields carrying different lifetime constraints.
In some cases, this can be resolved by constraining the lifetime of the record
to the shortest, shared lifetime of all its fields.
However, if the lifetimes of these fields are disjoint, 
as happens with invariant lifetimes, no such resolution is possible.
In these cases, a compiler error must be raised whenever an attempt is made
to instantiate such a record.

## Thread bound ##

For many types it is [quite safe](post/interior-references-and-shared-mutability)
to allow multiple shared references to have mutable access to the same object,
so long as all those references are constrained to the same thread.
This compile-time constraint protects against data races without the need for
runtime locks, thereby improving both performance and safety.

Once again, this thread-bound constraint is infectious.
If a record holds a thread-bound value, then we cannot safely send the record
(or any reference to that record) to another thread.

## Impure functions ##

For my final example, let's switch gears and talk about function types.
In many languages, it is useful to distinguish between pure functions,
which may not access global state and whose side-effects are constrained to
the return value(s) or argument-passed mutable references,
and impure functions that have no such constraints.
Pure functions can make it easier to reason about code logic,
and offer guarantees useful to compile-time analysis (e.g., compile-time function execution).

Functions are composed of its expressions and statements.
As it turns out, impurity (or Haskell's monadic IO) is infectious.
A function cannot be pure if it invokes even a single impure function.

Here too, the infectiousness of impurity can offer interesting complexities,
as there can be several valuable variations on function purity.
In some cases, a purity scale can be formulated,
where we can assess how much contamination a certain form of some function's impurity injects
into another function that calls it.

It is also interesting to notice that infectiousness here works in the opposite
direction from the earlier examples.
In the earlier examples, the constraints of the parts were visited on the whole.
With functions, it was the *lack* of constraints on the called functions
that infected the composed function.

## Additional Challenges ##

All of the above examples are relevant to Cone's evolving type system,
which needs to calculate and enforce these kinds of constraints.
It is quite possible that other such examples of infectious typing will emerge
as time goes forward.

Infectious typing becomes even more complicated to manage correctly
in the presence of recursive types and generics.

### Recursive, Infectious Types ###

Types are recursive when you can draw out cyclic dependencies between one or more types.
For example, a List node might point to another List node.
Or some record of type A might have a field that points to an object of type B, whose field
may in turn point back to an object of type A.

Recursive types can be problematic for language compilers, 
as their propensity towards infinite regress can lead to problems with
semantic analysis, storage space, or instantiation challenges.
Many languages impose restrictions on how recursive types may be used
to avoid these problems.

Applying infectious typing to recursive types adds an additional difficulty.
Can one always resolve infectious typing in the face of recursive type cycles?
I suspect that the answer is yes, but I am still working out the best deterministic
way to accurately calculate these effects.

### Generics and infectious constraints ###

It can be very convenient for a language to allow the specification of
these kind of infectious constraints as bounds on which types a generic can instantiate.
If a type needs to comply with move (or copy) semantics, it would be good to be able
to specify it. This applies as well to lifetime and threadbound constraints.
The ability to do this improves code readability and makes compiler errors
easier to diagnose.

More troubling is the potential for constraints on generic type parameters
to interfere with each other in interesting non-compositional ways.
I have already seen the potential for these interference effects between the three
type parameters used to define a Cone reference:  region, permission, and value type.
For example, a reference is move semantics if either the region or permission
is linear, but whether or not the value type is move semantics has no say over
whether the reference type is move semantics.
As another example, if the value type makes use of dependent pair or functions,
this might constrain what can be done with the reference
depending on its permission.

Working out these challenges is something I will have to tackle incrementally,
via practical examples and careful analysis once the language is robust
enough to experiment with these possibilities.