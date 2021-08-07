---
title: "Data Flow Analysis"
date: 2021-08-07T10:50:48+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Intermediate Representation"
  - "Memory Management"
tags:
  - "Cone"
  - "Rust"
  - "Cyclone"
---

*Note: This is a heavily revised version of an [earlier post](/post/data-flow-analysis-old)*

The Cone compiler performs a data flow analysis pass after
[name resolution](/post/the-ir-tree-named-nodes) 
and [type checking](/post/the-ir-tree-typed-nodes).
Given that this sort of analysis is rarely covered by compiler literature,
I thought it might be useful to jot down some thoughts
about its purpose and intriguing mechanics.

**Trigger Warning:**  This blog post is highly technical and brief.
It reads more like an organizing outline for a design spec than a typical
essay-oriented post.

## Goals

Like Rust and Cyclone (and unlike C), Cone constrains the use of variables and values
in the pursuit of type, memory and concurrency safety.
In pursuit of optimal performance, most constraints are enforced entirely by the compiler.
However, some do make use of a mix of compile-time and run-time mechanisms.

Which constraints (and capabilities) does Cone facilitate during data flow analysis?

- **Initialization before use**. Ensure every variable has a valid value before it can be used.
- **[Borrowed references](/post/reference-lifecycle/#borrowed-reference)**. 
  Ensure the source of borrowed references cannot be used within the lifetime of the borrowed reference.
  Check for region liveness or permission lock when borrowing from a weak or owning reference.
- **[Lifetimes](/post/lifetimes)**. Ensure that lifetime-constrained values (particularly borrowed references) never survive past their scope.
- **[Move semantics](/post/move-mechanics/)**. 
  Make variable bindings unusable when non-copyable values are moved out of them, and reactivate their
  use when they once again are assigned a value.
- **[Region Aliasing, De-aliasing and Existence](/post/reference-lifecycle/)**. 
  - Invoke region aliasing logic when copies are made of owning references
  - Invoke region de-aliasing logic when a non-escaping alias of an owning resource expires in its scope,
  - Invoke region existence logic to determine whether a weak reference still points to its original object
- **[Permissions](/post/race-safe-strategies/)**. Enforce reference permissions on whether the pointed-at value can be read or changed
  and whether a reference can be aliased. Also, invoke lock-based permission logic during borrowing.
- **Drop logic**. Invoke drop and region free logic, automatically or explicitly, 
  when the last owning reference to a resource is no longer usable.

Although most of these mechanics are needed to ensure pointers are used safely,
enforcement ends up bleeding into values of complex record types that have references as elements, 
as well as those that make use of a finalizer or move semantics.

## Context-specific Expression Tree Analysis

Data flow analysis is performed on each function's logic nodes immediately after its nodes have been typed check,
since analysis requires type information to make decisions.
AST/IR nodes are not lowered to a control-flow graph, as 
the already simple, block-structured representation of a function's control flow is easy enough to follow (more on that later).

Data flow analysis simply traverses the function's logic nodes in order of execution, visiting each node at least once.
Each node is viewed through the lens of how its expression value(s) are to be used.
There are four such contexts that are needed to enforce the behavior described earlier:

- Loading a value.
- Copying/moving a value.
- Storing a value
- Discarding a value.

### Loading a value ###

Many expression nodes compute new values based on values they are given. It is the leaf nodes of such expression trees
that provide the source values for the expression. A source node is typically a literal or a variable.
For data flow analysis, literals can be ignored. It is an expression's variables that need to be analyzed.

In particular, we want to statically ensure that a variable holds (or points to) a valid value we are allowed to read.
This means performing all these checks on the variable node:

- Does the variable not yet have some initial value?
- Has the variable's value been moved out using move semantics?
- Is the variable temporarily deactivated from use, because some reference borrowed from it is still alive?
- Do the variable's (or reference's) permissions forbid us from reading its stored/pointed-at value?

If any of the answers is yes, we have a broken data flow constraint. This generates a compiler error.

These checks depend on every variable beginning with an initial state that may change as a result of subsequent expressions
(as described below). For example, all global variables and non-`out` function parameters are treated as being initialized.
However, a function's local variables are considered initialized only when they are given a value by the function's logic.

**Deref Node.** Because of viewpoint adaptation, we don't just examine the permission of the source variable node.
We also need to examine each deref node's behavior:

- Does the reference's permission permit the value to be read. The answer is no to all runtime permissions,
  as these must be first unlocked by borrowing a reference to the value.
