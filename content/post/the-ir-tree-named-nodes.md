---
title: "The IR Tree: Named Nodes and Namespaces"
date: 2018-08-09T19:27:41+10:00
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

Working out effective name handling mechanisms in Cone's IR took several tries.
The key challenges:

- Although most node types don't use names, the ones that do are spread out unevenly
  across every node group:  a few expressions (variables and functions), most types,
  and some statements (e.g., module nodes).
  What is the best way to gain access to a node's name information, given this disparity?
- Cone's namespace rules
  vary depending on the semantic context a name is declared within.
  How should various different namespaces be represented to ease correct name resolution?
- Namespace mechanisms allow importing of unowned names, including via an alias
- How can costly string matching be reduced to improve compiler performance?

## Interface

There is no interface usable for accessing a node's name regardless of its node type.
Thus, it is necessary to match against the node type first, or use an interface that
does have the name. These are common headers used for nodes:

- **INodeHdr** *(required)* - lexical context, type and flags
- **IExpNodeHdr** *(optional)* - the type of a value (for expression nodes)
- **ITypeNodeHdr** *(optional)* - info common across all types
- **INsTypeNodeHdr** *(optional)* - for type nodes containing a namespace of methods/fields

**Note**: A node type choosing to use any optional header gets all the fields of dependent headers.
The performance benefit of being able to use fixed offsets to find information quickly
is worth the price of some memory being wasted.

## Name Interning

A name is uniquely identified by its unicode (UTF8) string.
In an effort to reduce string comparison and handling costs, the lexer 
[interns](https://en.wikipedia.org/wiki/String_interning) all
source program names into the 
[global name table](https://github.com/jondgoodwin/cone/blob/master/src/c-compiler/ir/nametbl.h).

When the lexer encounters a name, it does a hash lookup to see if it is already in the
global name table. If not, it allocates persistent memory for the name using an extended
Name struct. The Name struct captures the name's string value, its length, its hash, and
the node it is associated with. A pointer to this Name struct is then stored in the
appropriate hashed cell of the global name table for future look ups.

IR nodes refer to a name 
simply by using a pointer to its Name struct (which is unique for that name and never moves).
By interning names, name equality checks become as fast as comparing two pointers.
Name resolution is also quick, because it often requires just a simple dereference
to retrieve the node associated with the name.

You might wonder: what if a program uses the same name in different scopes to refer to different declared names?
Hold on to that thought until we talk about namespace hooking.

## Namespace Rules

performing name resolution in Cone is complicated by Cone's
diversity in [namespace rules](http://cone.jondgoodwin.com/coneref/refnamespace.html):

- **Module** namespaces do not allow overloaded names and the order of named nodes is irrelevant.
  Because of this, module nodes use a hashed Namespace struct to facilitate fast lookup,
  especially for namespace-qualified names (e.g., Table::index).
- **Type** namespaces allow overloaded names for methods, and the order of its nodes is significant.
  Type declaration nodes use a Nodes struct to preserve the search order of its named nodes.
- **Function blocks** do not allow overloaded names within a block, but do allow an inner block
  to override a name defined in an outer block.
  Furthermore, the position of a named node within a block influences when it can be used.
  Block nodes use a Nodes struct to preserve the precise order of its named and unnamed nodes.

## Name resolution mechanics

Cone's IR uses different node types for name declarations vs. name references.
For example, a program can declare a variable (a name declaration node), and then later
refer to that variable name in expressions (nameuse nodes).

The purpose of name resolution is to traverse the IR tree and 
connect every nameuse node to the correct name declaration node. 
Several techniques are used (mostly during the first semantic pass) to
determine which declaration node matches a nameuse node:

- **Namespace hooking**.
  During tree traversal, whenever a node with a namespace is encountered,
  it will "hook" all its name declaration nodes into the global name table's Names
  (saving the nodes an outer scope earlier defined for each Name).
  Because of this hooking,
  an encountered nameuse node need only look at the node in its Name struct to see
  the lexically-closest name declaration node it needs to point to.
  The saved nodes hooked earlier for these names are automatically restored when the namespace goes out of scope.

- **Qualified names**.
  Some nameuse nodes represent namespace-qualified names (e.g., Module::inc).
  In these cases, name resolution requires that a specific name declaration node be searched
  for in the specified module or type namespace.

- **Method or property lookup**.
  The name resolution for methods and properties cannot be performed until after type checking
  has finished, as before then it is impossible to know which of 
  the object type's several overloaded methods
  is the correct parametric type match.

Name resolution is finalized by filling in a
field in the nameuse node that points to the found name declaration node.
Where appropriate, the nameuse node also copies over the declared name's type information,
thereby simplifying future type inference work.