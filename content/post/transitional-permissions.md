---
title: "Transitional Permissions"
date: 2019-01-12T18:19:02+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Permissions"
tags:
  - "Rust"
  - "Cone"
  - "Pony"
---

To complete our three-part series on permissions,
which began with [Race-Safe Strategies](/post/race-safe-strategies/),
let's talk about the transitional nature of reference permissions.

When are permissions transitional?
When we can safely create a copy of a reference which has a different
permission than the reference it copied from.
There are several ways in which this can happen, which this diagram
summarizes (and the following sections explain):

![Transitions](/images/PermissionTransitions.png)

The following sections describe the nature of several one-way transitions
that flow downward in the diagram. In the last section, we will explore whether
it is ever safe to swim upstream.
The capabilities described here broaden on similar features supported by
the Rust<sup>2</sup> and 
[Pony](https://tutorial.ponylang.io/capabilities/reference-capabilities.html)<sup>3</sup>
languages, and reflect capabilities planned for 
[Cone](http://cone.jondgoodwin.com/coneref/refperm.html)<sup>1</sup>.

## `uni` - the Universal Donor ##

As you may recall, the `uni` permission's constraint ensures that when
a `uni` reference is usable, no other reference to the object exists.
This is clearly true when an object is first allocated,
as we start off with just one reference to the new object.
[Move semantics](/post/move-mechanics/) turns
this `uni` reference into a solo, mutable traveller,
able to surf from scope to scope, thread to thread.

Because `uni` guarantees its reference runs alone,
we know that move semantics can also be used to safely transition it to a reference of another permission.
Move semantics not only makes a new copy of the reference with a different permission,
it also destroys the original reference (thereby terminating its single reference guarantee).

We transition away from `uni` when we need
multiple, shared copies of this reference.
We must choose one of the many shared permissions (the second row) to transition to.
When we use move semantics to
transition to any shared reference (whose aliases can then scatter in the wind),
we lose the ability to make another safe transition.
Thus, the transition is irreversible:
We cannot safely go back, nor can we safely go sideways later to a different permission.

Depending on our choice, object access will thereafter constrained in the same
way for the rest of the object's life:  either you can't change it (`imm`),
you can't share it across threads (`mut`),
access is managed by runtime locks (`mutex` and `mutex1`),
or access is limited to `atomic` operations.

## `opaq` - the Inscrutable Permission

Let's skip to the bottom of the diagram and introduce the new `opaq` (opaque) permission.
Crazy as it may seem, an `opaq` reference may never examine or change the
contents of the object it refers to.
Despite the severity of this restriction, there exist situations where
`opaq` is quite handy, such as:

- **Opaque structs**.
  Imagine your program receives a reference to some data from another system
  (e.g., a WebAssembly program getting a reference to a Javascript object).
  Your program has no insight into the composite structure of the data,
  nor even how much space it takes up.
  Under these circumstances, it makes no sense to try to access the data,
  so the read/write restrictions cause no problems.
  What you still can do, which is valuable, is pass the value around
  (including back to the program you got it from as a "handle")
  and compare it for equality to another reference you might have.

- **Functions**.
  A program's functions are code, which is data.
  However, commonly, a program is unable to view or change the executable code of
  a running program. Again, the `opaq` permission's restrictions pose no hardship.
  All function references are thus `opaq`, which helps enforce this,
  while still allowing "functions as first-class values" to be used to
  call functions indirectly. Function references may also be compared and passed around.
  `opaq` is also useful for other executable abstractions,
  such as actors or co-routines.
  
Where `uni` is the universal donor, `opaq` is the universal receiver.
A reference having any other permission may be transitioned to an `opaq` reference.
Other than from `uni` (which requires a destructive move), this transition is
performed via a simple copy.

## `const` and polymorphism ##

To introduce `const`, our final new permission, let's set the stage...

Not uncommonly, we want to call some function and pass it one or more references
which we know the function will never try to mutate.
Given that reference permissions are part of the type system,
when we pass one or more references to a function, the permissions of passed references
need to match the permission of expected references. 
Should we fail to enforce that match,
we open the door to race or other safety issues.
But, if we are too rigid and strict about this match, 
we force the creation (via generics or otherwise) of
multiply-typed versions of essentially the same functionality.

`const` offers a simple fix to this polymorphic challenge.
Like `imm`, `const` prevents the function from changing the value.
Unlike `imm`, it also prevents the function from sending the reference to another thread.
The guarantees of these tight constraints
make it safe to transition a `uni`, `imm`, `mut`, `mutex` or `mutex1`
reference to `const`.

In the case of `mutex` and `mutex1` (the runtime permissions),
we cannot just do a copy to create the `const` reference.
An extra mechanism is needed.
First, a lock must be successfully obtained.
Then, a borrowed `const` reference may be created.
When the borrowed reference expires, so does the lock.

## Permission Recovery ##

We have talked so far about permission transitions which only flow safely downstream.
Is it ever possible for a language to offer permission transitions that safely flow
upstream?

Sort of...

We can get close to this capability by making certain downward-flowing transitions temporary.
These transitions are explicitly performed within a certain scope.
When that scope ends, we recover the original permissions we had prior to the transitions,
much like "popping" a transition state stack.
Furthermore, safety requires that the inner scope be isolated so that it cannot
access certain source reference(s) outside that scope.

The Pony language offers a 
[`recover` mechanism](https://tutorial.ponylang.io/capabilities/recovering-capabilities.html)
that does exactly this.
The `recover` block establishes the boundary between inner and outer scopes.
For isolation safety, it restricts the inner block to "sendable" outer block references,
as explained by the "How Does This Work?" section in the linked tutorial page.
(Note: Pony uses the term "reference capabilites" for permissions,
and it uses very different names for those capabilities.
This [blog post](https://bluishcoder.co.nz/2017/07/31/reference_capabilities_consume_recover_in_pony.html)
offers an additional explanation of these concepts in Pony.)

Rust accomplishes the same thing via a different mechanism: borrowed references.
Conceptually, this works in a similar way:
When a reference is borrowed (possibly with a transitioned permission), the source reference may be
[frozen](https://doc.rust-lang.org/rust-by-example/scope/borrow/freeze.html),
which isolates the inner scope from certain outer scope references.
Every borrowed reference has a built-in, scope-based lifetime.
When that lifetime expires, the source reference is unfrozen and usable again,
essentially restoring the original permission.

Early on, we described transitions away from `uni` as irreversible.
And that is true if the transition is accomplished using move semantics.
However, if we perform the transition using borrowed references,
we open up an exciting new capability...

For some period of time, we can freeze and clone our 
scope- and thread-surfing solo traveler.
Within that arbitrarily-long lifetime, we can use these shared references as needed.
When that lifetime expires, we can once again recover our `uni` reference,
freeing it to once again surf across scopes and threads.
Later on, it can be re-frozen to a different choice of shared reference,
only to once again be recovered, and so on.

Allowing permissions to transition across references to the same object
is exciting, because it opens the door to
many useful data flow strategies and architectures!

--------------------------

**Figure**: *Comparing permission terms across languages:*

**<sup>1</sup>Cone** | <sup>2</sup>**Rust** | <sup>3</sup>**Pony** | **[M# (Midori)](http://joeduffyblog.com/2016/11/30/15-years-of-concurrency/)**
---------|----------|----------|----------------
uni      | mut      | iso      | isolated
imm      | *(default)* | val   | immutable
mut      | Cell     | ref      | mutable
mutex    | RwLock   |          |
mutex1   | RefCell  |          |
atomic   | atomic   |          |
const (&) | &        | box      | readonly
opaq     |          | tag      |
         |          | trn      |