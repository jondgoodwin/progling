---
title: "Move Mechanics"
date: 2018-10-22T21:28:32+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory Management"
tags:
  - "Cone"
  - "Rust"
  - "C++"
---

Most programming languages support only copy semantics.
A value passed to a function or stored in a variable
is a copy of the original value.
We know it is a copy, because any change we make to the copy has no
impact on the original value.

A few languages, like C++ and Rust, also support move semantics.
Unlike a copy, a transfer moves the original value to its new home;
that value is no longer accessible at its previous home.
Move semantics make possible some interesting safety-based capabilities,
such as the fast single-owner memory management strategy,
the ability to safely send mutable data between threads without the need for locks,
and a way to isolate and enforce dependency invariants in objects.

Cone supports both copy and move semantics.
This post summarizes Cone's approach to move semantics:
its decision criteria for choosing between copy vs. move
as well as the detailed mechanics the compiler uses to ensure
moves are correctly handled.

## Contrasting Cone's approach to C++ and Rust

C++ introduced move semantics as part of C++11 via such features as:
rvalue references, unique_ptr, std::move, move constructors and move assignment operator.
to the "rule of five".
C++ backward compatibility constraints weakened the committee's design options,
resulting in move mechanics that are neither simple nor bullet-proof.
Adopting move semantics also expanded the intriguing
["rule of three"](https://en.wikipedia.org/wiki/Rule_of_three_(C%2B%2B_programming))

By contrast, Rust's approach is easier-to-use, more concise, and more thoroughly enforced.
Indeed, move semantics are a central design feature for the Rust language,
leveraging safety through the effective application of
[linear/affine type theory](https://en.wikipedia.org/wiki/Substructural_type_system).

With Rust, it is a value's type that determines whether it is copied or moved.
By default, a newly-defined type makes use of move semantics.
However, if the type implements the 'Copy' trait, its values will use copy semantics instead.

Cone's move semantics are very similar to Rust's.
Move vs. copy are determined by the type.
Move semantics are thoroughly enforced by the compiler using similar rules.
The biggest difference lies in how Cone determines whether a
type uses copy vs. move semantics.

## Cone: Copy vs. Move Types

If a type's values can be copied safely, it uses copy semantics.
Otherwise, it uses move semantics.

When would a type's values not be safe to copy?

- References that use the `&own` allocator or the `uni` permission.
  `&own` guarantees that there is only one owner.
  `uni` guarantees that this is the only "owning" reference to a value.
  Being able to copy such references would break their single alias guarantee.

- Compound types (like structs, arrays, or variant types)
  that make use of *any* non-copyable type as a field/element.
  Non-copyability is infectious in this way;
  if you cannot copy part of a value, you cannot copy the whole value.

- Any type that implements a finalizer.
  The existence of a finalizer typically means the type
  expects its values to entangle dependencies with other system objects.
  Such dependencies would be unlikely to transition correctly
  via a shallow copy.
  This principle is reminiscent of the C++ rule of three/five.

Cone automatically detects these conditions in a type and, if found,
uses move (rather than copy) semantics for its values.

The only exception to this rule is when such a type implements the 'clone' method.
The compiler *always* uses the 'clone' method (when implemented)
to copy all values of that type.
Since the 'clone' method provides a safe way to make copies of the type's values,
the Cone compiler chooses copy semantics instead.

## Enforcing move semantics

Unlike copying, handling move-only values introduces complexity for the compiler.
Part of this complexity arises from the fact that a "move" likely performs
a shallow copy under the covers.
What makes this move-triggered copy safe is that the compiler prevents access to the original value
at its old location:

    imm ref = &own 12   // Allocate a new single-owner reference to '12'
    imm ref2 = ref      // Move the reference to ref2
    print(*ref)         // **ERROR**: ref is no longer usable as its value moved out!

In this case, enforcing this access restriction is easily accomplished.
During the Cone compiler's [data flow analysis pass](/post/data-flow-analysis/),
a flag is set on the 'ref' variable's declaration node to indicate its value has been moved out.
Any attempt to use the variable after that point notices the set flag and generates
a compiler error message.

As you might expect, prohibiting access to the value's original home
is not always this easy (or even possible).

## Field-based moves

How should we handle when a struct's field holds a move-only value?
Moving a value into that field poses no safety challenges.
However, moving a value out of the field does.
The struct is now "missing" a value in one of its fields, making it unsafe to use.

A simplistic solution would invalidate the whole struct if any of its fields has moved out.
Unfortunately, that is too strict, as we may need to extract other fields
before we throw the whole struct out.

A better approach is to deactivate access to the whole struct, while still allowing
other fields to be accessed or extracted.
This would need to ensure each move-only field is extracted only once
(and then never accessible after that).
In order to keep this enforcement tracking from getting complicated to implement,
one could restrict all such extraction to a single lexical scope
and to only one level down into a struct's fields.
Violation of these rules would generate a compiler error.

## Illegal moves

Unfortunately, there are some cases where it is simply not feasible
to track and deactivate the original holder of a moved value.
Where such tracking and deactivation is not feasible,
any attempt to move out a value must fail with a compiler error.

Here are situations where moves are illegal:

- **Arrays**. One cannot extract a move value held by an array element.
  In order to make this safe, one would need a complex run-time
  mechanism that notices and panics on any attempt to extract a value
  that has already been moved out at any arbitrary index.

- **Moves inside loops**.
  Cone does not allow moves within a loop
  from variables whose scope/lifetime is larger than the loop.
  This is because a loop cannot safely re-move a value already moved out on a prior iteration.

These constraints are not necessarily as restrictive as they may first appear.
There are often effective work-arounds for most such situations.
One of the more effective workarounds is to use Cone's 'swap' statement.
Swap makes it possible to safely extract a move-only value from anywhere,
while still ensuring the original home is holding a safe (if not useful) value.

## Scope-surfing moves

A compiler's move mechanics has two essential enforcement responsibilities:

- Deactivate access to the moved value's original home (as we have discussed)
- Determine when to destroy a movable object

One of C++'s valuable innovations is the strangely-named
[Resource Acquisition is Initialization](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
(RAII).
A common pattern for RAII use ensures that allocated memory (a "resource")
is automatically and safely destroyed at the end of the block scope it is acquired within.

Moves complicate scope-based destruction in exciting ways,
since moves allow a value to safely escape its current scope.
Indeed, one of the fun aspects of movable values is watching them surf gracefully
in and out of multiple scopes.
They die only when the music stops and they have not found a new scope chair to sit in.

To support this behavior, a compiler requires escape analysis as part of its move mechanics.
In many cases, this can be handled easily.
Remember that flag we set on a variable whose value moved out?
We can use that flag not only to prevent further access to the variable,
but also to ensure that its value must not be destroyed by a scope-ending RAII mechanism.
This simple compile-time test ensures a movable value is only destroyed
when it has not hopped out of the scope.

## Conditional moves

Unsurprisingly, escape analysis is not always that easy. Are you surprised?

Escape analysis is complicated by conditional logic at runtime.
When a move happens within a conditional block, the compiler has no way
to decide if the value escapes the scope or not.

Don't despair. All is not lost!
Even though the compiler cannot know for sure whether or not the value moves,
it is nonetheless able to generate run-time logic that handles this situation safely and gracefully.

The compiler marks such a variable's node with a conditional-move flag
indicating that it may or may not move.
The generator uses this flag to generate additional runtime logic for this variable:

- Space is reserved in the block scope for a runtime flag that indicates whether
  the variable's value has been moved. It is initialized to 'not moved'.
- Whenever the variable's value is moved, extra logic is generated that changes
  the variable's move flag to 'moved'.
- At the end of the block's scope, the variable's move flag is consulted.
  Only if the flag shows 'not moved' is the variable's value destroyed.

The astute reader will now wonder if conditional moves also complicate
the mechanisms for deactivating use of variable after its value conditionally moves out?
Yes, of course it does.
When the compiler ever becomes unsure, in a given scope, whether a value has moved out
or not, subsequent use produces a compiler error indicating that the variable cannot be used
due to the potential move.
