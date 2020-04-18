---
title: "Searching for Quarks in Systems Types"
date: 2020-03-01T16:21:43+10:00
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

When I began work on the [Cone](http://cone.jondgoodwin.com/) programming language,
I believed I had a confident handle on which primitive types
this modern, safe systems language required:
various sizes of atomic number types,
fixed-size arrays, programmer-defined structs, raw pointers, safe references,
discriminated (and raw) unions, generics, and limited support for void and tuples.

My confidence was misplaced.

Once or twice a year, implementation challenges and further research
forced me to re-think aspects of the type system, each time deepening my
understanding of what it required. On the latest go-round, implementing generics
triggered another re-examination: void and tuples need expanded roles,
atomic number types require more nominal varieties, and
references are richer and easier to manage when defined using generics and inheritance.

This historical analogy offers context for my journey:
In the early 1800s, John Dalton introduced the first evidence-based theory
showing that matter is composed of several kinds of "un-divisible" atoms.
It took another hundred years of research (and Mendeleev's formulation of the periodic table)
until Rutherford, van den Broek, and Moseley convincingly
demonstrated that atoms are, in fact, not atomic.
Every atom is composed of more elementary particles:
a nucleus of protons and neutrons surrounded by a statistical cloud of electrons.
Only 50 years later, Gell-Mann went deeper still, demonstrating that neutrons and protons
are also not fundamental, as each may be de-composed into three 
colored and flavored quarks. Although we have hit bottom, are we done?

At each stage in this process of discovery, our earlier models remains largely correct and useful,
so that we can usefully talk about the behavior of atoms, say, or electrons.
And yet, our knowledge deepened every time,
re-shaping a more precise understanding of the universe's underlying particles and forces.

My deepened understanding of Cone's type system has unfolded in a similar way.
It began with a well-known and still useful collection of fundamental, primitive types.
However, not all of these types are, in fact, atomic.
Progressively, step-by-step, I find that I am discovering the quarks and fundamental forces
that undergird these still valid but not-quite-primitive types.

It is unlikely that this journey of discovery has finished.
However, I have received several requests to document the latest model
for Cone's type system, highlighting its key underlying mechanisms
and explaining the value these mechanisms offer to programmers.

## Goals for Systems Types ##

Before diving into the details, here are the
goals which drive the evolution of
[Cone](http://cone.jondgoodwin.com/)'s type system:

- Tight alignment between the type primitives and the binary interface surfaced by its target CPUs.
  Key examples: power-of-two-sized integers/floats, aligned fields in record structures,
  and direct manipulation of memory via pointers.
  Such tight alignment facilitates both performance and fine-grained resource control,
  qualities systems languages are prized for.
  
- Avoiding the undefined behavior and safety issues associated with languages
  like C or C++, resulting in significant improvements to code quality, maintainability,
  and imperviousness to security attacks.
  Cyclone and Rust demonstrate that a systems language need not lose
  any of its advantages by adopting a sound type system.
  
- Fusing together high-level, rigorous abstractions from multiple traditions 
  that more accurately capture design intent and improve developer productivity.
  Good examples include generics (polymorphism), traits (subtyping), move semantics (affine types),
  type-to-type inheritance, isolation, variant types that go beyond sum types, 
  row polymorphism, and first-class closures.

- Versatile and safe abstractions that allow the programmer to choose
  the optimal mix of [memory management](http://cone.jondgoodwin.com/memory.html)
  and concurrency strategies that best address a program's requirements,
  using regions, lifetimes and permissions.
  
## Atomic Number Types ##

Atomic number types are fundamental and pervasive.
CPUs natively support two kinds of numbers: integers and
[IEEE-754](https://en.wikipedia.org/wiki/IEEE_754) floating point numbers.
To this end, Cone's core library offers a convenient starter set of number types,
whose names specify how many bits encode a number:

- Signed integers: `i8`, `i16`, `i32`, `i64`
- Unsigned integers: `u8`, `u16`, `u32`, `u64`
- Floats: `f32` and `f64`
- `Bool` (effectively a 1-bit integer)

These convenient, built-in types support a variety of basic
arithmetic, bitwise logic, and comparison operations.
Using type extension, additional methods may be defined for
these types (e.g., adding trigonometric functions to floating-point numbers).

Befitting this post's theme, these are not fundamental types.
Instead, they are composed from other, more primitive language capabilities.
This is done using a type declarator (e.g., `integer`) that assigns a type name
to a new number type. The declaration specifies:

- whether the number is a signed integer, unsigned integer, or floating point number.
- the number of bits used to encode the number.
- the methods which can be used to work with numbers of this type.
  These methods can overload operators (e.g., "+") and may access compiler-intrinsic
  functions to perform their work.

Programmer-defined number types have several interesting properties:

- Defined number types are nominal, meaning that two number types defined in exactly
  the same way, but with different names, are treated as different types.

- Number types may be defined using generics, varying according to the specified bit-size.
  Thus, `f32` and `f64` are specializations of the same generic floating-point number type.
  
- Number type literals may specify their correct type as a suffix (e.g., `2i8`).
  If unspecified, an integer literal defaults to `i32` and a float literal to `f32`.

- The built-in type conversion operator (`into`) makes it easy to convert a number of one type
  to a number of any other type, possibly with the loss of some precision.

This more-complex, nominal approach to number types enriches type safety,
by guarding against accidentally bleeding together two numbers that
have semantically different meanings. For example:

- One might want to use different number types to distinguish between units of measure, thereby
  potentially avoiding problems like the loss of the $125 million 
  [Mars probe](https://www.latimes.com/archives/la-xpm-1999-oct-01-mn-17288-story.html).
- Different number types could make different promises for undefined behavior
  (e.g., protecting against undesired overflows), or could guarantee the
  number values are bounded by some range.
- Each named set of enumerated values should belong to its own independent type,
  and should prevent arithmetic on an enumerated integer value.
- Integers that support intrinsic atomic operations can be separately typed.

## Composite Types ##

Composite types make it possible to work with a structured collection of values
that are stored side-by-side in memory.
There are four kinds of composite types in Cone: arrays, tuple, structs, and traits.

### Arrays ###

An array in Cone holds *n* values of the same non-zero-sized element type.
Its element values are properly aligned one-after-another in memory.
The size *n* is some positive, non-zero integer value known at compile-time.
(Later we will describe array references, which support arrays whose
size is determined at runtime.)

Arrays are structural types; if two array variables are declared with the same size and element type
(e.g., `[4] f32`), their types will match and values can be copied freely between them.
Methods cannot be directly defined for any specific array type.
However, you can accomplish this goal by embedding the array as the only field in a named `struct` type.

Cone arrays are thus similar to C arrays, except for two improvements:

- Indexing is bounds-checked at compile-time or run-time.
- Arrays are copyable values (rather than implicit pointers to element values).

Sidenote: At some future point-in-time, Cone may also support SIMD-capable vectors
which syntactically resemble arrays.

### Tuples ###

A tuple is an ad hoc, ordered, aligned collection of sized values of possibly varying types.
Tuples are particularly convenient for parallel assignment,
multiple return values from a function, and pattern matching.
A tuple's type is the ordered list of the types of its values.

Like arrays, tuple types are structural.
So long as two tuples have the same number and types of values, they type match.
If two variables are implicitly defined as having the same tuple types,
one may copy the value of one variable into the other.

Originally, I restricted Cone's use of tuples to parallel assignment, return values,
and pattern matching, where their value is undeniable.
However, given that Cone uses the `Result` generic type for function call error handling,
it became evident that Cone must add support for tuple types
by field and variable values, so that a function non-exceptional
return value can be a tuple.

### Structs ##

A new named struct type is defined using `struct`.
This definition enumerates some number of uniquely-named fields, each having a specific type
and possibly a default value.
It may also define some number of methods that operate on values of this type.
Each field is designated as private or public, with private fields only
accessible by the type's methods.

Struct types may be correctly viewed as an enrichment of tuple types,
offering more flexibility in how they can be represented and used.
There are notable differences: struct types are nominal (vs. structural),
so two struct types never match, even if their fields and methods are identical.
Struct types must be separately defined before they can be used, making them
slightly less convenient, but also more easily maintained.
Struct typed-values cannot be de-structured the way tuples can,
but they ease named-access to field contents 
and support invariant isolation through the use of private fields.

Underneath this simple, ubiquitous capability lies several useful semantic variations,
specifyable using quark-like attributes.
Let's examine some of the more intriguing:

**Layout**: A struct type indicates how to lay out its field values:

- **aligned** (the default): byte padding may be inserted before each field
  to ensure its value is properly aligned according to its type.
  For example, a 64-bit integer will be pre-padded as needed to ensure
  it is 8-byte aligned (its memory address ends with the binary bits '000').
  Alignment ensures high-performance CPU access to the field value,
  but it can potentially waste memory.
- **packed** starts each field at the byte immediately following the bytes used
  by the previous field. This layout is effectively byte-aligned.
- **bitpacked** injects no padding for alignment.
  This makes it possible to tightly pack multiple bit flags into a single byte,
  further improving memory utilization and cache efficiency.
- **union** overlays all fields to start at the beginning of the structure.
  The size of the structure is the size of the largest field.
  Obviously, a union structure can only hold a value of one field at a time.
  It is up to some algorithm to safely determine which field a
  union struct is holding at any particular point in time.

**Opaque**: Sometimes we want to work with data whose size and contents are
unknown to a Cone program, but are known by some other system (such as an OS).
One example might be a WebAssembly program that holds a reference to some
Javascript object.
An opaque struct may be used to define this type to a Cone program.
It defines no fields, but may define methods.
For obvious reasons, a value of this type cannot be stored, accessed or worked with
directly. Opaque types are only allowed as part of a pointer or reference type
which forbids de-referencing.

**Inheritance**: Cone offers an somewhat different form of
[delegated, multiple inheritance](http://pling.jondgoodwin.com/post/favor-composition-over-inheritance/).
This allows one to call a field's methods or access a field's fields
directly, as if they were part of the aggregate type.
This capability makes it possible for a type to compositionally
reuse the state or behavior of designated fields.

### Traits ###

Trait definitions enumerate named fields and methods identically to struct definitions,
However, where structs are concrete types, traits are [abstract](https://en.wikipedia.org/wiki/Abstract_type).
This means:

- Trait values are never directly instantiated.
- Trait fields need not be specified
- Trait methods need not offer an implementation.

Traits abstract away what is common across multiple struct (or number) types,
and can do so even when a struct does not implement the trait (structural subtyping). 
This makes traits useful for several polymorphic roles:

**Bounds for generic type parameters**. Type checking uses this information
to enforce that any type substituted into a generic supports all
fields and methods referenced by the generic's logic.

**Reference-based virtual dispatch**.
Traits can build vtables able to map the fields and methods
of any compliant concrete type to the abstract interface described by the trait.
In Cone, vtables support not only traditional subtype polymorphism,
but also statically-enforced row polymorphism.

**Traits as Mixins**.
Similar to the struct-based inheritance described earlier,
traits may be [mixed into](http://cone.jondgoodwin.com/coneref/reftrait.html) structs (or traits).
The key difference is that any trait mixed into a struct
dissolves into it completely, such that all of the trait's fields,
as well as any methods not implemented by the struct,
are folded into the struct as if truly defined there.
It is as if the trait were being used as a reusable macro
by every struct that wants to offer the trait's capabilities.
This is helpful when we want multiple concrete types to share some characteristics in common,
such as for virtual dispatch or row polymorphism.

**Base Traits for Variant Types**
When a struct's definition begins with a trait mixin,
it is called a [base trait](http://cone.jondgoodwin.com/coneref/reftraitvar.html).
Since we know that a base trait's common fields are
found at the start of any concrete type that implements the base trait, 
we no longer need a vtable
to polymorphically perform operations on these common fields,
thereby improving performance and versatility.

Base traits support two intriguing "sub-atomic" attributes:

- **Tagged variants**. A base trait may designate one unnamed field to act
  as a discriminant (enumerated tag). Any concrete struct that implements the base trait
  carries this tag field. 
  The tag field's value is implicitly filled with a unique integer
  that corresponds to the concrete type used to construct the value.
  Pattern matching can then examine the value of this hidden tag field 
  to determine the concrete type of some object. 
  By specifying a tag field in a base trait, Cone's variant types
  offer all the same capability (and more) as sum types in other languages.

- **Same-sized vs. varying size**. Normally, the byte-size of some object
  is determined only by its concrete type. However, if a base trait is
  designated as "same-sized", then every concrete type that implements
  the base trait is padded, as needed, to ensure they are all the same size,
  as large as the largest concrete type that implements the base trait.
  This can be very convenient when we want to pass around
  variant type objects by value, or organize memory pools for
  cache-efficient handling of collections of such variant type values.

**Closed vs. Open**. Any base trait that specifies a tag field or is designated "same-sized"
is restricted to having a closed number of concrete variant types.
All variants must be declared in the same module that declared the base trait.
By contrast, base traits are open when they allow objects to vary in size and don't have a tag field.
Open base traits may be implemented (extended) by concrete types across
any arbitrary number of modules.

**Sum types**. It is also worth pointing out that Cone's type primitives are not built strictly
around the type-theoretic notion of algebraic product vs. sum types.
Instead, Cone's two "product"-like types, structs and traits, may be composed into
pure product types, pure sum types, or variant types (a superset hybrid of product and sum types).
Thus, any syntactic capability for constructing pure sum types
(ADTs, Rust's `enum`, `i32 | None`) is simply syntactic sugar that is lowered
implicitly into traits and structs.

## Function Signatures ##

A function's signature specifies the name, type and default value of its passed parameters,
as well as the type(s) of its return value(s). 
Every function signature is effectively a structural, opaque type.
It is structural because two function signatures are considered the same type
if their parameter and return types match exactly.

A function type is opaque, because we generally don't want to access, pass around, or mutate
the function's value (its compiled logic).
Instead, we largely just want to do two things with "values" of function type:

- Invoke their logic, passing them properly typed arguments and getting back return values.
- Pass around some pointer or reference to an arbitrary function we know matches
  an expected function signature, for later invocation. In other words, first-class functions.

Function types support a broad variety of sub-atomic attributes:

- **Method vs. static**. Methods are functions that operate on values of the
  method's type, passed as the first parameter named `self`.
  Methods may be overloaded, but static functions may not be.
  Further, methods may be designated as `get` (the default) or `set`.
  This form of syntactic overloading enables
  `set` method calls to be invoked when placed to the left of an assignment operator.

- **Purity**. [Purity](http://cone.jondgoodwin.com/coneref/reffunc.html#pure)
  attributes impose guaranteed constraints on
  what a function is allowed to do. Pure functions may not access global, mutable
  variables, nor may they call any function that is not pure.
  SImilarly, any function marked as `initpure` may not read any uninitialized global variables
  in the current module and may only call functions marked as pure or initpure.
  
- **Unsafe**. This flags a function (typically built in a different language) as
  not offering memory, concurrency or other safety guarantees.
  It forces any use of such functions to be marked as trusted, because
  the programmer has done the due diligence to ensure the function is only used safely.
  
- **Calling convention**. By default, Cone functions use the C calling convention.
  However, when calling functions written in another language (FFI),
  those functions may need to be annotated with some other calling convention
  that correctly reflects how registers should be assigned to parameter and return values.
  
Finally, although Cone supports **closures**, these are not a primitive type.
Instead, closures are syntactic sugar, lowered to a mix of structs, methods and
traits.

## Void Type ##

The void type represents the absence of any possible values.
An obvious, and safe, use of the void type is for the return type of a
function that returns no values.

Void is not a valid type for a variable, function parameter, or anything
else requiring a non-zero-sized value (e.g., an array element type).
For the same reason, it is not valid to have a pointer or reference to void,
despite this being a common practice in C.

The other place that void turns out to be useful (and allowed) is for fields.
As with tuples, the reason derives from Cone using `Result` for function return values.
We want an error-throwing function to be able to return nothing (void).
Thus `Result<T, E>` needs to allow `void` for T (or E).
Given that the Result generic is based on base traits and structs (as with all sum types),
this means we need to be able to define a field as having void type.
For safety, we disallow any attempt to retrieve or change a void-typed field.
It is as if a field is "deleted", if its type is void.

Cone's void is not the same as the empty or bottom type.
The bottom type is a subtype, which void is not.
Also, a function that returns the bottom type must never return.

One may interpret void as if it were a non-subtyped unit type,
as it operates effectively in the same fashion.
Despite the operational equivalence, I find it more convenient
to understand it as representing the absence of any value,
rather than a single-valued type.

## Pointers and References ##

One key way I distinguish a systems programming language from some other form of
programming language, is whether it supports pointers as a distinct form of type,
cleanly separated from other types. Non-systems languages suppport pointers or references,
but do so largely implicitly, where a declared object of some type 
is understood to really be a pointer to an object of that type.

Cone's pointer and reference types are cleanly separated from the other "value" types.
Reference types are surprisingly rich in subatomic structure,
especially given that references come in three flavors (regular, array, and virtual) 
and depend on several other kinds of type structures, such as regions, permissions, lifetimes
and affine/move semantics.
Because of this richness, I am going to save discussion of these types to another post
(this one is already too long). Hold on to your hat for that!

## Concluding Observations ##

So, that's a high-level overview of all the fundamental types for Cone.
I suspect there will be further adjustments over time, as it is unlikely
I have figured it all out.
For example, I may want a fundamental type for thread-based execution stacks.
I may also want to add the bottom type, as well as intersection and union ad hoc types.
We'll see.

It fascinates me how many ways Cone's type system diverges from C's,
despite many obvious similarities. Among other popular systems languages,
it is perhaps most similar to Rust, but even then there are notable differences,
particularly with traits, inheritance, and references, as well as many subatomic variations.
Noticing this gives me a deeper (and more humble) appreciation of
how many different ways that languages (even ones with similar goals)
can vary not just syntactically, but in their underlying type semantics.
No wonder it is profoundly difficult to create interoperability
bridges between languages that do not already share a common ecosystem
(e.g., JVM or .NET).

Although the resulting framework is versatile, safe and performant,
it also ends up in a very different place than the sort of type system
constructed by a PL theorist. This should not be surprising given
the very different goals of PL theory and systems programming languages.
Neither is empirically better or worse than the other, they are
just temperamentally designed for being good at different objectives.

I titled this post "Searching for Quarks in Systems Types", because
I wanted to focus special attention on the fascinating subatomic details that
can usefully live underneath several well-known type flavors.
I hope you have seen how this played out several times,
in the useful distinction between structural (for ad hoc convenience) 
and nominal types (to obtain type strictness and method capability),
and in the attribute-based flavors one can select for structs, traits
and functions. You'll see a lot more of this when the follow-on post
dives into [references](/post/reference-types). 