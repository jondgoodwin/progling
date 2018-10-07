---
title: "Data Flow Analysis"
date: 2018-10-07T10:50:48+10:00
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
  - "Rust"
---

The Cone compiler performs a data flow analysis pass after
[name resolution](/post/the-ir-tree-named-nodes) 
and [type checking](/post/the-ir-tree-typed-nodes).
Given that this sort of analysis is rarely covered by compiler literature,
I thought it might be useful to jot down some thoughts
about its purpose and intriguing mechanics.

## Goals

Like Rust (and unlike C), Cone applies constraints to references that ensure
they can only access memory safely, even in the face of concurrency.
Some of these constraints are completely enforced by the compiler.
Others involve a mix of compile-time and run-time logic.

The purpose of the data flow pass is to:

- Drop and free (or de-alias) variables at the end of their declared scope.
- Allow unique references to (conditionally) "escape" their current scope,
  thereby delaying when to drop and free/de-alias them.
- Track when copies (aliases) are made of a reference
- Ensure that lifetime-constrained borrowed references always outlive their containers.
- Deactivate variable bindings as a result of "move" semantics or
  for the lifetime of their borrowed references.
- Enforce reference (and variable) mutability and aliasing permissions
- Track whether every variable has been initialized and used

Although data flow analysis mostly focuses on references, it also works with
local variables that make use of a finalizer or move semantics.

## Function- and scope-based data flows

Data flow analysis requires complete and accurate type information,
which is why it happens after type checking.
It requires a separate pass from type checking largely because
type checking traverses the IR tree in a bottoms-up manner (since type inference flows
from the typed "leaves" upwards).
By contrast, data flow analysis is a top-down process.
It essentially traces the flow of references and other values from the beginning of a function 
to its end of its logic, navigating through all its nested scopes in between.

The Cone compiler begins data flow analysis for a function immediately after
type checking it. As this analysis winds its way into and out of lexical blocks,
it examines, in execution order, every place where references are 
created, copied, moved, used or destroyed,
enforcing key constraints and even generating additional runtime logic along the way.
Cone performs this traversal on (mostly) the same IR traversed during type checking
(as opposed to lowering the IR to a control-flow graph as Rust does with its MIR).

Several data flow mechanisms are easy to perform as the IR tree is traversed.
For example:

- **Permissions.** Whenever the function obtains data from a reference,
  the reference must have a permission that permits "read".
  Similarly, altering the information a reference points to
  requires some sort of mutable permission.
  Making a non-borrowed copy of a reference (aliasing) may only happen
  if the permission allows it. Other permissions are similarly enforced.
- **Lifetimes.** The compiler records the block scope whenever a borrowed reference is created.
  This scope is represented as integer that increments with each new level of block indentation.
  In most cases, we know it is unsafe to store a borrowed reference in some container
  if the scope number of the container is less than the scope number of the reference,
  as that means the reference's lifetime is shorter than how long we might keep it around.
  (Note: the proper handling of lifetime annotations requires a slightly more complex algorithm.)
- **Variable de-activation.** Creating a borrowed reference sometimes requires that
  the variable it acquired the reference from be de-activated from use for the lifetime
  of the borrowed reference. This is accomplished by setting a flag on the originating variable,
  which is later turned off when the borrowed reference has expired.
  This is only necessary when the variable's permission insists there be only
  one usable reference active at a time.
- **Aliasing.** A copy is made of a reference when it is passed as a function argument or
  assigned to some other storage location. When a reference is ref-counted,
  such aliasing automatically generates code which increments the reference counter.
- **De-aliasing** happens whenever a reference hits the end of its lifetime.
  This mostly happens when the scope ends for any variable holding the reference.
  It might also happen when a stored reference is overwritten by another reference
  or when an expression returns a reference that no one does anything with.

## Automated destruction

De-aliasing is where things start to get more interesting.
De-aliasing can sometimes be a trigger to automatically free memory
the reference points to. For example, de-aliasing a ref-counted reference
generates code that decrements its reference counter.
When that counter hits zero, it is time to trigger the free.

Flow analysis adds information to various IR nodes
to ensure the LLVM IR generation pass
generates the correct code. Sometimes this is as simple as setting some flags in a node.
In other cases, more information is required. 
For example the last statement in a program block may need to also 
list all variables in its scope that need to be properly de-aliased.

If de-aliasing does trigger a free, the generator may need to generate code
across several steps before the actual memory free may be performed:

- **Drop method**. If the variable's type implements a drop method (finalizer), it will be called.
  A finalizer typically unsubscribes itself or closes/returns any acquired resources.
- **Substructure dealiasing**. If the reference points to a struct, 
  the struct may have one or more fields
  that successively need to be dealiased and dropped.
- **Free**. The final step frees the reference's allocated memory.

The generator pass must precisely time when de-aliasing happens.
The return statement, for example, requires that any block-ending de-aliasing
logic happens *after* the return expression's value is calculated,
but *before* the return is performed.

As one final wrinkle, sometimes there can be multiple control flow exits from a block
or many returns from a function (including exceptions).
Flow analysis needs to ensure all paths end with the appropriate de-aliasing clean up.

## Escape analysis

Cone distinguishes between values that may be copied and those that may not.
Non-aliasable references may not be copied,
for example, nor may any struct that contains a non-aliasable reference.

If a non-copyable value is assigned to another variable, passed to a function,
or returned from a function, such an action actually *moves* the value from one
scope to another scope.
In those cases, we say that these non-copy values "escape" their original scope.
Indeed, such values can hop from scope to scope as much as a program wishes.

Flow analysis must track all such escapes to avoid performing de-aliasing
activities on these references at the end of the scope they have escaped from.

And that's not all!


Where escape analysis gets particularly complicated is when escapes are
conditional. In other words, it is possible for a function's logic to allow
a non-copy value to escape under some conditions, but not others.
When the determination of whether to escape happens at runtime,
the compiler cannot generate definitive logic.
In this situation, the variable is marked with a flag that indicates
it conditionally escapes. The generator can use this flag to generate
code that sets a runtime flag indicating whether the value has escaped.
This flag can then be tested at runtime prior to any generated de-aliasing logic.

As you see, the Cone compiler's data flow analysis pass really does have a lot of meaningful
work to do.