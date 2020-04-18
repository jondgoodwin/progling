---
title: "Reference Type Semantics"
date: 2020-03-09T18:19:19+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Types"
tags:
  - "Cone"
---

The [prior post](/post/searching-for-quarks/) described the semantics
underlying Cone's fundamental types, for all types
except the all-important pointer and reference types.
The presence of versatile pointers/references 
as an explicit and distinctly-separate kind of type is
a distinguishing characteristic of systems programming languages.
I choose to cover them here as a separate post,
since the safety guarantees of Cone's references make them semantically rich and complex.

In harmony with the subatomic theme of the prior post, 
reference types are themselves composed of "quarks": 
pointers, regions, permissions, and lifetimes.
I will start off with these,
before then showing how they can be composed together into
the different flavors of references that Cone supports.

## Pointer Types ##

Cone's pointer types work very similarly to C's pointer types.
A pointer is essentially an unsigned integer index (address) into some linear address space of memory bytes.
The pointer's address size depends on the target CPU architecture (typically 32-bit or 64-bit).
Dereferencing a pointer allows one to access or update the value in linear memory
at the pointer's address.

A pointer type specifies the type of the value it points to (e.g., `*f64`).
Pointer types are structural, such that two pointer types with the same structure
are treated as equivalent.

Internally, the compiler implements pointer types as if they were a generic type
over the specified value type.
Cone's pointer types offer built-in methods for performing various operations on pointers.
These methods support not only dereferencing, 
but also comparison and various forms of pointer arithmetic.
Pointer arithmetic is automatically scaled by the size of the type it points to,
so that an f64 pointer incremented by one actually adds 8 to the pointer's address.

Since pointers impose no constraints on their use, safety is not guaranteed.
Pointers are vulnerable to a range of memory or data race issues.

## Reference Safety Annotations ##

Cone's references are the safe approach to memory access.
References provide a similar capability to pointers, 
but impose constraints that ensure they can
only be used in safe ways. These constraints take the form of
annotations added to all reference types: regions, lifetimes, and permissions.
Think of them as subatomic quarks we can later use to build references.

Regions and lifetimes ensure memory safety.
Permissions ensure data race safety.

### Regions ###

