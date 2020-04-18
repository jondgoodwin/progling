---
title: "Delegated Inheritance"
date: 2019-07-19T07:42:08+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "inheritance"
tags:
  - "Cone"
  - "Go"
  - "D"
  - "Rust"
---

After removing the interface, inversion of control, and protected access capabilities
from traditional inheritance, what do we have left (besides composition)?
This is what we have: placing a few extra tokens on a derived class causes all named fields
and methods of one or more base classes to be absorbed as if explicitly incorporated.
Further, certain inherited methods can be customized (overridden) with their own implementation.

The primary selling point for inheritance has always been this sort of code reuse.
With little effort, one or more types can absorb and customize the
state and behavior of another type. Helpfully,
any future changes to the base type are automatically manifested in all derived types.

Sure, programs with largely-independent types
won't benefit from this "absorb and customize" capability.
However, there are many which would.
Consider programs which work with graph-based data structures holding
variant-typed nodes that share some common state and behavior.
The variant node types of a browser's DOM and a compiler's AST/IR become much easier
to define when this simple form of inheritance is available.

Although parametric polymorphism (generics/templates) also facilitates code reuse,
its gift is to stamp out multiple types with equivalent capability.
Generics aren't well-suited for creating a new type that adds to and customizes another.
Macros, which also facilitate code reuse, *can* mimic the code replication
capability of inheritance. 
However, not only is the technique for doing so more cumbersome and verbose,
customizing what gets replicated is pretty painful.

Are there any drawbacks to this basic inheritance capability?
I have heard these complaints:

* What if I really *don't* want the derived type to be polluted
  by absorbing *all* of a base type's behavior?
  Or, worse yet, all behavior added in the future to the base type?
* We expect a derived type to be a subtype of the base type,
  such that the derived type "is a" base type
  (e.g., a Cat is a Mammal). However, the inheritance relationship 
  between types often feels much more like possession
  ("has a") than subtype specialization ("is a").
* If multiple inheritance is used, how do we handle when
  multiple base types have methods or fields bearing the same name(s)?
  
As it turns out, all three of these complaints originate from the
same essential problem, for which a simple fix offers a remedy.
We will come back to this very point after a bit.

First, let's review the solutions various teams have
proposed for satisfying this legitimate requirement, 
without making use of inheritance.
What is interesting is not just how varied the proposed solutions are,
but also the different metaphors used to describe
how the solution works.

## Explicit Delegation ##

When *Design Patterns* recommends we favor composition over inheritance,
they know composition alone will not satisfy the code reuse requirement.
They propose that programs use explicit delegation.
"Delegation is a way of making composition as powerful for reuse as inheritance."

What do they mean by delegation, and is it really as powerful?

The heart of explicit delegation is the creation of proxy methods
that forward specific method invocations from one object to another.
Instead of the derived class inheriting methods from a base class,
both classes would be implemented independently.
The derived class's state would incorporate (or reference) an instance of the base class,
captured when an instance of the derived class is created.
Instead of automatically inheriting the base class's methods,
the derived class would create proxy methods of the same name
that forward the method request on to the base class's corresponding method,
and then forward back any response.

