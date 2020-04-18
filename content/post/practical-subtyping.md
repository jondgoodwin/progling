---
title: "Practical Subtyping"
date: 2020-03-17T17:50:31+10:00
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

The [last](/post/searching-for-quarks/) 
[two](/post/reference-types/)
posts introduced the fundamental elements underlying Cone's type system.
They also distinguished between nominal vs. structural type equivalence.
However, they were largely silent about subtyping relationships between types.
Given how important subtyping is to Cone's type versatility,
I thought it might also be valuable to offer a detailed treatment
of these mechanisms.

My approach will be practical, rather than 
[formal](https://www.cl.cam.ac.uk/~sd601/thesis.pdf) (alas! no subsumptions nor lattices),
distilling lessons learned from
implementing a rich, but type-safe collection of subtyping mechanisms.
Implementing subtyping in a systems language introduces challenges
not addressed by any current formal treatment of subtyping that I'm aware of,
particularly with regard to runtime substitution and subtype compromises.
These challenges are the catalyst for intriguing solutions.
These challenges and their solutions are not widely documented.

I intend to address these questions:

- How do programmers benefit from language-based subtyping?
- How do static and runtime subtyping differ? *This is the really fun one, 
  as it explores why subtyping rules can vary between value and reference types!*
- How can one evaluate a subtype relationship between two types?
- How do variance rules complicate subtyping?

## Why Subtype? ##

Prudent laziness is a programmer virtue.
For example, it is a delight to write code logic once and have it work
across a broad range of types.
Logic that works across multiple types of values is called 
[polymorphic](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)).
Such polymorphic logic is defined using generalized type parameters.