A [region](http://cone.jondgoodwin.com/coneref/refregionglo.html)
defines a logical collection of memory addresses
managed by the same memory manager.
Every new data object is allocated by some specified region,
returning a reference whose type indicates its region.
When the last reference to this object is gone,
the region automatically frees the object and recycles its memory.
Different regions support different memory management strategies,
including single-owner, reference counting, tracing garbage collection,
arenas and pools.

A region's capability comes in two parts: the region itself and the region annotations.
The region constitutes the memory area allocated
either directly from the operating system, or via another region
(i.e., first-class dynamic regions). 
Sometimes, a region also consists of global runtime logic and state data
responsible for managing the use of the region's memory.

The region annotations signify the data and logic that accompanies
every object reference managed by the region.
The region annotation is effectively a special-purpose trait which defines:

- The bookkeeping **fields** used by the memory manager for each allocated object.
  For example, a ref-counted region would use a field to track the count
  of how many references point to the object.
  Some regions (e.g., single owner) do not require any bookkeeping fields.
  
- Region-specific **methods**. All regions specify
  methods for allocation, aliasing, de-aliasing and freeing an object.
  Additional methods might also be handy, such as a `weak` method
  for creating a weak reference alias.

- Intrinsic region **attributes**. These attributes signal to the compiler
  the need for special handling of this region's references.
  For example, single-owner region references should have move semantics,
  so that they cannot be aliased. Likewise, regions that are marked as `traced`
  ensure that tracing maps are automatically created for all affected types.

By making the region annotation into a trait, we accomplish two things.
Since traits cannot be instantiated, neither can a region annotation.
Later, we will see how treating the region annotation as a base trait 
can be used to provide an envelope around some memory-managed object.
 
### Lifetimes ###

Borrowed references do not specify a region, as they are just temporary aliases
that are polymorphic no matter how the reference is created or destroyed.
Given the lack of region annotations on borrowed references,
we need some other constraint to ensure memory safety. 
That constraint is a [lifetime](/post/lifetimes/).

A lifetime is a compile-time value attached to some type
(e.g., a borrowed reference or any type containing a borrowed reference).
This value encodes the scope block beyond which any value of that type may not outlive.
The compiler captures and calculates lifetime information
independently on a function-by-function basis.
This lifetime information consists of:

- The **variable** that is the source of the lifetime, when the lifetime
  refers to a block scope within the function.
  The compiler ensures that this variable becomes inaccessible for use,
  so long as the lifetime-constrained value is still alive.
- The **invariance group** of the lifetime (unsigned integer).
  An invariance group of 0 is used for the global lifetime,
  values that are never lifetime constrained. An invariance group of
  255 represent all lifetimes whose block scope lies within the function.
  All other values in between represent lifetimes whose block scope
  lie within one of the function's callers. 
- The **relative scope** of the lifetime (unsigned integer)
  within its invariance group. For function-local lifetimes,
  the function's outer-most block has a scope value of 1.
  This increments by one for each inner block scope.

Lifetime values represent an unusual form of partial order:
The global invariance group has the largest lifetime, 
the function-local invariance group has the smallest lifetime
(and can further be compared using the relative scope number),
and the non-local invariance groups in between can only be compared when the group matches.

This partial order makes it possible to compare any two lifetimes,
much like any subtyping relationship between two types.
A lifetime comparison yields one of four outcomes, they are equal,
one is greater than the other, or no valid comparison is possible.

This number-based lifetime encoding scheme is sufficient not
only for handling borrowed references safely, it also serves
well for invariant lifetimes that can be used to guarantee the
correct pairing between a first-class region and any references
dynamically allocated out of that region.

### Permissions ###

Reference [permissions](/post/race-safe-strategies/) are used to prevent data races.
In a multi-threaded environment, they prevent the ability of
a mutable reference in one thread from being able to change
the value it points to at the same time that a reference 
in another thread is performing some atomic transaction with that value.
In a single-threaded, they prevent the ability to change 
the dependently-typed shape of some value while other inner references
exist that point to values within its original shape.

Like region annotations, permissions are special-purpose traits.
They may specify:

- Bookkeeping **fields** used by the runtime-based permission for an object.
  A lock-oriented permission might define an atomic value field
  used by atomic synchronization mechanisms to ensure only
  one thread has mutable access at a time.
  For static, lockless permissions, no fields are defined.
  
- Permission-specific **methods**. 
  For example, lock-based permissions might offer a `lock` method,
  which acquires the lock on the object and returns a borrowed method.
  An automatically invoked `unlock` method would then release the lock
  when the borrowed reference expires.

- Intrinsic permission **attributes**.
  Such attributes are used by the compiler to constrain whether
  the region's reference can be dereferenced, used for value mutation,
  or aliased to some other reference with possibly a different permission.

## Reference Types ##

Now that the region, lifetime and permission reference annotations
have been introduced, we can use them to compose various
flavors of reference types: regular, array and virtual,
as well as their borrowed reference shadows.

### Regular References ###

A [regular reference](http://cone.jondgoodwin.com/coneref/refallocref.html)
points to a single value of some type.
A regular reference's type signature specifies (in order)
its region, permission and the type of the value it points to.
For example, `&so mut i32` is a shared, mutable reference to an integer
which is managed by the single-owner region.

The reference's region and permission constraints act as both a compile-time and runtime 
envelope around the value the reference points to.
These constraints are a compile-time envelope because they guard the
enclosed value and ensure it is only handled in safe ways.

These constraints also act as a physical envelope at runtime,
because when an object of some type is allocated, 
the allocation automatically includes extra space for
any region or permission fields, which are also properly initialized.
It is as if the allocated object were a larger structure
composed of the region trait, the permission trait, and the value.
The reference points to this larger structure, giving it
statically-computable access to not only the value,
but also the region's and permission's runtime bookkeeeping fields.

Underneath Cone's ease-of-use features for references 
(syntactic sugar, defaults and inference),
we can thus formally model a reference type as a generic
type accepting three type parameters:
R(egion), P(ermission), and T (the type of the value Ref points to):

    typedef Ref[R, P, T] *struct {
	  mixin R
	  mixin P
	  value: T use *
	}

A regular reference thus has access to the region's and permission's
fields, as well as the pointed-at value itself.
Additionally, one can use the reference to invoke any of the methods
defined by the region trait, permission trait, or value type,
as well as the generic operators for regular references
(dereferencing, mutation and comparison).

As with pointer types, all reference types are essentially structural; 
two regular reference types match if R, P, and T are the same.

#### Borrowed References ####

A [borrowed reference](http://cone.jondgoodwin.com/coneref/refborref.html)
is effectively an aliased address to some value that
is also pointed-at by another region-managed (owning) reference,
which the compiler guarantees will last at least as long as the borrowed reference.
Being polymorphic across all regions, a borrowed reference specifies
no region annotation (you can think of it as having the top-most/any region).

In the place of the region annotation, the borrowed reference type
has an implicit/explicit lifetime attached to the type,
in addition to the permission trait and type of the pointed-at value.

    typedef Borref[L, P, T] L + * struct {
	  mixin P
	  value: T use *
	}
	
The lifetime is effectively a memory-safety constraint placed on the pointer.type,
with the addition operator acting like an type intersection operator
(i.e., it is a pointer and it is also lifetime-constrained).
Lifetime constraints may be implicitly or explicitly placed
on any other type, not just pointers.

### Array References ###

An array reference is subtly different than a regular reference,
in that it refers to some number of elements of type T.
The exact number of elements it refers to is only known at runtime.
An array reference is effectively a "fat pointer",
composing together a regular reference and the size information:

    struct ArrayRef[R, P, T] {
	  ptr: * struct {
	    mixin R
		mixin P
		val: T use *
	  }
	  len: usize
	}
	
The inheritance of fields
and methods mix together as described for regular references.
However, the core
methods for an array reference differ: comparison, indexing instead of de-referencing,
and methods for structuring and destructuring fat array references.

Array references are structural 
and support borrowed array references (slices) in a similar way as described above.

### Virtual References ###

Virtual references support virtual dispatch (subtype polymorphism)
and indirect field access (row polymorphism) for some collection of types
that share the same common supertype.

Like array references, a virtual reference is another kind of "fat pointer".
Instead of a length, it has a pointer to the supertype's vtable:

    struct VirtRef[R, P, T] {
	  ptr: * struct {
	    mixin R
		mixin P
		val: T use *
	  }
	  vtablep: *SuperTypeVtable
	}

The SuperTypeVtable is essentially an immutable global value
generated by the compiler for any specific supertype.
It consists of a list of function pointers
and field byte-offsets. The compiler knows the offsets of
these in the respective vtables, and uses this information to
generate native logic that performs virtual field access and method dispatch.

Virtual references are structural 
and support borrowed virtual references in a similar way as described above.

## Reflections on Infectious Properties ##

An earlier post highlighted how certain primitive type semantics
are [infectious](/post/infectious-typing/).
As it turns out, all three data type-based, infectious properties 
originate from the safety annotations placed on references:

- **Move semantics** arise from references that use the single-owner
  region or the `uni` permission.
- **Thread bound** restrictions arise from references whose permission
  allows lockless, shared, mutable access to a value (e.g., `mut` or `const`).
- **Lifetime** restrictions arise from borrowed references bearing lifetime constraints.

Any type (e.g., a struct) composed out of references that bear these properties will be similarly constrained.
Thus, these fundamental type properties are not just reference semantics.
They universally apply to nearly all the other types described in the previous post.

## Reflections on Type Entanglement ##

You have likely noticed that reference semantics 
are significantly more complex than pointer semantics,
even though pointers are more powerful.
The source of this complexity is safety.
More precisely, this complexity arises from Cone trying to
squeeze as much performance as it can out of pointer-like patterns,
while also cordoning off any use of references which could
potentially be memory- or data race-unsafe.

This leads to one final set of fascinating observations:
how complexity can also arise
from entangling effects between a reference type's quarks.
Much like how we don't normally expect the properties of a field's
type to infect the properties of its containing type,
we also do not expect types that are composed together
to constrain each other.
However, that can happen:
sometimes a reference's region, lifetime, permission and value type
will constrain each other, or the reference, in intriguing ways.

Curiously, two such entanglements loosely mirror physicists'
observations that the universe behaves rather strangely
at the very tiny (quantum probabilities) and very large (relativity).
Were we to arbitrarily position types along some scale of complexity,
we might place:

- **Linear/affine** types as having the "smallest" complexity.
  A reference is affine (move semantics) if linearity is
  constrained either by the region (e.g., single-owner) or permission (`uni`).
  However, whether the value type is affine has no impact on whether the reference is affine.
  
	  Linearity in references restricts the richness of data structures.
	  When a data graph's references are all affine, the data graph
	  can only be a [directed, rooted tree](https://en.wikipedia.org/wiki/Tree_(graph_theory)#Rooted_tree), 
	  as affine references can represent neither cycles nor multiple references to the same node,
	  as found in many [directed acyclic graphs](https://en.wikipedia.org/wiki/Directed_acyclic_graph).

- **Dependent** types as having the greatest complexity.
  Any value type that has type-shape changing capability (e.g., array references
  or variant types) may be viewed as effectively 
  [dependent types](https://en.wikipedia.org/wiki/Dependent_type),
  since the type's shape/definition depends on a value (e.g., the array size or the discriminant).
  If a reference points to a dependently-typed value
  and the reference permission permits unlocked shared mutability, 
  additional restrictions are imposed on what can be done with such references.
  As [this post](http://pling.jondgoodwin.com/post/interior-references-and-shared-mutable/)
  explains, these constraints prevent the creation of any interior reference 
  (one that points *inside* the referred-to value) which
  could be invalidated by another reference changing the value's shape.

The final form of entanglement I want to mention involves **subtyping** variance.
Because of its polymorphic flexibility, subtyping is a vital ingredient of
Cone's type semantics.
Since "practical" subtyping turns out to be
unexpectedly richer than the standard type-theoretic treatment of subtyping,
I intend to explore it more thoroughly in my 
[next post](/post/practical subtyping).

Until then, let me briefly summarize the relevance of subtyping to reference quantum entanglement.
It arises from the fact that we normally want subtyping to be applied
[covariantly](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))
across each of its type parameters.
This permits a wide-range of valuable reference polymorphism,
including coercion of:

- a region-based reference to a borrowed reference,
- any permissions to `opaq`, or most of them to `const`
- a lesser lifetime to a greater,
- a value type T to its supertype S, or 
- any combination of the above.

However, if the referenced permission allows the referenced-value to be changed but not read,
type safety requires that the reference become
*contravariant* over the reference's lifetime and value type. 
Similarly, if the permission permits the referenced-value to be viewed or changed,
the reference becomes *invariant* over the lifetime and value type.
