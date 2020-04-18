---
title: "When Sum Types Inherit"
date: 2019-07-31T10:05:24+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "inheritance"
tags:
  - "Cone"
  - "Rust"
---

Inheritance makes it easier than any other mechanism
(e.g. generics, macros, composition/delegation)
to define a type that reuses the state and some methods of other types.
After reading my [inheritance posts](/post/favor-composition-over-inheritance/), 
I hope you are convinced that simplifying inheritance 
to a namespace-based mechanism ensures we obtain this convenient reuse capability,
while avoiding most of the complexity and coupling
dangers of traditional inheritance.

However, you might still wonder whether real-world code needs inheritance's reuse capability.
As one affirmative example, let's explore how best to work with 
[graph-based data structures](https://en.wikipedia.org/wiki/Graph_(abstract_data_type))
where the type of each node varies based on information gathered at run-time.
This is common in a number of programming domains, as illustrated by
AST/IR topologies (compilers), HTML-based DOMs (browsers),
and distribution networks (logistics software).

## Classes vs. Sum Types ##

What is the best way to implement such data structures?
Some programmers model them using object-oriented classes and inheritance.
Other programmers, concerned about class safety and performance,
prefer to model variant nodes using sum types.
Unfortunately, using sum types to implement such graphs
can sometimes run into some annoying limitations.

Let's summarize the differences 
between traditional classes and sum types using a simple example:
an internal data graph capturing the content of some arbitrary
[JSON file](https://www.json.org/).
This internal representation would capture JSON's seven distinct types of values
using six variant **Node** types:
**NullNode**, **BoolNode** (for "true" and "false"),
**NumberNode**, **StringNode**, **ArrayNode**, and **ObjectNode**.

* **Classes**. In an object-oriented language, these six variant types would typically be
  implemented as classes that inherit from a **Node** base class.
  **Node** might define additional state information in common
  across all nodes, such as the line and column position of the node's value in the source file.
  The base class might also define one or more virtual methods (e.g., "serialize"),
  which the six subclasses would each implement in a specialized way.

    When using classes, each node instance is often heap-allocated.
    Its allocation size varies based on the variant class's state.
    A variant class node can be safely and implicitly upcast to its base class.
    In that form, vtable-based virtual dispatch may be used to call virtual methods.
    Downcasting requires explicit casting, and may generate a runtime error
    if performed incorrectly.

* **Sum Types**. In a language offering sum types, a **Node** algebraic datatype
  would declare six variant types. So far, so good.
  However, common fields cannot be defined as part of **Node**,
  and would have to be specified outside the sum type.
  Further, many languages do not support methods on sum types,
  expecting functions to hard-code pattern matching to handle variant logic.
  In those that do, sum type-based methods can be awkward to use.

    All instances of a sum type are the same size, regardless of the variant type.
	This means one variant's value can be replaced with another in place,
	something that is not possible with differently-sized classes.
	Downcasting is safely (and exhaustively) handled using pattern matching.

That's a lot of underlying differences between these two abstractions,
despite their superficial similarities!
Wouldn't it be nice if we could collapse both abstractions into a single
unified abstraction that delivered the best capabilities of both?<sup>1</sup>
I am going to answer this question across two posts:

* In the rest of this post, let's first explore whether it makes sense to
  enrich sum types with common fields, methods, and inheritance,
  so they can support the same capabilities as classes.
* In the [followup post](/post/unified-variant-type), we focus on
  what it would take to converge classes and sum types
  into a more versatile, unified abstraction usable by graph-based data structures.

## Common Fields ##

Many sum types share no fields in common across all variants (except the discriminant).
However, some would benefit from this capability, as the JSON example illustrates.

When sum types do not support common fields,
they must be implemented in their own
record structure, and then separately made accessible to variant-specific logic.
This workaround is inconvenient 
and makes the resulting logic more complicated than necessary.

Given that graph-based variant nodes often need common fields,
it makes sense to enrich
sum types with the ability to define them.
These common fields should be accessible on a variant value
without the need for pattern matching.
However, they should also be accessible by any variant's logic
in the same way as its own unique fields.

Put in other words, we want variant types
to be able to inherit state from their base sum type.

### Constructors ###

To guarantee whole value integrity, languages with sum types typically offer
a built-in type constructor for creating a new value of any type.
For a variant type, the constructor expects the specification 
of a value for each of its fields.

Now that a variant's state now includes its base type's state,
how should the type constructor adapt to this extra fields.
There are two possible, safe approaches:

* The most convenient is a flat approach: the constructor
  accepts each common field's value alongside the values
  specified for the variant's unique fields.
  For example: `NumberNode{line: 10, col: 15, nbr: 4.5}`
* Alternatively, the sum type's constructor could be used
  to create a record for the common fields,
  and then that aggregate value is specified as one "field"
  within the variant type's constructor.
  This approach can be valuable when we want to make use
  of some function or method to create
  the aggregate value for the commmon fields.
  For example: `NumberNode{node: Node{line: 10, col: 15}, nbr: 4.5}`
  
It seems to me there are good reasons to support both approaches,
with the first being more likely.
It is also worth pointing out that "class" constructors are
safer when they also support this same approach.

## Methods ##

Languages with sum types typically handle them as data values.
Whenever logic needs to reach into a variant value,
a pattern-matching block is used in place 
to customize the desired logic for each variant.
Often, this direct, performant strategy is what you want.

However, there can be compelling architectural reasons to centralize
some or all of a sum type's behavioral capabilities into methods on the type.
This is attractive when variant logic is complex or when
changes to the internal composition or number of variants will likely change over time.

As with classes, we want to allow method definitions not only on each variant type,
but also the base sum type itself.
These base methods can then be inherited or overridden by each variant.

Static dispatch of such methods is certainly possible, when
we know at compile-time which variant is calling the method.
However, the more common scenario for sum types is that we don't
know until run-time which variant we have.
Thus, method dispatch is far more likely to be virtual.
Virtual dispatch can be an effective, almost-as-performant 
substitute for pattern-matching.

## Multiple Inheritance Levels ##

Through the magic of inheritance and polymorphism,
we have now brought sum types up to parity with classes.
Sum types now support:

- Common fields on the base type, accessible with or without pattern matching,
  and initialized as part of the variant type's constructor.
- Methods on base and variant types, supporting both static and virtual dispatch.

We should be prepared for the possibility that one level
of inheritance is insufficient for certain kinds of nodes.
This would be true when we want to logically group certain nodes together.

For example, in an AST, some nodes might evaluate to a value and others
might represent type declarations. 
All value nodes might share the same common fields (the type of the value),
different from the common fields used by all type declarations.
Two layers of inheritance would be needed to support this example,
with additional "intermediate" base types needed to capture
common fields and virtual methods for their variant types.

## Syntactic impact ##

By adding common field and methods to sum types,
we must anticipate the additional complexity and bulk
these features bring to a once-simple 
algebraic data type (ADT) syntax, 
where many sum types can be defined
in just a few lines of code within one lexical block.

It is not hard to imagine that for some complex graph-based nodes,
we might want to devote one or more source files to specify
all the methods (and state) for each variant node.
This is a common practice for languages that support classes.

To support this sort of modular file organization,
it would be helpful to flatten the declaration of all types,
and let the compiler use information in each type declaration
to stitch together their inheritance relationships.
	
## What's Next? ##

Now that sum types can do every thing that classes can do,
do we need two separate abstractions?
The [next post](/post/unified-variant-type)
explains how they still differ
and what we can do about those differences. 

----------------

<sup>1</sup> Niko Matsakis covers similar territory in his [blog post]
(http://smallcultfollowing.com/babysteps/blog/2015/08/20/virtual-structs-part-3-bringing-enums-and-structs-together/).