Subtyping makes this kind of polymorphism work safely.
A [subtype](https://en.wikipedia.org/wiki/Subtyping) is "a datatype
that is related to another datatype (the supertype) by some notion of substitutability."
Safe substitution is the key notion here:
When polymorphic logic is referenced/used, concrete subtypes are substituted into that logic 
in place of the generalized types.
This substitution may occur at compile-time or runtime.

The benefit of subtypes
is thus the productivity advantage the programmer obtains from building
reusable polymorphic code that supports safe substitution of a wide range of types.
The compiler uses carefully-selected subtyping rules to ensure
polymorphic substitution is always done in a type-safe way.

## Compile-time vs. Runtime Subtyping ##

Before detailing subtyping rules, let's explore how subtyping
differs between compile-time and runtime substitution.
This matters, because runtime substitution imposes notable constraints on subtyping.

### Monomorphization ###

Typically, generics or templates are a form of
[**parametric polymorphism**](https://en.wikipedia.org/wiki/Parametric_polymorphism).
Systems languages like Rust, C++ and Cone
use compile-time monomorphization to substitute subtypes into
the generic logic's type parameters.

Consider, for example, the generic 'max' function:

    fn max[T Comparable](m T, n T) T { if m > n {m} else {n} }

The generic type parameter T is given a bound of `Comparable`.
Effectively, this supertype constrains what types
can be substituted for T. Only subtypes may be.
(Note, if no bound is given, this is basically equivalent
to the `Any` supertype, the 
[universal top type](https://en.wikipedia.org/wiki/Top_type).)

If we make use of the generic max function like this: `max(1u, 5u)`, 
this causes an instance of the max function
to be created that substitutes the subtype `u32` into T.
Similarly, another use `max(5u8, 2u8)` would create a second instance (copy) of the
max function which substitutes the subtype `u8` into T.

Thus, monomorphization makes a separate physical copy of polymorphic logic
for every unique combination of type parameters. The resulting runtime has
multiple copies of the `max` function, each specialized to a specific subtype.
Generating multiple instantiations increases code size,
but it has the advantage that the instantiated logic works perfectly
and safely for all subtypes of the type parameter bounds.

### Coercion ###

Subtyping's alternative substitution strategy is coercion.
Coercion (or upcasting) is the implicit, automatic conversion
of any subtype-based value to its equivalent supertype value,
as expected by polymorphic logic.
With this strategy, a single instance of the polymorphic logic is versatile enough to accept
and handle values from many different subtypes. 

There are two flavors of coercion:

- **Compile-time coercion**. This just *recasts* the value's type to a new type.
  No runtime logic is needed to convert the encoded value to its new type,
  as the runtime value is already correctly encoded for use by the supertype.
  
- **Runtime coercion**. This injects runtime logic that *converts* the subtype's
  value to its equivalent supertype value.
  The need to convert the value can arise from many reasons.
  For example, the supertype's value may require additional memory
  space and data to accommodate all possible values from every conceivable subtype.

The subtyping rules for compile-time coercion are as flexible as for monomorphization.
However, the same cannot be said for runtime coercion.
The fact that runtime coercion triggers value conversions 
introduces practical constraints on subtyping substitution that
don't apply to compile-time strategies.
It is not always convenient, or possible, to convert some subtypes' values
to what might be a valid compile-time supertype. 
This is usually due to memory sizing or allocation challenges involved
when creating the needed supertype value.
Examples of these difficulties will be given later.

Both flavors of coercion also introduce downcasting challenges.
Often we also want to convert backwards from a supertype value to a subtype value.
Although upcasting is always type safe, downcasting can only be done safely
by checking, at runtime, whether the value complies with the desired subtype.
The two common approaches to safe downcasting are virtual dispatch and pattern matching.
Such downcasting is only possible when the encoded supertype
value contains enough information to discriminate between possible subtypes.
Such discrimination typically makes use of a variant type's tag
or a virtual reference's vtable.

To summarize, monomorphization and both flavors of coercion are useful,
offering different trade-offs. 
Coercion must be used when the value's type cannot be determined until runtime.
It can also reduce executable size and compilation speed over monomorphization, 
by avoiding logic duplication.
In some cases, however, runtime coercion imposes significant subtyping constraints
and may well degrade performance.
Whether coercion of a value involves compile-time recasting vs. runtime conversion
depends very much on the structures of the subtype and supertype,
which will be explained as we cover the subtyping rules for various types.

## Subtype Evaluation ##

So then, how do we determine that one type is a subtype of another?
What rules can a compiler (or programmer)
use to compare two types and determine one is a subtype of the other?

Helpfully, we know these type comparison rules must comply with the gold standard for subtyping: 
[strong behavioral subtyping](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
(Liskov, 1987).  For type S to be considered a subtype of type T,
it must be true that, for every *provable property* of all objects of type T,
all such properties must also hold true for every object of type S.
This standard is demanding, but it makes intuitive sense.
If we are going to substitute a value of S into logic intended for values of type T,
we want to know the S-typed values will play correctly in that logic.

To derive the subtype comparison rules, it is illuminating
to divide a type's provable properties into three kinds:

- the type's actual **values**, 
- the functional **operations** on a type's values, and
- any **predicates** used to constrain the type's values or operations

For each kind of property, let's articulate its subtype rules,
as illustrated by the atomic number types.

### Value-based Subtyping ###

Every concrete type represents a finite set of values of that type.
For example, Cone's `u8` has 256 possible values, ranging from 0 to 255.
`u16` has 65,536 possible values, ranging from 0 to 65,535.

Notice that every value of `u8` is found in `u16`. 
Looking only from the point-of-view of a type's values, 
we can say that `u8` is a subtype of `u16` (`u8 <: u16`),
since any value of `u8` may be substituted into `u16` with no loss of information.

This gives us our first subtyping rule: every value in a subtype
must have a distinct, "equivalent" value in the supertype.
A subtype holds some conceptual subset of the supertype's values.
It specializes the supertype's values.

Subtyping is unidirectional and not commutative.
When `u8 <: u16` is true, the reverse `u16 <: u8` is not.
This is obvious since many values in `u16` 
(e.g., 1000) are not found in `u8`. Any attempt to substitute would lose information.

Often, two types have no subtyping relationship either way.
For example, `i8` has values not found in `u8` (e.g., -10)
and `u8` has values not found in `i8` (e.g., 200).

From these simple examples, we can already see that a subtype comparison between
types establishes a [preorder](https://en.wikipedia.org/wiki/Preorder)
(and in most languages a [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set)),
similar to the way comparison operators do for numbers.
A subtype evaluation between any two types T1 and T2 yields one of four results:

- T1 is the same as T2 (type equivalence), 
- T1 is a subtype of T2
- T2 is a subtype of T1, or
- T1 and T2 are not comparable.

**Runtime Coercion of Numbers**: Languages that allow subtype coercions between integer or floating point number types,
will typically need to do so using runtime conversion.
For example, the runtime coercion of a `u8` value to `u32` performs a conversion that left-pads
the top three bytes with 0.

### Operation-based Subtyping ###

Types don't just represent a set of values, they also establish which
operations may be performed on those values.
For example, one can use functions or methods to
add, subtract, multiply, and divide integer numbers (e.g., `u8`).

This is our second subtyping rule: 
every operation that can be performed on a supertype must also be
supported by every subtype.
Further, every operation must evaluate to an equivalent result.
For example, adding 3 and 6 must result in 9 for both `u8` and `u16`.
Largely this is true, but it can break down when computations overflow.
Thus, `u8` is nearly, but not strictly, a subtype of `u16`.

In Cone, a type's operations are methods,
defined syntactically as part of the type.
This makes it easier for the Cone compiler to compare all method signatures
between two types as part of a subtype check.
It requires not only that a subtype must implement
at least every method implemented by a supertype, 
it also requires that all such method signatures match 
in name, parameter types and return type.
Later, the rules for doing a subtype match on function (or method) signatures
will be articulated.

### Predicate-based Subtyping ###

A type predicate is a logical proposition expected to hold true
for a type's values or operations. Consider these predicate examples:

- Only the even numbers from 0 to 255 (a refinement type of `u8`).
- For some type's pair of numbers, the first number must be no greater than the second
  (an invariant for a dependent pair type).
- The square root operation requires and must return a positive number
  (pre- and postcondition contracts).
- The size of a vector value is unchangeable (Liskov's history constraint)
- All operations are guaranteed to terminate (the halting problem)

As these examples demonstrate, predicates can introduce considerable type complexity
to a language, sometimes so much that a compiler (or even a person) cannot guarantee
their complete compliance. Few languages support type predicates,
and the ones that do often de-fang them to some degree.

For languages that do support predicates, these subtype principles apply:

- Types with "refinements" (predicates on data) are subtypes of the same type
  lacking those refinements.
- Every invariant found on a supertype must also be enforced on subtypes.
- Preconditions may not be strengthened in a subtype
- Postconditions may not be weakened in a subtype

The [circle-ellipse](https://en.wikipedia.org/wiki/Circle%E2%80%93ellipse_problem)
offers a helpful illustration of the invariance considerations needed to ensure
that a Circle type can be correctly considered a subtype of the Ellipse type.

Due to their complicating impact, Cone offers very limited support for predicates.
Cone will support preconditions and postconditions on methods, but its subtyping rules will
likely only check for equivalent compliance vs. weakening or strengthening.

As for explicit refinements (data invariants), Cone will not support them.
However, such invariants might be implicitly protected by method logic.
If a type has implicitly enforced invariants, 
it should make private all fields protected by the invariants.
Doing so helps the compiler
make safer subtype decisions, as private fields are handled differently.

## Record Types ##

Extending these subtyping rules to record types (e.g., structs or classes) 
that support named, typed fields and methods is largely straightforward:

- Every **field** in a supertype is also part of the subtype.
  The type of the subtype's field should be equivalent, or a subtype of, the supertype's field.
  (It is called depth subtyping when fields can subtype.)
  These value-oriented rules ensure every possible value of any subtype
  has an equivalent value in the supertype.
  
- Every **method** in a supertype must be implemented by the subtype.
  All such method signatures must match in name, parameter types and return type.

Record type subtyping introduces additional considerations:

- **Subclasses vs. subtypes**.
  In object-oriented languages, subtyping is sometimes confused with subclassing
  (where a new class inherits fields and methods from other classes).
  The concepts often overlap, but it is possible to have one without the other;
  subtypes may be defined without inheritance and subclasses might not be subtypes.
  Although a subclass will be a valid subtype in terms of values and methods
  (because inheritance ensures all fields/methods of the base classes are in the subclass),
  it may fail the predicate compliance requirement of behavioral subtyping.

    To address this, Cone does not structurally match on private fields of record types.
    As mentioned earlier, this makes it easier to enforce implicit invariant predicates
    in the presence of subtyping, so long as fields subject to invariants are
    made private and the logic of all methods protect those invariants.

- **Nominal vs. structural subtyping**.
  A language compiler must decide whether to use nominal vs. structural subtyping
  when deciding whether one record type is a subtype of another.
  Structural subtyping involves looking at all fields and methods of both types
  to ensure they match, according to the above rules.
  Nominal subtyping need only examine whether the subtype explicitly indicates
  it implements the supertype. Explicit implementation guarantees a structural match.
  
    Cone supports structural subtyping, 
    because interfaces may be defined independently of (and after) concrete types.

- **Record Coercion**.
  A specific record type allows compile-time coercion to a supertype
  only when no fields' types require a runtime conversion and there are no
  subtype-only fields. If the record types fails these requirements,
  it is theoretically possible that a runtime coercion capability could be implemented.
  However, most languages (including Cone) 
  choose not to support runtime conversion of record values,
  as doing so would require involved logic that allocates memory for the supertype value,
  and then converts or copies over each field's values one-after-another and recursively.
  The common workaround for not allowing runtime coercion on record type values is to 
  support coercion between *references* (or pointers) to record-structured values.

### Arrays ###

A static-sized array may be viewed as comparable to a record type that
has as many identically-typed fields as the array's size.
Viewed this way, array subtyping rules are easily derived:
an array subtype must have at least as many elements as the supertype,
and the element type must be equivalent or a subtype of the supertype's element type
(barring issues around mutability and variance, 
which will be covered shortly).
Like record types, array values do not support runtime coercions,
but do support compile-time coercion if coercion of the element type is also compile-time.

## Abstract vs. Concrete Types ###

Earlier, I was vague on whether a record subtype
may have more fields than its supertype (width subtyping).
This is because doing so provokes an interesting dilemma.
If we allow a subtype to define more fields than its supertype, then the subtype
is able to represent distinct values not also possible in the supertype.
This dilemma may be resolved by
distinguishing between concrete and abstract types.

So far, this post has only dealt with concrete types,
types that have a finite number of enumerable values.
[Abstract types](https://en.wikipedia.org/wiki/Abstract_type)
(otherwise known as existential types, traits, interfaces,
abstract classes, etc.) do not have a finite number of enumerable values.
Instead, they represent criteria that concrete types must satisfy
in order to allow safe subtype substitution for polymorphic code.
Because of this role, abstract types are most commonly used by
polymorphic code, as bounds on generics or as the types expected by polymorphic functions.

Although abstract types are often syntactically similar to record types,
the interpretation is different;
it specifies that any compliant concrete type must *at least* have its fields
(by name and type) and its methods (by name and signature).
Any concrete type that complies is effectively a subtype of the abstract type,
even when it adds more fields ([row polymorphism](https://en.wikipedia.org/wiki/Row_polymorphism))
or methods not defined by the abstract type.

Class-based languages that allow a concrete subclass to be inherited and extended
get around this conceptual dilemma by treating the superclass
as an abstract type when handling polymorphic substitution.
By contrast, Cone's structs are concrete types and traits are abstract types.
Traits perform two related roles: they act as
supertypes of other types that implement or comply with all their defined
fields or methods, and they offer the reuse benefits of inheritance when
defining new compliant subtypes.

### Sum/Variant Types ###

Sum types and abstract classes are supertypes of their variant types,
by design. It does not take much thought to see how their subtype
capability derive easily from record subtype rules.
These supertypes are abstract types that well support compile-time substitution,
but may or may not support compile-time coercion.

Compile-time coercion becomes possible when the base supertype 
defines a fixed (closed) number of same-sized variant subtypes,
as well as a discriminant tag field.
As with abstract types, you cannot instance the supertype value directly, 
but you can instance one of its variants,
and then later easily coerce it to a supertype value.
Recast coercion is possible at compile-time 
because the compiler effectively guarantees that all subtypes and the supertype
have the same size. Thus, no data conversion is needed, and 
the type of the value can always be pattern matched using the discriminant.

Open-ended variant types that do not have a same-size guarantee 
(as is true for classes in most OOP languages) can be statically coerced or not
according to the same rules given for record types.

## Variance ##

No conversation about subtyping is complete without discussing 
[variance](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)).
Variance refers to the subtyping relationship between a complex
type (e.g., a generic) and its component types:

- **Covariance** preserves the order of subtyping.
  Depth subtyping in record and array types is covariant:
  A struct's fields subtype in the same direction as the struct.
- **Contravariance** reverses the order of subtyping.
  This shows up with function types and mutable references.
- **Invariance** turns off subtyping, allowing only type equivalence.

I find it helpful to visualize variance in terms of information flow.
A value of a subtype can flow into a supertype, but not vice-versa.
Using this concept, it becomes easier to visualize why functions
and mutable references would introduce contravariance.

### Function Subtyping ###

Languages generally don't substitute (or coerce!) functions themselves.
But they certainly do substitute references to functions (or methods),
which is why we need to know their subtyping rules.

In the earlier section on operation-based subtyping rules,
I indicated vaguely that method parameters and return types need to match.
but did not say how. Now I can:
function (and method) signatures are contravariant on their parameter types
and covariant on their return types.
This aligns with the observation that argument values flow
into the function and return values flow out.

### References ###

We are back once again to the powerful (and complex)
semantics of reference types.
There are two interesting things to say about the subtyping rules for references:
first, how the reference's permission affects subtyping's variance rules,
and then how easily and richly references support coercion.

#### Reference Variance ####

Reference regions, permissions and lifetimes all support subtyping:

- **Region**: a borrowed reference is a supertype to any region-owned reference
- **Permission**: `const` is a supertype to most permissions. `opaq` is a supertype to all.
- **Lifetime**: A longer lifetime is a subtype of a shorter one whose scope is wholly
  within the longer lifetime.

Bearing that in mind,
a reference is always covariant over its region and permission type.
However, the variance of the reference's lifetime and value type
is determined by the reference's permission:

- Covariant, when the permission forbids mutation of the referred-to object
- Contravariant, when only mutation is allowed.
- Invariant, when both reading and mutation are allowed.

#### Reference Coercion ####

Most types discussed so far support rich compile-time subtyping,
but largely slam the door on runtime coercion.
By contrast, references are more amenable to runtime coercion because the small, fixed-size
of the underlying pointer makes pointer conversions easy to perform.
This is why it is sometimes possible to coerce a *reference* to some record subtype
into a reference to the record's supertype, even when it would not be
practical to coerce the record type's *value* to its supertype.
It does not matter if the pointed-at value has a varying, unknown size,
because it remains unchanged and unconverted.

Whether the coercion is compile-time or runtime-based depends on the
nature of the transition between the subtype and supertype. For example:

- **To a regular reference to a trait**. It is a compile-time coercion
  when the value's supertype is a base trait mixed into the subtype.
  The coerced regular reference may be only be used to view the
  common trait's fields or perform its default methods.
- **To a permission supertype**. It is a compile-time coercion when from
  a static permission, and a runtime coercion when from a locked permission.
- **To a borrowed reference**. It is a runtime coercion when from a region-owned reference,
  because the pointer's position needs to be recalculated.
- **To a virtual reference**. It is a runtime coercion
  when the coercion creates a virtual reference bearing an attached vtable.
  The virtual reference allows virtual dispatch to trait-declared
  methods and indirect access to trait-declared fields.

### Generics ###

In many languages (e.g., C# or Scala), it is possible to mark a generic's type parameters
to indicate their expected variance.
I am not yet sure whether these annotations will be required in Cone,
as I am wondering whether these variance annotations can be inferred
based on syntactic positioning and existing annotations on types (e.g., reference permissions).
If such inference is not possible, I will likely support
variance annotations on a generic's type parameters, where required.

## Summary ##

Summarizing the practical considerations for offering subtyping support in a 
programming language has been a long read!
Although a compiler cannot perfectly ensure subtyping is always done safely,
it can go a long way to enforce useful subtyping rules that permit
a wide-range of versatile polymorphism with types.

A big takeaway is understanding how restrictive runtime-based coercion is
compared to the rich capability supported by compile-time monomorphization and coercion.
That said, there are still many useful times when a value may be safely and implicitly coerced
to its supertype's value, for example:

- From `u8` to `u32` (*runtime*), left-padding with three bytes of 0.
- From `i32` to `Option[i32]` or `Result[i32, ErrCode]` (*compile-time*), for sum/variant types
- From `&[10]u8` to `&[]u8` (*runtime*), from an array reference to a slice
- From `&Car` to `&<Vehicle` (*runtime*), a reference to a vtable-based virtual reference
- From `&rc mut i32` to `&i32` (*compile-time*), a safe, static re-cast to a borrowed, const reference
