---
title: "Modules vs Types"
date: 2020-04-13T10:40:42+10:00
draft: true
---

First off, I want to ignore (for now) the issue of source file separation. It is important, of course, but I think too many schemes (Rust) confuse the logical issues that modules/types are intended for vs. the source code divisions
Secondly, I will argue that both modules and types have a responsibility for managing namespaces: allowing unique and qualified public vs. private names
Further similarities:  both modules and types are able to package together state (variables/fields), functionality (functions/methods) and "sub" types (types defined within another type or module)
So, then, what distinguishes modules and types?
One of them is the stratification issue mentioned earlier:  modules and types can define named types, but types cannot incorporate a module within it. I believe this to be true, but explaining why requires digging deeper
Oh - I am also a believer in the idea that modules dependencies should be representable via a DAG, with cycles specifically disallowed (Rust violates this)
Ok - so here is the deepest difference I have found between modules and types:  module state is entirely represented in the global memory region, gathered together by the linkeditor and accessed effectively by a hard-coded memory address.
Notably, all functions in a module do not have a "self" parameter to access the module's state. They do so via direct access to the global variables by virtual address.
A type, by contrast, represents state that is at any arbitrary place in memory (including global), and its methods all have self that provides access to the state as either a value or (more commonly) by reference
Those implementation differences are huge, particularly in a systems language
And part of the implication of them is that modules are essentially understood to always be singletons (state-wise) from an execution point-of-view. Whereas types generally are not, they are understood to work with data instances, where all instances have the same properties.
So then, if this distinction holds true, then the questions I have been asking are whether the same facilities we give to types to parameterize (and monomorphize) them with generics and abstract their essential API (using traits/type classes/interfaces) should also be valuably provided to modules
In essence, if modules are a distinct language mechanism for destructuring and componentizing programs/libraries, should we apply bounded universal and existential mechanisms to them, must as we do to types.
And I think the answer is "yes", that doing so improves modularity of our program's large-scale components
because it allows us to define an interface for a module that multiple implementations can satisfy, thereby allowing one module to make use of any other module that complies with the module interface it requires
and it also allows us to define the logic for a module generically and then instantiate multiple versions of it parameterized by type and by module interface.

types/modules as namespaces

Going up:  a module A imports B and "elevates" via use all publics in B to be publics in A. If module Z imports A and elevates all its publics, should B's be elevated too, or only the one's known by A (option on which)
I think generally, sharing names going downward needs to be done via parameters (including siblings), much like the way that a function cannot see anything about the state, etc. of its calling function
my fear is that if I break away the child's isolation from its parent, I destroy decompositional modularity in a meaningful way
and my sense that parameters are exactly the way we generally agree to inject dependencies
which is, for me, another argument for allowing modules (vs. types) to be parameterized
Because then I can safely inject dependencies without breaking the modular isolation of a component that I may wish to reuse in other contexts.

Instancing/ Modules as singletons

Module[i32] and Module[f32] are both statically instanced, with separate state. Make sense?
If a program uses both of them, then the linkeditor includes both of them: state and functions, etc.
I think the more common use case is the ability for a library to be monomorphized once according to the needs of the program that is stitching together the libraries it needs. So more often than not, a program will only have one instance of a module it is using.
but that module has been customized specifically for the program's needs in a static way
Perhaps because of platform differences, Windows, Linux, etc. Perhaps because of the GUI it uses, perhaps because of the data structure it makes use of, etc.
Can you accompish this sort of customization entirely using types, and indirect references, sure, but sometimes you pay a performance price for doing so. If modules also had this capability you might reach for it instead.
Besides hammering out a good distinction between modules and types, the question about the value of parameterizing and abstracting modules is the other hard question. There is just enough potential promise there to intrigue me, but I have no sense right now on how often one might want to reach for it
because nearly no language even offers this capability, our minds automatically know how to solve it with types
With relatively small programs, that's probably fine.
Where it gets interesting is when programs/libraries get MASSIVE
LLVM, for example, plugging in codegen across multiple target platforms. That is singletons (modules) not instances (types)

parameterization, abstraction and virtual dispatch

https://futhark.readthedocs.io/en/latest/language-reference.html#module-system
https://mirage.io/ unikernals via OCaml module system
https://arxiv.org/pdf/1512.01895.pdf Modular implicits
https://people.mpi-sws.org/~dreyer/papers/mtc/main-long.pdf
