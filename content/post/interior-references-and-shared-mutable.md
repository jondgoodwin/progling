---
title: "Interior References and Shared Mutability"
date: 2018-12-17T08:24:17+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Permissions"
tags:
  - "Cone"
  - "Rust"
---

In my last post, [Race-safe Strategies](post/race-safe-strategies/),
one footnote stated
"safety issues which look suspiciously similar to race conditions
can crop up when a language supports the creation of “interior references” 
to shared, mutable values of certain types".
Let's explore that now.

I will begin by recapitulating Manish Goregaokar's excellent post
"[The Problem With Single-Threaded Shared Mutable](https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/)".
His post clearly explains why the Rust language wishes to steer developers
towards [RefCell](https://doc.rust-lang.org/std/cell/index.html) for shared references
over use of Cell, its inflexible shared, mutable counterpart.
Although I agree with his problem diagnosis,
I want to offer a more expansive take on other ways to deal with shared mutability,
following in the spirit of [this feedback](https://www.reddit.com/r/rust/comments/369jnx/the_problem_with_singlethreaded_shared_mutability/crcqisq).

## The Safety Challenge ##

Let's say we have multiple references all pointing to the same object.
All of them have the `mut` (shared, mutable) permission.
This means any of them can be used to mutate the object.
They are all constrained to the same thread to avoid a race-safety violation.

How can their use lead to memory- or type-safety problems?
There are two representative situations where such safety issues can arise.

**Resizable Array Example:**
Imagine we have a resizeable array containing 100 32-bit integers.
We have a variable called `arrayref` holding a mutable reference to this array.
Sequentially, some piece of code performs these steps:

1. Creates a reference (named `fifty`) to the 50th integer in the array using `arrayref`.
2. Resizes the array to hold 1,000 integers.
   To find room, the array moves to new memory locations.
3. Dereferences `fifty` to retrieve the integer it points to.

The last step is problematic, since it still points to memory that no longer holds
array integers. Maybe it will seg-fault, but more likely it will fail silently.
(Note: a very similar problem can happen when dealing with single-owner allocated
memory encapsulated by an owning struct type.
Any reference to it would be invalidated if the memory block is replaced with another.)<sup>1</sup>

Even if resizing doesn't move the array, we could still have a problem
if the resizing shrinks the array smaller than the element we are pointing to.
A common flavor of this problematic behavior is called
[iterator invalidation](http://wiki.c2.com/?IteratorInvalidationProblem).

**Variant Type Example:**
Let's say we have a variant type `SomeInt` capable of holding either an integer or
a reference to an integer.
We have a SomeInt-typed value called `someInt` holding a reference to the integer 5.
Our code unfolds similarly to the above:

1. After pattern matching that `someInt` holds a ref to an integer,
   we obtain a reference named `someref` to that reference.
2. We mutate `someInt` to now hold the integer 10.
3. We dereference `someref` twice

This will generate a seg-fault since we are effectively dereferencing an integer
as if it would be a pointer. In other similar situations,
it is more likely to fail silently by treating a value of one type as if it had a different
type (and value).

## Mutual Exclusion to the Rescue ##

As Manish's post makes clear, Rust believes the safest pattern is to allow either
a) a single mutable reference or b) multiple immutable references,
but *not both* at the same time.
This mutual exclusion pattern is implemented by several language abstractions:

- Borrowed references enforce it at compile-time
- RwLock mutual exclusion lock enforces it at runtime across thread
- RefCell mutual exclusion enforces it at runtime within a single thread

RwLock and RefCell are needed because sometimes our programs
really do need to store and work with multiple references to the same object across a wide programmatic scope.
When we want to access the value from one of them, we must first acquire a read or write lock.
Such a lock not only restricts mutability to a single reference,
it also ensures that no other reference has read access for the duration of any mutable lock.

How does mutual exclusion solve the above safety issues?
Because these safety issues all result from having two references,
one of which is mutable.
Mutual exclusion makes two such references impossible to attempt,
either due to a compile-time error from the borrow checker
or a runtime panic from a lock.

Going beyond the examples given above,
Manish also praises how mutual exclusion helps enforce certain programmatic invariants.
Although mutual exclusion cannot prevent every way to violate an invariant,
it is true it can prevent a very specific and hard-to-diagnose kind.
This happens when code logically cycles back to where mutation of an object began,
attempting to make use of another reference to the same data we still have "open" for mutation.
Mutual exclusion makes this impossible to attempt too, for much the same reason.

Fear can be very persuasive. Manish paints a forbidding picture of danger at a distance 
and then reassures us of the potency of mutual exclusion as the preferred cure.
In doing so, he understandably glosses over the added costs this imposes over shared, mutability:
the performance hit and code complexity overhead of runtime locks along with
the architectural inflexibility of only allowing one mutable (and **no** read-only) 
reference within a given scope.

There is a lot to like about mutual exclusion, even the single-threaded variety.
But I would like to discover situations where we can just as safely choose the
faster and more flexible shared, mutable permission.

## Revisiting Invariant Protection ##

Mutual exclusion cannot prevent invariant violations.
At best, it helps only with some minority of those exposures.
Where one has reason to fear that something like Manish's execution cycle example *could*
manifest itself in their code, 
mutual exclusion may well be a quick and effective antidote.
However, I am left to wonder whether code written to be so vulnerable
to invariant violation has been architected well to begin with.

Hygienic architectures carefully isolate all code that temporarily holds
data that violates an invariant.
Invariant-conforming data fields are kept private
and inject code that works with that data into
atomic methods/functions that are simply not susceptible to code cycles,
at least while the data is in an "incomplete" state.

In my experience, dealing with invariants is either largely unnecessary
or often easily isolated in this way.
Using these simple safeguards significantly minimizes how often
we really need to use mutual exclusion to successfully protect invariants.
I would rather see a language adopt mechanisms like
contract-based programming or perhaps dependent types
as more comprehensive and robust ways to enforce invariants.


## Revisiting Safety ##

Let's turn our attention back to the examples shown above,
to discover where it might be safe to allow shared, mutable instead of mutual exclusion.

In these examples, a fundamental pattern leaps out:

- One reference is an interior reference:
  a reference to some sub-structure within a larger, composite structure.
- The other (mutable) reference changes the "shape" of the composite structure.
  By shape, we mean that it alters some typed characteristic of the structure
  (e.g, the dimension of an array or the type for the inner value the variant type contains).

This makes sense: if we capture a frame for what lies inside a structure,
then change the structure's shape where the frame is positioned,
of course subsequent use of that frame will be problematic!
If we can prevent this specific sequence of events from happening,
then shared, mutable references become safe to use!

## Statically-shaped types ##

In an edit to his post, Manish acknowledges
that certain types do not require mutual exclusion for safety:
"In case of many primitives like simple integers, 
the problems with shared mutability turn out to not be a major issue.
For these, we have a type called Cell which lets these be mutated and shared simultaneously."
Cell offers no support for directly accessing, or obtaining interior references to,
a Cell-wrapped struct's fields.<sup>1</sup>

Such restrictions are (fortunately) unnecessary for safety.
Any type that does not change its "shape" at runtime can safely support
static, mutable references *even if* those references are
borrowed interior references.
This includes not only primitive numbers (e.g., integers and floats), but also
pointers, references, fixed-size arrays,
and (importantly) every user-defined struct-based type.

With this leap we are already a lot better off,
now that we only require mutual exclusion for resizable arrays and sum-typed values!
Can we do better still?

## Interior reference vs. shape restrictions ##

As already mentioned, the safety issue with resizable/single-owner and sum types
only arises when we change an object's shape *in between* obtaining an interior
reference and dereferencing that interior reference.
There are several approaches we might take to prevent
this sequence of events.

- **Invalidate borrowed interior references whenever we detect a shape change.**
  When this logic entirely happens within a single function,
  we could enforce this. Logic spread out across function boundaries
  makes it significantly more difficult to detect and restrict this pattern.
  
- **Invalidate shape changes whenever we know a borrowed interior references exist.**
  This bears the same difficulty as above.

- **Do not allow shape changes to objects referred to by shared, mutable references.**<sup>1</sup>
  Thus, arrays would not be resizable and sum-typed values could not be altered to
  hold a value of a different type.
  These restrictions are achievable, but significantly reduce their capability.

- **Do not allow the creation of interior references to array/sum values**
  pointed to by shared, mutable references.
  This is easily accomplished.
  Although this imposes some restrictions, these restrictions have workarounds:
  indexing can be used to retrieve values from arrays and the value enclosed
  by a sum type can be copied in or out.

  *Note: The last two options are not necessarily mutually exclusive.<sup>1</sup>*

Intriguingly, this last option is the one taken by safe, non-systems programming languages.
Because they offer no capability to create interior pointers, they cannot
experience this shared, mutable safety exposure.
Instead, access to an array's elements is made possible via indexing and (sometimes) slices,
both using indirection
and runtime-bounds checks to ensure that shape changes won't trigger safety violations.

It is worth noting that these indexing and slice mechanisms carry a performance
overhead on every use.
Depending on the program's requirements, it might be cheaper to take
the performance hit on obtaining a mutual exclusion lock rather than
the added pointer indirection cost imposed by indexing.
Additionally, an index makes no guarantee that it always "points" to the same
element, as element insertion or removal can shift an indexed value.<sup>2</sup>

## Summary ##

To offer protection against the safety challenges of shared, mutable references,
the following permission mechanisms are needed beyond those described in the previous post:

- **mutex1**, a single-threaded mutual exclusion permission.
  It cannot be dereferenced directly.
  It uses a runtime lock to ensure one may only obtain one mutable borrowed reference
  or multiple immutable borrowed references. It cannot be moved to or shared across threads.
  
- Allow references to all non-shape-changing types to be `mut` (shared, mutable) without
  any additional restrictions.
  
- For `mut` references to shape-changing types, enforce the constraint that one cannot
  create interior references to their values.

-------------

<sup>1</sup> Hat tip to Russell.

<sup>2</sup> Hat tip to matthieum