---
title: "Can a compiler guarantee multi-owner memory safety?"
date: 2018-12-01T12:37:15+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory Management"
tags:
  - "Rust"
---

Is it possible to improve on Rust's single-owner strategy to support more complex data structures?
Before digging into this challenge, let's summarize the story so far...

## The Promise and Limitations of Single-Owner ##

Rust's single-owner memory management is a form of 
[automatic memory management](https://en.wikipedia.org/wiki/Category:Automatic_memory_management);
a [garbage collection strategy](https://words.steveklabnik.com/borrow-checking-escape-analysis-and-the-generational-hypothesis)
that is distinct from tracing and reference-counting.
Fundamentally, it is an improvement on [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization),
which automatically finalizes some defined resource at the end of its defined lexical scope.
Rust adds to this model two important innovations:

- The resource may be moved (and therefore escape from) its current scope.
  This allows it to travel freely from scope to scope, even function to function.
  Whichever scope it does not escape from carries the responsibility for finalizing (freeing) it.
- References can be created that "borrow" access to the owned resource.
  Such references are lifetime-constrained to the scope they are created in,
  ensuring a borrowed reference can never outlive the owner reference.

One of the key benefits of single-owner memory management is performance.
Unlike reference counting and tracing, there is no runtime bookkeeping overhead
needed to calculate when it is safe to free the resource.
The compiler can completely and accurately calculate this at compile-time.

A key downside to single-ownership is that it restricts the data structure it supports.
As its name suggests, it works best for data structures
that need only one reference to an object: e.g., a hierarchical tree structure.
More flexible graph structures, such as cycles or node-joins, become potentially problematic.
This restriction is significantly more severe than ref-counting's leaky cycles.
The classic, simple example of a problematic data structure is the doubly-linked list.

Let's be clear here:
Rust does indeed support [doubly-linked lists](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) 
without runtime bookkeeping costs.
However, it accomplishes this by using unsafe blocks and pointers to
bypass the restrictions of the single-owner memory model.
Another technique uses [vector indexing](https://www.reddit.com/r/rust/comments/2u53le/this_is_a_doubly_linked_list_in_safe_rust/).
Using integers as references avoids the safety restrictions imposed on references.
However, indexing has its own downsides: performance is worse and
a runtime mechanism is required for bounds-checking.

## The multi-owner challenge ##

The data structure restrictions of single-owner is a source of frustration for many people.
At least for doubly-linked lists (and other data structures by extension),
why isn't there a way to manage and use multiply-owned references in a way
that the compiler can guarantee is type- and memory-safe,
without requiring any runtime bookkeeping overhead?
After all, Rust's implementation of doubly-linked lists is in fact safe, uses pointers,
and requires no runtime overhead.
Why is the compiler unable to verify this?

My intuition says that a compile-time-only memory management strategy
can never become as flexible as a runtime-based strategy like tracing.
This conjecture is likely provable.
That said, can't we find some way to provably extend Rust's single-owner
scheme to support some memory-safe flavor of multi-owner data structures?

Chris Hall and I had several conversations to explore this challenge.
At one point, I sent him an e-mail outlining one promising approach.
Since both of us had other more pressing priorities, nothing further came of this idea.
However, the topic recently came up again in an online conversation with
Jeff Walker and others.
I offered to capture the idea in this post on the off-chance that someone
else might wish to pursue it further.

## The Importance of Isolation ##

Let's simplify the problem to make it more tractable to a solution.
Our first simplification focuses on the isolation benefits of Rust's linked-list implementation:
Because the fields of the LinkedList structs are private,
direct public access to and mutation of the list's references is impossible.
Only LinkList's methods have the ability to retrieve or change key reference fields:
`head`, `tail`, `next` and `prev`.

The code that creates a LinkedList
is the single-owner of a specific, allocated LinkedList collection.
It uses methods to work with that list.
This method-based API constrains access to the content payload of nodes:

- A node's content can be moved into (insertion) or out of (deletion) a linked list.
  Practically speaking, a node's content only has one true "owner" at a time.
- A method can return a lifetime-constrained borrowed reference to a node's content,
  enabling the caller to view or change that node's contents.

One advantage of this isolation, for our purposes, is that it
reduces how much code the compiler must analyze to ensure that
owner references are always handled in a memory-safe way.

## The Automatic Memory Management Invariant ##

The second way to simplify our challenge involves distilling
what all memory management strategies require to guarantee safety.
The principle shared by single-owner, ref-counting and tracing GC
is this: free an allocated object when we first
discover that no references that we care about point to it.
Put another way, we need some fool-proof way to detect when
the reference count transitions from >0 to 0.

With ref-counting and tracing, runtime bookkeeping performs this detection,
because we simply don't have enough information to predict this at compile time.
By contrast, this transition *can* be predicted for single-owner at compile-time.
Why is this possible?  Because the language guarantees
(due to move semantics and lifetime-constraints on borrowed references)
that there is only **one** reference to the object at scope-end for that reference.
Thus the compiler can safely inject object finalization logic at scope end.

So, can we perform a similar compile-time prediction for a doubly-linked list?
Perhaps so ... since it turns out that every node,
at the completion of any LinkedList method,
always has exactly **two** owner references to it.
The number of owner references to a node are always either 2 or 0.

This suggests that we could generalize our compiler algorithm beyond linear logic,
supporting data structures where the number of references
is some compile-time predictable number.
If the compiler can prove that the number of references is quantized
such that it always jumps from some number (e.g., 1 or 2) to 0 by the time
an isolated method finishes its work, we can still achieve
the provable safety of a compile-time memory management strategy
that now supports some varieties of multi-owner data structures.

That sounds promising! Is it really that easy?

No.

## Counting Multi-Owner References ##

A compiler is surely able to analyze an isolated method and (in many cases)
calculate the net change to references based on watching references be copied and destroyed.
After all, compilers already do something like this for single-owner and for reference counting.
So, this is not an unreasonable expansion of a compiler's capability.

Unfortunately, noticing that a method decreases reference counts by 2
is insufficient cause for freeing a node.
The problem here is our indeterminism about which object's references
are being counted. If a method were to delete two references to different objects,
neither should be freed and this should be flagged as an error.

So, we need a counting mechanism that more precisely counts reference changes
on a per object basis.
The simplest such mechanism I have come up with for doing this on a doubly-linked
list involves the observation that whatever a node points at also points back.

With that in mind, what might be valuable would be declaring this invariant as a 
[contract](https://en.wikipedia.org/wiki/Design_by_contract) on the doubly-linked list type.
The essence of the contract would need to assert, for the compiler's consideration,
that if x.next (or x.head) points to y, then y.prev (or y.tail) must also point back to x.

Tracking this within the context of a specific method should be something the compiler can do.
The important thing is that the compiler is doing data flow analysis
on changes to reference fields in the two structs throughout the logic of each method.
Only at the end of the method, does it assess whether the method honored the
contract (invariant) successfully.
(It's okay if the contract has not been honored part way through the method.)
If any method fails the contract, a compiler error can be generated that
indicates memory safety cannot be ascertained by the compiler.

Intriguingly, the specific contract I describe for a doubly-linked list
could likely be generalized to any hierarchical tree structure where the children
also point back to their parents.
The more fascinating question is whether there are other forms of contracts
which might be formulated that make it possible for 
a richer range of multi-owner data structures to support compile-time
automatic memory management.

## Summary ##

This post is just a thought experiment on an intriguing challenge and solution-approach.
It may be promising, but a lot of work is still needed to:

- formalize how such reference-based contracts would be specified
- detail how the compiler would enforce such contracts via data flow analysis
- prove that the proposed contract guarantees memory safety

From my perspective, I did not pursue it any further because I have enough on my
plate to demonstrate that gradual memory management is viable and useful.
The value-add does not seem worth the cost for just a few additional data structures.

From a formal perspective, it is definitely worthwhile to demonstrate that doubly-linked
can be proven safe at compile-time. But from a practical standpoint,
Rust's choice to implement doubly-linked via a carefully isolated API that relies on
"unsafe" blocks works just as well to guarantee safety.
Until a richer contract approach comes along and is proven effective,
my feeling is that 
static enforcement of multi-owner data structures will add unnecessary extra
costs to the language, the compiler and compiler performance,
costs that may not be commensurate with marginal (if any) reduction in risk.