A somewhat more involved example uses interfaces to enable dynamic delegation.
A generic public Interface would be defined,
which the base class (and others) would then implement.
Instead of referencing the base class, the derived class would
hold a reference to some instance that correctly implements the interface.
Again, proxy methods would forward method calls from the derived class
to whatever instance properly implements the interface.
An illustration of this approach is shown in Wikipedia's
article on [Composition Over Inheritance](https://en.wikipedia.org/wiki/Composition_over_inheritance#Composition_and_interfaces).

Is explicit delegation really as powerful as inheritance?

Functionally speaking, not quite.
Explicit forwarding delegation alone
is an insufficient substitute for 
inheritance's Inversion of Control and Protected Access capabilities.
However, if we take those capabilities off the table, 
as we did in the previous post, then yes:
Explicit forwarding delegation gives us nearly the same functional capability as inheritance.
The only adjustment we would have to make is that accessing a base class
field via the derived class requires a somewhat longer term
(e.g., `derived.base.x` instead of `derived.x`).

This approach works. It is used by programs written in languages that offer
no broad-based inheritance capability.
However, this technique requires extra work on the part of the programming team
to create, test, and maintain all these proxy methods
in the face of potential future changes to the base types.
One Rust RFC proposal expresses the need for a leaner approach eloquently:
"Efficient code reuse is about making us more productive, 
not just in terms of typing less, which is nice, 
but being able to clearly express intent in our code, 
making it easier to read, understand, refactor and prototype quickly."

The other drawback is the potential impact on program performance.
Depending on the effectiveness of the optimizer,
routing method invocations through one or more extra layers of
hand-written forwarding methods has the
potential to slow down program throughput.

## Rust's Delegation Sugar ##

Rust made the deliberate decision to not support inheritance:
"To avoid the pitfalls that many inheritance-based languages have fallen into, 
Rust has avoided that form of polymorphism entirely, 
preferring instead a combination of behaviorally constrained parametric polymorphism 
(traits and generics) and straightforward type composition."
In point of fact, Rust does support a simple form of inheritance:
Traits may implement default methods that will be inherited by any type
that implements the trait but does not provide an implementation for those methods.

For years now, various teams have been exploring and proposing
different ways to support enriched forms of inheritance
(for example: [this summary](https://github.com/rust-lang/meeting-minutes/blob/master/Meeting-inheritance-2014-09-23.md)
and [Niko's intriguing post](http://smallcultfollowing.com/babysteps/blog/2015/08/20/virtual-structs-part-3-bringing-enums-and-structs-together/)).
One of the more promising proposals is a
[year-old RFC #2393](https://github.com/elahn/rfcs/blob/delegation2018/text/0000-delegation.md)
that buys into the delegation metaphor, but
offers sugar that automates the generation of delegation methods.

If a struct S has a field f whose type implements the TR trait,
one could write:

	impl TR for S {
		delegate * to self.f;
	}

The Rust compiler would then auto-generate any unrepresented delegation methods
which effectively forward the trait-based method calls for S to those
implemented by the f field for that trait:

	impl TR for S {
		fn foo(&self) -> u32 {
			self.f.foo()
		}
		fn bar(&self, x: u32, y: u32, z: u32) -> u32 {
			self.f.bar(x, y, z)
		}
		// ...
	}
	
Attractively, this proposal supports enumeration of the specific
methods to auto-delegate (rather than `*`, which obtains them all).

That said, I wish requesting delegation was less wordy, not so trait-oriented,
and did not actually generate delegation methods.
The proposal describes other limitations to the technique and
recommends future improvements.

## D: alias this ##

Unlike Rust, the D language does offer support for class and interface inheritance.
However, it also offers an unusual 
["aliasing" capability](https://dlang.org/spec/class.html#alias-this)
which tells the compiler to treat use of a struct or class value as if it were acting
on one specific field.

In a struct declaration, a field is marked with `alias this` to "subtype" it:

	struct S
	{
		int x;
		alias x this;
	}

Any methods or operations invoked on `s` which are not implemented on `S`
are statically resolved to corresponding methods on `x`. 
It also treats an unadorned `s` as if it were `s.x`.

By describing it as aliasing, rather than delegation, 
D's approach offers a different mental model to explain what we are doing.
However, the functional result is largely similar to inheritance and delegation.
The implementation is closer to inheritance in that no extra delegation methods are being created,
which is easier on the compiler and maybe throughput.
Two key limitations of this technique, as currently implemented, are that
only one field may be aliased and the aliasing is total.
There is no way to alias only some of `x`'s capabilities.

## Go embedded types ##

Go does not support traditional inheritance. 
In its place, it offers an intriguing feature called
[embedded types](https://go101.org/article/type-embedding.html).
While declaring a struct's fields,
specifying only a type name creates a field of that name
which is given additional capabilities:

	type Singer struct {
		Person // extends Person by embedding it
		works  []string
	}

Any methods invoked on a singer, not implemented by Singer, will
be statically resolved into a direct call to Person's methods.
Implicitly, this can also resolve to a `*Singer` method. 
Similarly, unresolved field requests on a singer are treated as if
on a Person (e.g., `gaga.Name` may be used in place of `gaga.Person.Name`).

Usefully, a struct may list multiple embedded types.
Go applies several rules to determine when
overloaded names are treated as shadowing (choosing the resolution)
or colliding (generating an compile error).

Go offers us yet another way to understand what is happening.
Instead of inheriting, delegating, or aliasing, 
we are "embedding" another type's capabilities.
And yet, functionally we obtain more or less the same result as inheritance.

I like these aspects of Go's and D's technique:

* Aliasing/embedding is effectively
  ordinary field composition with special marking where forwarding is desired.
* Instead of creating forwarding methods, automatic
  field/method "forwarding" is compile-time resolved at the call site.

Unfortunately, Go/D's forwarding is all or nothing. 
One cannot enumerate which methods to forward.
	
## Namespace Folding for Types ##

The last proposal I want to explore is inspired by
how some languages can fold names from one module
[namespace](https://en.wikipedia.org/wiki/Namespace#Use_in_common_languages) into another.
In C++, for example, it can get tiresome to
prefix every standard library name with `std::` (e.g., `std::unique_ptr`).
This can be avoided with [a `using` statement](https://en.cppreference.com/w/cpp/language/namespace#Using-directives):

    using namespace std;
	
Effectively, this "folds" all the names found in the `std` namespace
into the current namespace. Thereafter, the code can use
the name `unique_ptr` and the compiler knows this refers to `std::unique_ptr`.

When only a few names are needed, it can be prudent to fold in only those names:

	using std::unique_ptr, std::shared_ptr

What if we were to use namespace folding to implement inheritance?

	struct Spaceship
	  ship Vehicle uae *
	  engine ImpulseDrive use thrust, engageWarp
	  crewMax i32

With something like this syntax, we fold all names from Vehicle,
as well as the `thrust` and `engageWarp` names from ImpulseDrive,
into Spaceship.
It improves on Go's and D's approach by offering precise control
over what names we inherit.
If names collide across namespaces, we trigger a compile error.
We can even minimize this likelihood by allowing name aliasing
(e.g., `thrust as setThrust`).

This one weird trick may make doctors angry,
but it gives us all of the convenient code reuse capability
that people turn to inheritance for, plus a bit more:

* We once said inheritance was composition plus some magic.
  This approach takes this concept literally.
  composition of fields and methods is exactly the same with or without inheritance.
  The magic is simply name folding.
* Multiple (stateful) inheritance is easily supported, safe, and 
  non-problematic. There is no 
  [diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem).
* Method overriding can be easily handled by not folding in
  any `*` methods names which the outer type implements.
* Because this approach is essentially name elision sugar, the compiler
  directly resolves dot-based field and method access statically.
  No extra delegation methods need be generated by programmers or the compiler.
* We can also choose to fold in default behaviors from traits/interfaces
  via a similar technique.

The ability to precisely specify which names are inherited
offers a nice solution to the three complaints about simple inheritance
I listed at the start of this post.

## Subtyping ##

Remember the confusion over whether inheritance reflects an
"is a" (subtype specialization) or "has a" (possession) relationship?
The truth is that it always reflects a "has a" relationship
whenever it composes in new fields.
The above example thus illustrates that a Spaceship *has* an Engine,
even though we folded in a couple of names.

However, whenever we fold in *all* the names of a base
type into a new type (e.g., Vehicle into Spaceship),
we are effectively saying we can treat the new type as
a subtype of the base type (a Spaceship *is* a Vehicle).
Likewise, whenever a concrete type implements (conforms)
to an interface, we can say it is a subtype of the interface's abstract type.

It is important to know when a subtype relationship exists between types,
as we then know that substitution and coercion of values 
from one type to another can take place,
constrained, of course, by variance and the rules of the language.

## Summary ##

We started out enumerating six capabilities supported by traditional inheritance.
After all these many words, what do we have left for Cone to support?

* <s>**Inversion of Control**</s>. Gone. Use dependency injection instead.

* <s>**Protected Access**</s>. Gone.
  Design your public interfaces wisely and parsimoniously,
  and use type and module namespaces to further isolate implementation details.

* **Interfaces**. This MVP is promoted to its own abstraction (traits)
  and given additional powers<sup>1</sup>: 
  
  - it can bound (constrain) generics,
  - a struct may implement multiple traits,
  - field-based row polymorphism, and
  - structural (vs. nominal) compliance.
  
* **Composition**. Types inheriting types compose
  their inclusion in exactly the same way as for non-inherited types.
  
* **Delegation**. Namespace-folding supports the absorption and
  customization of another type's fields and methods.
  Having precise, simple control over what is absorbed
  makes both "has a" and "is a" relationships possible
  across multiple types without the risk of name conflicts.
  
* **Subtyping**. A type is a subtype of some base type when it
  complies fully with the base type's interface, which
  can happen when it folds in all names.
  
What we have may not be the rich, traditional form of inheritance
commonly found in static languages, but it is
a legitimate form of inheritance all the same.
I am calling it "delegated inheritance"
to signal that we now have a simpler, safer
inheritance abstraction that uses composition and
a more convenient and efficient form of delegation
to cure the many ills that have plagued us for decades.

----------------------

<sup>1</sup>Probably worth covering in more detail in a separate post.