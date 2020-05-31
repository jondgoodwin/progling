---
title: "Region Modules: The Rest of the Story"
date: 2020-05-31T14:55:29+10:00
draft: true
---

## Static and Dynamic Regions ##

Let's switch gears. So far, we have described the working relationship between
references and regions, but have not described the independent nature of
the regions themselves: their state and their runtime logic.

Earlier, it was stated that region annotations on references are programmer-definable `struct`-like types,
that specify their own fields and methods.
In truth, regions are more than these reference-based region annotations.
A region is actually fully defined by an importable module, within which lies:

- the region annotation definition (reference-based state and methods)
- the region's global state
- the region's functions, the API that offers up the runtime logic for the region

The details will obviously vary greatly from one region type to the next.

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

### First-class Regions ###

Integer (non-pointer) references used by first-class
arenas or pools, as their mechanisms work differently in important ways.
Lifetime generativity on return values.
