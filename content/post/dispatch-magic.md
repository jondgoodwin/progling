---
title: "Dispatch Magic and Concurrency"
date: 2021-10-10T13:40:17+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "General"
  - "Concurrency"
tags:
  - "Cone"
  - "Pony"
  - "Smalltalk"
---

I have long enjoyed the convenience of the "dot" [dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch) syntax:

    instance.log("Ethereal cognistrands do not support quantum entanglement")
	 
The magic of [Universal Function Call Syntax](https://en.wikipedia.org/wiki/Uniform_Function_Call_Syntax)
makes this sugar for a function call:

    log(instance, "Ethereal cognistrands do not support quantum entanglement")
	
The broad popularity of method dispatch is driven by these benefits:

- [**Ad hoc polymorphism**](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)). 
  When used in conjunction with 
  [method overloading](https://en.wikipedia.org/wiki/Function_overloading),
  we can reuse the same easier-to-remember semantic names across multiple types and parameter configurations,
  with minimal-to-no ambiguity on which implementation to use.
- [**Method chaining**](https://en.wikipedia.org/wiki/Method_chaining)
  makes it easy to build a data transformation pipelines.
  C# Linq and Rust's combinators are powerful examples of this.
- [**IDE auto-completion**](https://en.wikipedia.org/wiki/Intelligent_code_completion) 
  makes code typing faster and more accurate,
  because the code editor makes it easy to select the desired method,
  narrowed down based on the object's type.
  
This post reviews the underlying magic of dispatch by:

- Summarizing the implementated variations of method dispatch supported by different languages.
  Some languages support multiple approaches, but no one language supports them all.
- Exploring how we can extend dispatch beyond what is broadly
  familiar in today's language, particularly to facilitate convenient concurrency.

## Function Call Dispatch Mechanisms ##

There are fundamentally three dispatch mechanisms broadly implemented by 
many programming languages. These are the names I will use for them:

- **Static**
- **Virtual**, sometimes described as dynamic dispatch with early binding
- **Dynamic**, sometimes described as late binding or message-passing

Single-dispatch is the most common, where the type of a single (first) object is sufficient
to determine which function to dispatch to.
However, there are definitely situations where multiple dispatch is useful,
when we want to select the method based on the types of multiple objects.
For example, multiple dispatch is useful when we want to calculate whether two
meshes have collided, when they are modeled using different primitive geometries
(e.g., rectangular, sphere, cylinder, or plane).

### Static Dispatch ###

With static dispatch, the compiler knows exactly which method to call, based on the concrete
type of the object. 
Static, multiple dispatch comes for free when a language supports method overloading,
as method selection uses parameter arity and types to select the right method.

Static dispatch offers the best performance because dispatch is just a simple function call.
However, we cannot use static dispatch if we have to wait until runtime to determine 
the object's concrete type or the method implementation associated with a method name.

### Virtual Dispatch ###

Virtual dispatch is valuable when we don't know the object's concrete type until runtime.
This happens when we take advantage of subtype polymorphism, where an object's
type is an abstract supertype that generalizes the same-named and signatured methods across multiple concrete subtypes.

Virtual dispatch works by looking up the concrete type's method implementation
within the the appropriate compiler-generated vtable, and then calling the method indirectly.
When all method names and implementations are known at compile time (early binding),
the vtable can be statically generated as a simple array of function pointers,
whose lookup is performed by fast integer indexing. In effect, each abstract type's
method names are statically reduced to an integer index.

Languages differ on where to store the vtable.
In some languages (e.g., C++), one or more vtables are stored as part of the object's state.
In other languages (e.g., Rust trait objects and Haskell typeclasses), 
the vtable is implicitly passed as part of the virtual function call alongside the object pointer.

There are distinct pros and cons to each approach: 

- Object-encoded vtables make possible
multiple inheritance (C++) and virtual object pointers are half the size.
- Call-passed (fat pointer) vtables dispatch more quickly, and free the object's underlying data structure
from having to know all supertypes it conforms to (required by Go's structural subtyping and Rust's type extensions).

Only a few languages (e.g., Common Lisp, Dylan and Julia) offer built-in support for virtual, multiple dispatch.
Its omission by other languages is usually the result of two factors:  the need for multiple dispatch is relatively rare,
and it is more difficult to implement than single, virtual dispatch,
since vtables are optimized for a single type.
When virtual, multiple dispatch is needed but not built-in, it can often be hacked by
hard-coding multiple stages of single-dispatch.

### Dynamic Dispatch ###

In dynamically-typed languages (beginning with SmallTalk), classes (or prototypes) are typically mutable, first-class objects.
This means that a class's methods can be added, deleted or changed through-out run-time.
This is colloquially referred to as ["monkey-patching"](https://en.wikipedia.org/wiki/Monkey_patch).

When the methods (and implementations) are not known at compile-time, we cannot optimize dispatch via vtables.
To support late (runtime) binding, dynamically-typed languages embed a mutable hashed dictionary inside every class
(or prototype object). This dictionary maps a method name's symbol to the implementation logc for that method.

Using a hashed dictionary every time we dispatch is generally a lot slower than an integer-indexed vtable array.
To compensate for this throughput loss, some dynamic languages optimize the dispatch mechanism to use a cache
or profile-guided Jit compile to speed up when a given dispatch always goes to the same place.

It is rare for dynamic languages to support multiple dispatch. That said, dynamic typing makes it
really easy to hack in a multi-stage dispatch, given that parameters can hold values of any type.

The dynamic dispatch mechanism can vary in several ways across languages:

- [**Prototype vs. Class**](https://en.wikipedia.org/wiki/Prototype-based_programming). 
  In prototype-based languages like Javascript and Lua, the dictionary is built into every object and can also be
  inherited from its prototype object. Every object can effectively independently define its own dispatch patterns.
  In class-based languages, like Python and Ruby, every object belongs to a class which holds the dispatch dictionary.
  Classes also inherit dictionaries from their superclass.
- [**C3 linearization**](https://en.wikipedia.org/wiki/C3_linearization).
  When a language supports multiple-inheritance, the same superclass could pop up in different
  sections of the inheritance tree. The operative question becomes: what is the standard, unambiguous order to use when
  searching for the right implementation for some method? Some languages make use of C3 linearization to resolve this question.

Although most static-typed languages support virtual dispatch, a few (notably Objective-C and Swift)
support both dynamic and static dispatch.

## Intermission Observations ##

Before moving on to applying dispatch to concurrency, I want to make a couple relevant observations:

- **Method dispatch vs. function calls**. Some languages support either function calls or method dispatch, but not both.
  My preference is when a language supports both, especially when mapped by universal function call syntax.
  I prefer method dispatch when an operation is largely centered around some specific object's data,
  narrowly as an object we are working on or broadly as a shared context for a number of related operations.
  I prefer function call syntax when there is no obvious primacy of focus among a function's parameters.
  This would include use cases like a program's high-level procedures, anonymous functions, closures, etc.
  
- **Proxy objects**. All the dispatch mechanisms described so far use dispatch as an envelope for a stack-based
  function call. However, there are times we want to synchronously invoke functionality that lives outside
  the CPU's current thread, perhaps in another thread, process or even machine 
  (e.g., [Remote procedure call](https://en.wikipedia.org/wiki/Remote_procedure_call)).
  
    Often, invoking such functionality may not syntactically resemble a function call (or method dispatch).
    This can be ameliorated by creating an in-thread proxy object that acts as an intermediary to remote functionality.
    It accepts method dispatch calls, converts/serializes the parametric data and transmits it to the remote functionality.
    When it gets back a response, it converts it into data structures that can be returned to the caller.
  
## Concurrency and Dispatch ##

Are these three language-supported dispatch mechanisms sufficient for all common needs?
I believe not.

Let's explore how we can usefully apply dispatch syntax and semantics to concurrency scenarios, 
particularly channel-based communications and cooperative co-routines.

### Actors ###

The [Pony](https://www.ponylang.io/) language, like Erlang, is built around 
[actors](https://tutorial.ponylang.io/types/actors.html#sequential).
. 
Each actor is effectively a separate logical thread,
with its own execution stack and the ability to be scheduled on any available CPU core.
The work of an actor is driven by the messages on its 
[multiple producer, single consumer](https://en.wikipedia.org/wiki/Non-blocking_algorithm#Implementation) 
[queue](https://en.wikipedia.org/wiki/Circular_buffer).
When activated, the actor pulls off one message at-a-time, in order, from its queue
and completely performs the entire functionality atomically (with no locks or waits).<sup>1</sup>

Intriguingly, Pony uses dispatch syntax to send a message from one actor to another:

    actor.behavior(parameters)
	
Notice that Pony actors define behaviors, instead of methods.
There are two key difference between behaviors and methods.

- Behaviors don't specify a return type, because behavior dispatch is always a one-way trip.<sup>2</sup>
  It is an asynchronous request with no expectation of ever getting a returned value.
  If you do want to eventually get a response, you send along the calling actor as
  one of the parameters, in hopes that the dispatched-to actor ultimately sends back a behavioral message.
  
- Behavior dispatch does not map to a function call. Instead it packages up the parameters
  into a message which is then appended to the receiving actor's work queue.
  
I admire Pony's design approach to behavioral dispatch because of how it re-purposes
a very familiar dispatch syntax and semantics seamlessly on to a very different underlying dispatch mechanism,
based on true message passing. Along the way, it simplifies what the programmer needs
to specify with minimal loss of flexibility (compare it to how Go supports channel-based communications
between gothreads):

- Pony implicitly transforms an actor's behaviors and their dispatch signatures info a sum type
  that completely defines the format of all messages to that actor. 
  By contrast, Go requires the type for a channel to be explicitly declared as a data structure.
- The Pony compiler automatically converts behavior dispatch calls into message instances
  which are automatically appended to the receiving actor's queue.
  Go channel messages have to be explicitly created with constructors 
  and then explicitly transmitted to the channel as a separate step.
- Pony's support for multiple behaviors improves modularity.
  When a Pony actor receives a message as its next unit-of-work, it automatically
  decodes which behavior to dispatch a function call to.
  By contrast, Go requires the programmer to explicitly select the right logic
  to perform using a switch or interface-based dispatch.
  Go's lack of support for sum types makes this even more awkward, especially when each message type
  requires a very different signature of accompanying data.  
- Go's explicit channels are implicitly baked into every Pony actor. 

To summarize, Pony does not require the programmer to 
separately define and manage the channel, the message type, the gothread, message queuing and
message dispatch.
This simplifies the mental model and leverages familiar syntax in the process, 
making actors easier to plug together and use for the majority of use cases.

Inspired by Pony's design, let's see how these insights can be applied to other forms of concurrency.

### Cooperative co-routines ###

In their most general form, cooperative co-routines separate processing logic
across multiple co-routines, each of which has its own execution stack.
One co-routine can make a synchronous call to another. This suspends the calling co-routine
and passes parametric data to the called (but suspended) co-routine.
The called co-routine resumes execution where it left off, performs some work,
and eventually yields a result back to the calling co-routine.
The called co-routine is suspended and the calling co-routine resumes.

Co-routine are similar to other mechanisms, particularly:

- **Threads**. The similarity is that both have independent execution stacks.
  However, communications with threads are asynchronous (and one-way), whereas co-routines communicate synchronously
  (with two-way exchange of information).
- **Closures**. Simple co-routines can often be optimized into closures who effectively 
  represent the execution context using a pre-allocated shared data structure.
  This is a common technique used by languages like Rust for async/await.
  However, complex co-routines that make heavy use of recursion or have multiple yields at different places 
  deep within the execution stack
  cannot be (gracefully) transformed into closures.
  
Most languages that support co-routines use function call syntax to transfer control
from one coroutine to another (and yield to return). This is another way co-routines feel like closures.
When there is effectively one sort of logic we want a co-routine to perform,
this is the right choice of syntax.

However, sometimes we want co-routines to behave like actors,
supporting multiple behaviors that correspond to different parameter signature and modular logic paths.
An excellent example would be co-routines (or closures!) that act as state machines.

To me, it makes sense in these situations to communicate with multi-behavior co-routines
using behavioral dispatch syntax. Underneath, the underlying mechanism is neither a function call
nor message passing. The underlying mechanism is a context switch, swapping out the calling co-routine's
execution stack for the called co-routine's suspended execution stack.

## Summary ##

By taking concurrency into account, we can expand our original repertoire of three dispatch techniques
to five. Rather exciting!

What other dispatch mechanisms would you like to see built into programming languages?

------------------

<sup>1</sup>The [actor model](https://en.wikipedia.org/wiki/Actor_model) is a more accurate fit to Alan Kay's vision for object-oriented
message-passing than Smalltalk, in no small part because we actually are really passing messages
(rather than calling functions).

<sup>2</sup>This means we cannot chain together behavior dispatches the way we can method dispatches!