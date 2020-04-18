---
title: "Unified Variant Type"
date: 2019-08-13T10:05:24+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "types"
tags:
  - "Cone"
---

The prior post, [When Sum Types Inherit](/post/when-sum-types-inherit/),
shows that we can (and should) use the magic of inheritance
to enrich sum types to offer as powerful a capability as traditional classes.
Graph-based, variant node data structures benefit from variant nodes
being able to support common fields, methods, and virtual dispatch.

That result, however, raises a new question: do we really need
two separate language abstractions that offer very similar capabilities?
Would it be better to unify them into a single, more versatile abstraction?

## Two Dualities ##

To help answer that, let's analyze the remaining semantic differences between them.
These differences boil down to two dualities,
corresponding to the horizontal and vertical dimensions of
a variant node inheritance tree:

* (Horizontal) Are the number of variants open or closed?
  They are closed if all variants are known and enumerated within a single module.
  They are open if other modules can extend the tree
  with any number of additional variant types.

* (Vertical) Are variant nodes fixed-sized or variable-sized?
  Their size is fixed, if the data size for all nodes is
  set to the size of the largest node type.
  Their size is variable, if each node takes up only as much room as its state requires.

As it turns out, the choice we make on each of these dualities
has a noticeable impact on how we implement them,
which in turn results in different advantages.
Let's take a closer look...

### Closed vs. Open ###

Pattern matching uses the discriminant to
figure out at runtime which variant type a value currently is.
The discriminant (or tag) is an "invisible" field,
either stored within the value's data
or located using the pointer to that data.

Closed vs. open determines how the discriminant is encoded
and pattern-matching performance:

* For closed variant types (like sum types), we know
  how many variants there will be at compile-time,
  as they are all enumerated within the same module.
  The discriminant encodes the variant type using 
  an different integer value for each variant type.
  Integer-based pattern-matching dispatch is quite quick.
  
* For open variant types (like classes), the
  compiler/linker cannot assign a unique 
  sequentially-enumerated integer to
  each variant type. Accomplishing the same result
  requires use of a unique pointer to type metadata.
  Pattern-matching will be slower,
  as it has to check each possibility one-at-a-time.

If we *need* to implement extensible graph data structures that
make use of node types implemented across multiple program modules,
then we need the flexibility of open variants.
However, when we don't need that flexibility,
we should implement them as closed, to get faster pattern-matching,
as well as the guarantee that pattern matching explicitly handles all variants.

### Fixed vs. Varying Size ###

Forcing every variant type's values to have the same size
offers these potential benefits:

* Variant type values may be passed around by value or reference.
* A value of one variant type may be replaced (in place) with
  that of a different variant type.
* The pool memory allocator may be used to more rapidly
  allocate and free variant values.

Reducing memory consumption is the primary advantage
for allowing variant type values to vary in size.
When the variation in size is significant, this factor
might outweigh the fixed-sized advantages listed above.
This is particularly true if those potential advantages
don't apply to use of the type.

## Four possibilities ##

Across these two dualities lie four possibilities.
Which should we support?

* **Closed and Fixed-size**.
  This describes sum types. 
  This offers the best performance, making it ideal when
  we have little concern about memory waste and no need for variant extensibility. 

* **Open and variable-size**.
  This describes classes.
  This is needed when variant extensibility is a real requirement.
  
* **Open and fixed-size**.
  If we don't know how many variants there will be, the compiler
  cannot deduce the largest size.
  Although this difficulty could be overcome by using a programming-specified
  size (with the compiler rejecting types that are larger),
  it is a risky, untidy solution.
  
* **Closed and Variable-size**.
  This is the best option if memory waste is a concern
  and faster pattern-matching is desired.
  This is particularly attractive if the variant types are recursive
  (and therefore cannot be value-based), we don't need in-place mutation,
  or some other memory allocation strategy is preferred to pools.

Three out of the four possibilities seem worth supporting, 
each ideal for different requirements on variant type use.

## Unification ##

Now that we know what variant type capabilities we want.
how might we unify these capabilities into clear, simple syntax?
Cone's [traits](http://cone.jondgoodwin.com/coneref/reftrait.html)
offers one potential design strategy:

* The base type is a trait, an abstract type which cannot be instantiated.
  It may define common fields and methods.
  Its methods need not be implemented.

* The variant type is a struct, a concrete type which inherits from a trait.
  It must implement methods not implemented by the trait.
  It may override methods implemented by the trait.

* The trait type may be defined using either `trait` (variable size)
  or `enumtrait` (fixed-size). When using `enumtrait`, all variant types
  must be defined in the same module as the trait type.
  
* The trait may explicitly define a tag field, specifying its
  size and position. This decides whether variants are tagged or not,
  and therefore whether indexed pattern matching is supported.
  If a tag field is specified, all variant types must be defined
  in the same module as the trait type.

* Appropriate support is provided for constructors, upcasting,
  downcasting, and virtual references.

The resulting design gives the programmer greater
versatility than either classes or sum types alone
when choosing how to effectively implement variant types
that well serve their required use.
Two clear knobs can be easily flipped to best achieve
the desired performance and space trade-offs.

