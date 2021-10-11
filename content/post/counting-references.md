---
title: "The Challenge of Counting References"
date: 2018-10-13T10:37:40+10:00
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

*Note: The latter part of this post is outdated in terms of the mechanisms that 
the Cone compiler uses to de-alias ref-counted references.
See [this post](/post/data-flow-analysis) for an updated description.*

In the world of automatic memory management,
[reference counting](https://en.wikipedia.org/wiki/Automatic_Reference_Counting)
is considered to be one of the easiest to implement.
The rules seem simple:

- When a reference is created to an allocated memory area, set its counter to 1
- When the reference is copied (aliased), increment the counter
- When an alias is destroyed (de-aliased), decrement the counter
- When the counter reaches zero, free the reference's memory area

The simplicity of these rules does not always translate to a simple implementation.
Let's explore how complicated it can get when applied to
[Cone](https://github.com/jondgoodwin/cone).

## How Rust does it

If a language already supports generics and robust move semantics (neither being easy),
implementing a reference counting capability can be straightforward.
This is how Rust implements [Rc\<T\>](https://doc.rust-lang.org/std/rc/index.html):

- Aliasing must be explicitly requested using Rc::clone() on an existing reference.
  clone() increments the counter.
- De-aliasing leverages Rust's single owner "move" semantics:
  At the end of a reference's last scope, its "drop" logic is invoked.
  This decrements the counter and handles when the counter reaches 0.
  
There is an interesting trade-off in Rust's approach: programmer convenience vs. throughput.
The programmer must remember to explicitly create each alias whenever needed.
However, the ability to move an alias around minimizes the number of times
a counter is incremented and decremented, thereby improving performance.

## How C++ does it

The design approach taken by C++, Python, and Swift focuses more on programmer convenience.
References are automatically (and implicitly) aliased and de-aliased.

Similar to Rust, C++'s `shared_ptr` takes advantage of its template and move semantic features,
but does so in a different way.

- Function calls invoke the shared_ptr copy constructor, ensuring aliasing takes place.
- Assignment triggers the shared_ptr assignment, which aliases the rval and de-aliases the lval.
- RAII ensures that unmoved shared_ptr's are de-aliased.

Although this approach is simple, it is insufficient for Cone's needs.
It *can* leak memory and several useful language
features, tuples and blocks/if as expressions, are not supported.

## Counting references in Cone

A key design goal for Cone is to offer a broad range of interoperable,
safe memory management strategies.
Automatic aliasing is particularly important,
because we want reference use to be as consistent as possible,
regardless of which memory management strategy originally allocated each reference.

For Cone, let's start with the basic rules from C++:

- Aliasing happens automatically whenever a reference is passed to another function.
- Aliasing happens when a reference is stored into another variable
  (or really any lval, such as a dereference, struct field, array element, etc.).
  The lval's previous reference must also be de-aliased.
- De-aliasing occurs automatically
  whenever a variable holding a reference goes out of scope.

Let's explore where these rules fall short...

## Unbound references

The potential for memory leaks arises from differences between
*unbound* and *stored* references.
A **stored** reference is one that is stored somewhere in memory
addressable by the program (e.g., in a variable, structure, or array).
An **unbound** reference is not (yet) stored.
It is a pointer in transition, held in a register or on the stack as a temporary value.

In Cone, a new ref-counted reference can be easily created in an expression
using the ampersand operator.
For example: `&rc 4` allocates new memory that holds the integer 4.
This expression returns an *unbound* reference to that newly allocated memory.

If we want to count references correctly, we must handle unbound and stored references differently.
Consider this snippet which declares two immutable variables:

    imm ref1 = &rc 5   // Allocate a new ref-counted reference
	imm ref2 = ref1    // Copy that reference (the counter is now 2)

This shows we must refine our rule about de-aliasing assignment lvals:
Since ref1 and ref2 are just being declared, they don't yet hold a reference that
needs to be de-aliased.
The same principle also applies when assigning a reference
to a variable that has been declared but not initialized.
It would also apply to nullable references:
if the lval's reference is null, don't de-alias it.

The snippet also demonstrates that we must treat unbound and stored references differently.
In line 2 we alias `ref1`, but in line 1 we *don't* alias `&rc 5`.
Of course we shouldn't, as a just-allocated reference already has the correct count (1).
Clearly, our automatic aliasing logic has to be sensitive to the source of the reference
when deciding whether to increment its counter.
We need to update our rules to not alias an *unbound reference*
on assignment or when passed to a function.

As it turns out, improper handling of unbound references can also cause memory leaks.
Imagine a program with the following expression-as-a-statement:

    &rc 10

Here we allocate a new reference that is never stored anywhere.
A reference that is not preserved will never be used or freed.
If we don't want memory leaks, we must add a new rule that ensures
unbound references evaluated from an expression statement are automatically de-aliased
if they never transition to stored references.

It gets worse.

## Returned references

How should we handle references returned by functions? For example:

	fn condref(cond i32) &rc i32
	  if cond==1
	    imm ref = &rc 10       // counter = 1
		return ref             // ref de-aliases counter to 0?
	  else
	    return &rc 5           // counter = 1
	fn main()
	  imm mainref = condref(2) // Do we alias counter to 2?

As this example illustrates, returns unlock new complications:

- The scope-ending de-aliasing rule means `ref`'s count goes to 0, which will free it.
  This means we will return a reference to a value that no longer exists. Not good.
- If we return an unbound reference, we should *not* alias it on assignment to `mainref`.
  That would bring the count up to 2, one more than the number of aliases that now exist.

Since functions are opaque to their caller, their behavior must be consistent.
So, we must decide whether returned references should already be aliased or not.
It does not take much thought to realize they should be already aliased.
This choice means the caller should treat references returned from a function call
as unbound references.
Treating function returns as unbound references ensures that the mainref assignment
won't alias the reference it receives from `condref`.

According to the improved rule, the return of `&rc 5` is correct as is.
However, the return of `ref` is not, since even without the problematic free,
the count value would be one less than expected.
To fix this problem, we must now modify our scope-ending de-aliasing rule
so that it does not de-alias any variable listed on the return statement.
Doing that ensures that `ref`'s count stays at 1, the correct value.

Our code example does not show returned stored references extracted
from a struct field, array element, or a dereference.
How should these be handled?
To ensure a correct count, they need to be aliased as part of the return logic.

By stepping back and looking at the rule changes we have made,
we can summarize them by saying that returning a reference from a function
should be aliased just like passing a reference to a function
(alias stored references but not unbound references)
-- excepting only when returning a local variable.
This exception is effectively an optimization that realizes
that an alias followed by a scope-ending de-alias cancels each other out.

It gets worse.

## Blocks-as-expressions

Like Rust, Cone treats blocks as expressions.
The value of a block is the value of its last statement-expression.
This applies to an `if` structure as well:
its value is the last statement-expression in whichever branch is selected.

How should we handle references emitted from a block or if?

One approach is to apply the same rules as with return values.
This will keep our reference counting implementation from getting any more complicated.
However, there is a potential performance cost to having a block/if alias a reference,
only to have it be immediately de-aliased should the reference be discarded.

We can avoid this performance penalty through some extra data flow analysis.
This analysis would examine whether a block's reference value will be used or discarded.
If the reference will be used, it ensures it is properly aliased.
If the reference will be discarded, it ensures unbound references are de-aliased. 

It gets worse.

## Tuples

Cone's tuples make possible parallel assignment and multiple return values.
These are useful features, worth the added compiler complexity.

Nonetheless, tuples do in fact complicate reference counting.
This complexity is not so bad when returning multiple references.
Following precedent, the returning function must ensure they are all properly aliased,
and the caller handles that accordingly.

Parallel assignment is a bit trickier. Consider:

    _, x = &rc 4, ref2, ref3

Since `ref2` is being stored in x, it needs to be aliased (since it is not an unbound reference).
However, `&rc 4` and `ref3` are being discarded to the bit-bucket in the sky.
This means the unbound reference `&rc 4` must be de-aliased and `ref3` should be left alone.
Handling this correctly requires more data flow analysis and IR node flags.

That's not going to be enough, unfortunately, as assignments are also expressions.
Consider this disruptive example (and there are worse ones I could show):

    return (_, x = &rc 4, ref2, ref3)

The complicated approach to data flow analysis would involve aliasing and de-aliasing
on the fly as you unwrap each part of the onion.
The onion unwrapping approach gets potentially even more complicated when dealing with chained
incomplete assignments.
Another non-trivial approach would be to translate tuples into hidden structs,
which gets the job done correctly, but benefits greatly from aliasing optimizations.
The simplest approach might be to throw up your hands and just adopt return's
requirement that all references be aliased before being thrown over the wall.

Which approach would you choose?

## Enough!

There's more we could explore (e.g., how to handle weak pointers),
but let's stop here.
My point should be obvious: implementing reference counting correctly and automatically
can get a lot more complicated than it first appears.

Solving reference counting is a vital part of my work on Cone's 
new [data flow pass](/post/data-flow-analysis).
I actually do have a solid design that addresses all these requirements in a (mostly)
straightforward way.
It does not require any sort of template/generic feature nor does it rely on move semantics.
It ensures memory safety, even helping with other non-ref-counted allocated references.

The planned approach takes advantage of what I call the alias stack.
While traversing expression nodes during data flow analysis,
I start with an alias counter of 0 (for a block expression statement) or 1 (for passing arguments
or returned values). Every time I run across a non-underscore lval in an assignment node,
the counter is incremented.
The rval is the final stop; if it is an unbound reference, I decrement the counter.
The final count indicates whether the rval is to be aliased (>0) or dealiased (-1),
signalled to the code generator via an injected IR node.
Handling multiple such counters as part of a stack makes it mostly straightforward to support
tuples and if/block as expressions (even across multiple branches).

Although correctly counting references turned out to be more complicated than I expected,
it is still much easier than implementing a modern, performant tracing garbage collection.
A tracing GC is also part of Cone's future.
However, we'll save its challenges for another day!