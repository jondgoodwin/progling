---
title: "Race-Safe Strategies"
date: 2018-12-16T10:21:16+10:00
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
  - "Pony"
---

I recently made the observation that many people seem unaware of
the full collection of constraint mechanisms available for protecting race safety.
Someone sensibly asked for a link to an article that provides a modern, comprehensive review.
It turns out that the pickings are very slim;
the best I could find is this Wikipedia article on 
[thread safety](https://en.wikipedia.org/wiki/Thread_safety#Implementation_approaches).
It's accurate, but incomplete.
To close that gap, let me take a stab here at more comprehensive treatment.

In particular, I want to focus on when a thread can send references to another thread
(or share them via global variables).
If such references are handled poorly, these threads could use such references to trigger a 
[race condition](https://en.wikipedia.org/wiki/Race_condition)
where interleaved actions by both threads on the same object results in a 
[programming bug](https://en.wikipedia.org/wiki/Race_condition#Example).

I will enumerate the various race-safe strategies
by categorizing references according to the unique constraints
each category imposes on how a reference of that type can be used.
Let's begin with the more well-known strategies.

## imm - immutable references ##

The state of a referred-to immutable object cannot be changed (mutated) after it has been created.
Because it can never change, all access to the object is read-only.
Therefore a race condition (which requires the possibility of mutation) is impossible.

Immutability is often the preferred strategy of functional programming.
If your data never really needs to change, it's a great approach.
When your data does need to change, there are several clever ways to
"mimic" mutation without actually changing a value in place (e.g., recursion, monads,
and persistent data structures). However, these techniques can often be
slower, more memory hungry, and more algorithmically complicated than in-place mutation.

## mutex - mutual exclusion ##

[Mutual exclusion](https://en.wikipedia.org/wiki/Mutual_exclusion)
means that retrieval or mutation of the reference's value is allowed only if
the lock to that value has been acquired.
This protects against race conditions by forcing threads to take turns one-at-a-time
(serialization) when accessing the same value. While one thread holds the lock,
every other thread needing access to that value must wait until that thread frees the lock.
Then the next thread can take its turn by acquiring the lock, and so on.

When two threads really do require shared, multi-operation, and possibly mutable access
to the same value(s), mutual exclusion is often the best solution.
One downside of mutual exclusion is a reduction in multi-threaded throughput.
Besides the runtime overhead of acquiring and freeing the locks,
escalating contention over locks means a lot of thread time is wasted waiting on a lock to
become available.
Another downside of mutual exclusion is the risk of 
[deadlocks](https://en.wikipedia.org/wiki/Deadlock)
and other synchronization challenges.

## atomic - atomic references ##

There are times when the operation we want to perform on a referenced value can be
performed using a [single, machine-language instruction](https://en.wikipedia.org/wiki/Linearizability#Primitive_atomic_instructions).
Since these operations are atomic, the shared data is always kept in a valid state,
no matter how other threads access it.

Many languages make use of atomics, but usually "under the covers",
often just as a way to implement the mutual exclusion locking mechanism.
However, some languages do surface the use of atomic references explicitly
for use by programmers
(for example, [C++ std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)).
Atomic references should be used whenever operations on the value are always "atomic".<sup>3</sup>
Atomic references are not suitable if multiple operations sometimes need to be performed
on the referred-to value in a serialized way.

## uni - unique reference ##

If we can guarantee that only one live reference exists to an object,
we can safely move that reference around from thread to thread.
Any thread may use that reference to retrieve or change the reference's value
without any risk of race conditions, since no other thread
can have a reference to the same object at the same time (much less use it).

This unique reference capability, based on
[linear/affine types](https://en.wikipedia.org/wiki/Substructural_type_system),
is relatively new to languages.
Although Rust has done a lot to popularize the idea, you do find it supported in other languages
(e.g., Pony's ['iso' reference capability](https://tutorial.ponylang.io/capabilities/reference-capabilities.html)).
So long as one is dealing with data values that only have a single owner
(e.g., simple hierarchical data structures), 
the unique reference strategy offers the fastest performing way
to move mutable values from one thread to another.
Its downside is that unique references cannot be used with data structures
that require multiple usable references to the same value at the same time.<sup>1</sup>

## mut - single-threaded, shared mutable ##

Any reference created and marked as shared mutable may never leave
the thread it was created in.
Even when it is possible to create many copies of such references,
they all reside in the same thread. None may leave.
Since all the references to this object live in the same thread,
a race condition between two threads is impossible.<sup>2</sup>

Shared, mutable can be a valid race-safe strategy to use when you
know that no other thread needs access to a created, mutable object.
It has no runtime performance penalty.
However, if you do need to safely share a mutable object between threads at the same time,
you will need to use a mutex reference strategy that
protects value access with a mutual exclusion lock.

## "What if I want them all?" ##

I suspected you might!

It *is* possible to design a language capable of supporting all these strategies.
There are several different ways to go about it.

The way I have chosen for [Cone](http://cone.jondgoodwin.com/) is to call these reference categories
**[permissions](http://cone.jondgoodwin.com/coneref/refperm.html)** 
and to annotate them as part of a reference's polymorphic type.
Thus, the reference type `&imm i32` is a reference to an immutable integer and
`&mut Point3` is a reference to a shared mutable value of type Point3.

One of the beautiful things about a language's type system is that
the compiler can use these reference type annotations to enforce the appropriate race-safety constraints.
The compiler enforces these constraints
at compile-time during the type-check and data-flow passes,
and (in some cases) at run time via generated code (e.g., lock acquisition and free).
In particular:

- **imm** references may never be dereferenced for mutation.
- **mutex** references may never be dereferenced in any way.
  The lock is acquired (using generated runtime code) when a borrowed reference is acquired.
  The borrowed reference may be used to view (or maybe change) the reference's value.
  At the end of the scope for the borrowed reference, the lock is freed
  using generated runtime code.
- **atomic** references only allow the use of certain atomic operations on the value.
- **uni** references prevent aliasing via move (and borrow) semantics.
- **mut** references can never be sent to another thread.

By annotating every reference with a permission, the programmer picks that reference's constraint "poison".
Cone enforces these constraints as just another aspect of the language's overall type system.

There is more to say about permissions, but let's save that for future posts!

--------------------------------------

<sup>1</sup> To be more precise, Rust's borrowed references
do make it possible to create and use temporary aliases (copies)
of a unique reference owner. Only one live mutable reference is usable at a time,
however multiple immutable borrowed references can be created and used.
Borrowed references have other constraints: their lifetime expires at end of scope
and they cannot be shared with other threads.
So long as a borrowed reference exists, the unique reference owner is inactive
and cannot be moved to another thread.
Because of these constraints on borrowed references, neither unique references
nor borrowed references are able to trigger a race condition.

<sup>2</sup> Although shared, mutable references are always race-safe,
they are not always memory- and type-safe.
Safety issues that look suspiciously similar to race conditions
can crop up when a language supports the creation of "interior references"
to shared, mutable values of certain types.
This challenge (and various solutions) will be explored in a future post.

<sup>3</sup> As matthieum correctly warns: at the micro-code level
atomics make use of hardware locks, which consume extra cycles to resolve.
Such locks ensure, for example, cache accuracy for memory shared by multiple CPUs.
As Matthieum states: "The cache request to go from "shared" to "exclusive"
will bubble up the cache hierarchy,
and may end up requiring a broadcast message to all other cores... 
including cores on another socket... and wait for their reply."
Thus, use of atomics is "not a free lunch".

<sup>4</sup> As matthieum observes:
Before sending an `imm` or `uni` reference to another thread,
a write barrier needs be emitted to ensure that CPU out-of-sequence optimizations
or accidental moves do not violate the program's intended sequence of events
or invariants.
