---
title: "The IR Tree: The INode Interface"
date: 2018-08-08T09:04:42+10:00
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
  - "Pony"
  - "Go"
---

Having spoken about the compiler's [IR tree in general terms](/post/the-ir-tree-intro),
let's focus in on an important detail: how to represent a node that could be any arbitary type.

Cone's IR makes use of dozens of different types of nodes, each defined using a different struct.
However, sometimes the compiler needs to point to a node 
without restricting which type of node it must be.
For example, consider an assignment node.
It needs to point to some node that represents the value on the right-hand side.
Each instance of an assignment node might need to point to a different type of node,
perhaps a literal, or a variable or some node representing a more complex expression.
Let's call a node an "INode" when we don't (yet) want to restrict what type it can be.

Representing an INode is easy when a compiler is written in a language
that supports either [variant/sum types](https://en.wikipedia.org/wiki/Tagged_union)
or [interfaces/traits](https://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)).
Nodes of known type are handled by defining a distinct struct for each specific type of node.
INodes are then handled by
either enumerating all the specific node types into a sum type
or else defining an interface or trait type that generalizes all the specific node types.

Coercing a specific node into an INode (upcasting) is safe and useful.
Coercing an INode into a specific node (downcasting) is also useful, 
so long as we do so safely using pattern matching.

## Cone's INode implementation

Cone's compiler is written in C, which supports neither sum types nor interfaces.
Fortunately, there are straightforward ways to concisely mimic similar abstractions in C,
albeit with less-enforced safety.
The essential technique is to use structs that we know can be safely 
[cast](https://en.wikipedia.org/wiki/Type_punning) to other structs.

In the Cone compiler, [INode](https://github.com/jondgoodwin/cone/blob/master/src/c-compiler/ir/inode.h)
is such a struct.
The structs for all node types, include INode, use the INodeHdr macro
to ensure all nodes begin with the exact same node header fields.
These fields specify:

- **The node's lexer context.**
  The only time this information is used is when the programmer makes a mistake.
  Diagnostic error messages use this lexer context to highlight exactly
  where the source code goes astray.
  (Note: The Go compiler demonstrates an
  [interesting way](https://github.com/golang/go/blob/master/src/go/token/position.go)
  to reduce the size of this information.)
- **The node's type**.
  This integer can be used to pattern match any INode and thereby correctly downcast
  an Inode pointer to the appropriate node-type-specific struct.
- **Node-specific flags**. These typically help specialize a node's properties.
  For example, a function would use these flags to indicate whether it is
  a method (vs. a function) or defined externally (and therefore has no implementation).

It is worth pointing out that Cone's node header does not include
a reference to the node's parent, as some languages do (e.g., Pony).
When walking the IR tree during semantic analysis, the compiler only rarely
requires contextual information from above.
When this context is required, it is captured as part of a mutable state
that is passed around throughout the traversal.
So far, it seems easier to do it this way vs. implementing an intelligent reverse traversal.

## Node group and name flag

The type field is more than an enumerated integer value:
it's actually a smart value that encodes additional node type properties as bits.

The type bits specify which group the node type belongs to.
A number of logic decisions can be quickly and helpfully based on node group.
The groups are:

- **Expression**. These node types return a typed-value.
  All assignment, function call, literal, variable, block and if nodes are expressions.
- **Type**. These node types refer to or define a Cone language type.
  Number, struct, function signature, and reference nodes are all part of the type group.
- **Statement**. These node types are neither expressions or types.
  Module and while nodes are part of the statement group.
  
The type field also has a bit indicating whether the node is named.
(e.g., nodes for functions, variables, many types, and modules).
From a namespace point-of-view, it is helpful to be able to quickly
know whether a node is named, regardless of its type.
The next post talks a lot more about named nodes and namespaces.

## Nodes = a list of INodes

Certain types of nodes need to refer to a list of nodes,
whose varying number is determined by what's in the source program.
For example: A *block* node holds any number of statements.
Likewise, a *function signature* node holds a varying number of parameters.

Such nodes use the Nodes struct to hold a list of nodes.
Nodes is basically a simple, ordered, resizable array of INode pointers.
It automatically doubles in size whenever it fills up.
A macro is used to iterate over its list of nodes (used during IR traversal).

Use of Nodes almost always will waste some memory space.
As an alternative, some compilers (e.g., Pony) use a linked-list approach,
where every node can tell you the sister node that is next in line.

## INode dispatchers

Compilers written in a language that supports 
[subtype polymorphism](https://en.wikipedia.org/wiki/Subtyping)
(as most object-oriented languages do)
can easily dispatch to node type-specific logic during IR tree traversal.
This is accomplished using a simple method call on a node.

Cone's compiler is currently written in C,
which provides no built-in abstractions for method dispatch.
To compensate, INode-based dispatch functions (e.g., inodeWalk) have been 
[manually constructed](https://github.com/jondgoodwin/cone/blob/master/src/c-compiler/ir/inode.c)
that perform the same work.
As this code demonstrates, inodeWalk is simple: it switches on the INode's type
and then calls the appropriate node-specific walker,
passing it the context and the recast, node-specific pointer.

One recent improvement to the inodeWalk dispatcher is that you
pass it a pointer to an INode reference.
This double layer of indirection makes it possible 
for a node-specific semantic pass to replace itself with another
node in the IR tree or to inject nodes in between.
