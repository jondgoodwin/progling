---
title: "The IR Tree: Typed Nodes"
date: 2018-08-14T17:01:25+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Intermediate Representation"
tags:
  - "Cone"
---

As mentioned in [the previous post](/post/the-ir-tree-named-nodes),
all typed nodes use the TypedNodeHdr common header.
It only contains a pointer to the node's type.
This type field applies to every node in the "expression" group.
This group holds all node types that return a value, including leaf nodes like literals and variables,
function call nodes, and even the block and if nodes.

The type check pass focuses largely on this type field,
ensuring it specifies a valid, consistent type.

## Type Inference

The Cone language uses a simple, uni-directional type inference strategy.
This strategy was chosen to ensure that inference was fast and deterministic
in the face of Cone's support for subtypes and complex reference types.
The downside of this strategy is that a program must have some type annotations.

Unidirectional type inference largely expects all literals and function/method signatures
to be fully and explicitly typed.
(Sometimes, a default type can be safely assumed based on the context.)
Similarly, variable declarations need to specify a type, unless given an initial value.

Given the presence of this explicit type information, the types of all expressions can be
inferred progressively from the inside out.

## Type check matches

Similar to how name resolution matches every name use to a name declaration,
type checking attempts to match the type of an expression's value to the type of
the receiver for that value. For example:

- Assignment nodes match the right hand expression's type to the type of the lval/variable.
- Function call nodes match the values of its passed arguments to the declared types of
  the function's parameters.
- Return nodes match the values of the returned value to the return type declared
  as part of the current function's signature.

Cone has an unusual and somewhat complex collection of reference types.
References not only declare what type of value they point to,
they also declare an access permission and allocator (or lifetime).
A successful reference type match requires that all three aspects match.
This need to match multiple aspects is required with other types as well,
such as function call parameters and struct properties.

## Type subtype coercions

For the sake of flexibility and usability, Cone will not always enforce a strict type match.
When the declared type of the receiver is a valid subtype of the type of the sent value,
Cone will consider it a valid match, and will even inject an implicit conversion as necessary.

Examples of valid coercions include:

- A struct pointer to a valid interface/trait pointer
- A static array to an array reference
- A reference to or from a pointer (within a trust block)
- Static permissions to a valid permission subtype
- An integer to a floating point number

The one place where this can get complicated is when determining which of several
overloaded methods is the "best" match for a method call.
A type preserves the order of its declared methods, which establishes the search order.
Most of the time, the first method that matches is the one selected.
However, sometimes Cone uses a simple scoring system to ensure selection
of a method later in the search order whose match is more perfect than a method
whose declared parameters were acceptably close subtype matches (but not perfect).
