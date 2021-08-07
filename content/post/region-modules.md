---
title: "Region Modules: The Rest of the Story"
date: 2020-05-31T14:55:29+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory management"
  - "Modules"
tags:
  - "Cone"
---

The previous [Lifecycle of a Reference](/post/reference-lifecycle) post
illuminates the working relationship between references and regions.
It explains how a reference's region annotations are programmer-definable `struct`-like types,
which can specify an allocated object's region state using fields,
and region-based operations on an object (e.g., allocate and free) using methods.
This, however, leaves out an important part of the story about region definitions:
their global state and global runtime logic.

Holistically, a Cone region is fully defined by an importable module, within which lies:

- the region annotation type(s) used by references
- the region's global state
- the region's functions (API), which defines the runtime logic for the region

Future posts will talk more about Cone modules. 
For now, a region module can be accurately characterized as a library package
that has its own namespace distinguishing public names (the API)
from private names and implementations.
Unlike types, a module is a singleton with a global state (ambient environment).
However, Cone modules (like types) may define initializers,
auto-run before a program starts, and finalizers, auto-run when a program shuts down.
Later on, we will talk about how valuable this capability is to certain regions.

With that background in place, let's take a look at how
we might define the very diverse collection of region types
which can be supported by Cone.

## Static Regions ##

Most regions are static (as opposed to first-class, which will be reviewed later). 
This means their state is global and readily accessible
anywhere in the program. 
Because these regions have global state, the potential lifetime of references
allocated from these regions is 'static, which allows them to be usable anywhere in the
program.

### Single-owner Region ###

Single-owner is the easiest region to implement.
It is so simple, here is its entire implementation:

    mod SingleOwnerRegion:
      region @move so:
        fn _alloc(:
  
Single-owner is the easiest region to implement, as it requires no
Cone-defined global state nor runtime logic, 
since it piggy-backs off a general purpose allocation functions like malloc/free.
Therefore, the single-owner region module only needs to define the region annotation type
used by references.



### Single-owner, Reference Count and Static Arenas ###

For single-owner and reference counted regions, their state and runtime logic
is  minimal and is often off-loaded to the operating system or language runtime.
These regions typically hook into a general-purpose allocator (e.g., malloc/free),
but may alternatively hook into a lower-level API (e.g., mmap).
The state of these regions is effectively global (static)
and is simply a part of the assumed ambient environment of a program.

A static arena region is almost as easy to implement, and can also
be built on top of malloc/free. One approach is to have a global, nullable, mutable
variable able to point to a linked-list of large, allocated blocks of memory.
Object allocation uses a bump-pointer to carve out a slice of the latest block
of memory. As each block fills up, another is added to the start of the linked list.
Just before the program ends, runtime arena logic would be called able to
free all memory blocks in the linked list.

As a safe convenience to programmers, Cone allows an imported module
to specify an initialization function, to initialize its required global state,
and a finalization function, to clean up any accumulated state.
So, when a program imports a static arena region library, the global state
it requires is already correctly set up and ready-to-go.

### Tracing Garbage Collection ###

Tracing garbage collection regions are much more complicated to implement,
requiring more extensive global state and runtime logic.
The largest part of the runtime logic is the independently-executed garbage collector,
but also includes its own custom allocator.
The global state needs to keep track of the root collection of references
all objects that have been allocated and are still alive for all generations,
and the current state of the garbage collector.

There are so many varieties of tracing garbage collection regions.
It is beyond the scope of this post to get into any of the working details
of any of these strategies. 
However, it is relevant to describe how the compiler facilitates
key aspects of tracing garbage collection logic:

- 

Need tracing maps/functions, safepoints, root set, stack tracing, RTTI.

## First-class Regions ##

What does first-class region mean?

Integer (non-pointer) references used by first-class
arenas or pools, as their mechanisms work differently in important ways.
Lifetime generativity on return values.