- If it is a weak reference whose region annotation defines deref behavior, that logic
  will be generated in place of the default pointer deref behavior.

**Borrow Node.** One other node that is examined in this context is the borrow node.
This node creates a borrowed reference to some data structure.

- We need to mark that the source variable, used to create the borrowed reference, is marked as borrowed-from
  (and therefore unavailable for use so long as the borrowed reference exists)
- If the source is an owning reference with a lock permission, we must generate the logic for obtaining the lock
  (and then free it later when the borrow expires).

Nearly all expression nodes in a function are evaluated by this "loading a value" context in top-down order.
Evaluating nodes in the other three contexts happens as nodes are visited by this process, right after performing
the "loading a value" constraint check.

### Copying/moving a value ###

Copying or moving some existing value is only performed by one of these nodes:

- Function call:  each argument
- Variable declaration: initialization value
- Assignment to an lval: rvalues
- Allocation: initial value
- Return/break/blockret

Whether the expression's value is copied or moved is determined by whether the expression's type
subscribes to move or copy semantics.

If the expression's value is being moved, we may need to deactivate the source of that value,
when that source was an variable embedded in an lval expression.
Effectively, we signal this by modifying the variable node to say the value has been moved out of it.
We do not allow a move out of a global value.

Most of the time, when an expression's value is copied, we don't care. 
However, we do care when the value being copied is (or contains) an owning reference whose region
has defined behavior for aliasing the reference. An aliasing node is injected in this case.
For tuple or struct values, this may recursively generate aliasing logic for multiple references.

### Storing a value ###

For an assignment node (to an lval), we must do more than the move/copy analysis just described.
We must also ensure it is valid to store the value in the lval.

- If the lval is a variable that has not yet been initialized, no further analysis work is needed.
- if the lval is a variable that has had its value moved out, the variable must be mutable.
- If the lval has a source variable that has an active borrow, we terminate the borrow (if possible).
- We ensure the lval is fully mutable (via view adaptation).
- If the lval contains/points-to one or more owning reference whose region has de-referencing behavior,
  we wrap it in a dealiasing node.
- If the lifetime of the lval lasts longer than the lifetime of the value, this is an error.

### Discarding a value ###

The following nodes discard a value which might be, or might have, an owning reference:

- A block's expression statement, where we essentially are going to throw away all values
- A de-reference of an owning reference that is not sourced by a variable
- Assignment of an rval to `_`
- Applying drop to an owning reference

In all these situations, we wrap the expression for the value in a dealias node.

## Scope and Control Flow Considerations ##

The above rules are largely fine for a simple function. 
However, additional rules are needed when we take into account multi-layered block scopes, loops,
and conditional flow.

### Block Scope ###

At the end of every block, we need to de-alias any values still held by the block's local variables:

- Any uninitialized or moved variable is ignored. Moving a value is how it can safely escape its scope.
- Any variable that is still borrowed against, has its borrow terminated. 
  If the borrow involved acquiring a lock from a lock permission, code is generated to release the lock
- Region de-alias and/or drop logic is applied to the values in all variables with valid values,
  depending on whether the value is copy (de-alias) or move (drop).
  The last statement in a block is updated to
  list all variables in its scope that need to be de-aliased or dropped.
  
De-aliasing can sometimes be a trigger to automatically free memory
the reference points to. For example, de-aliasing a ref-counted reference
generates code that decrements its reference counter.
When that counter hits zero, it is time to trigger the free.

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

### Looping blocks ###

Moves can be dangerous inside a looping block, when applied to local variables outside the scope of the looping block.
If we have moved the value out of the variable on the first iteration of the loop,
it is not there to move out again on subsequent iterations.
Therefore, such moves are forbidden and generate a compiler error.

The tricky part of this mechanism is distinguish which logic is inside the loop
and which can be considered essentially outside the loop, since we know it is an exit path only performed once.
In exit paths, the move can be allowed.

### Conditional Data Flow Analysis ###

Where move/escape analysis gets particularly complicated is when escapes are
conditional. In other words, it is possible for a function's logic to allow
a non-copy value to escape under some conditions, but not others.
When the determination of whether to escape happens at runtime,
the compiler cannot generate definitive logic.
In this situation, the variable is marked with a flag that indicates
it conditionally escapes. The generator can use this flag to generate
code that sets a runtime flag indicating whether the value has escaped.
This flag can then be tested at runtime prior to any generated de-aliasing logic.

