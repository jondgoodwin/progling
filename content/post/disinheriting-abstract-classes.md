---
title: "Disinheriting Abstract Classes"
date: 2019-07-18T12:14:51+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "inheritance"
tags:
  - "C++"
  - "Cone"
---

Anti-inheritance advocates are likely to enthusiastically support this post.
It promotes the most useful feature of traditional inheritance (Interfaces),
turning it into a more valuable abstraction that is largely independent of inheritance.
It discards two other traditional features of inheritance,
Inversion of Control and Protected Access, as both unnecessary and dangerous.

Let's examine each in turn...

## Interfaces ##

An interface is "an abstract type that contains no data but defines behaviors as method signatures." 
([Wikipedia](https://en.wikipedia.org/wiki/Interface_(computing)))
Increasingly, there has been a growing recognition
of the importance of interface-based architectural design.

Many older languages offer interface capability as
part of the inheritance mechanism, supporting them using 
[abstract classes](https://en.wikipedia.org/wiki/Abstract_data_type).
However, this connection is unnecessary because, in their purest form,
no inheritance is actually taking place.
Derived classes often obtain no implementation information from interfaces.
Furthermore, it turns out that 
interfaces are useful for more than virtual dispatch.
They can also play a valuable role as generic constraints.

These considerations are why we increasingly see the rise of "class-less" 
interface abstractions cropping up across many languages:
protocols (Swift), traits (Rust), interfaces (Go), concepts (C++),
and contracts (Go). Instead of "inheriting" an interface,
we talk about a concrete type conforming to or implementing an interface.
Go goes even further: concrete types need not explicitly implement an interface
to conform and take advantage of it.

Separating the interface capability into its own abstraction comes with no obvious
downsides. Not only does doing so making it more broadly useful,
it also underscores the fact that interface-based polymorphism
(rather than inheritance)
is often all you really need to build well-architected libraries and programs.
The growing evidence for this has deepened many people's belief
that inheritance offers no benefit, only pain.

## Inversion of Control ##

Let's switch gears and focus on a very different inheritance capability:
the powerful (and problematic)
ability for a concrete base class to access the state and behavior
of its derived classes.

A base class accomplishes this by calling virtual methods it defines,
but which it anticipates the derived class 
will overwrite with its own implementations.
These overwritten methods have full access to the derived class's
state and methods. Thus, these overwritten methods become a
portal through which the base class may access any derived class's 
state and behavior.

Although inheritance is often described as a mechanism for code reuse,
this goes way beyond the simple idea that a derived class
is simply reusing the base class's methods and data.
Remember how we said that inheritance begins with composition?
That a stateful base type can be understood as one of
many fields in the state of its derived types?
What we see happening here is that *one* of the struct's fields 
(and its methods) obtains the superpower to
see and control the *entire* struct's state and behavior.
This is like giving the windshield wipers the power to steer the car,
or launch nuclear missiles!

As this power is increasingly exploited,
control flow paths become more complicated (and harder to follow)
and classes become more tightly coupled in mysterious ways.
When using a base class's methods, we lose clarity on
whether it will act only on its own data, or will trigger
logic that acts on its owner's data or its owner's owner's data.
New requirements can often worsen this condition, by deepening
class hierarchies and interdependencies, thereby worsening
complexity and coupling to byzantine levels,
especially when stateful flags are introduced to re-route control paths.
No wonder programmers have learned to fear breaking
complex, inheritance-based class hierarchies!

The *Design Patterns* book calls this form of 
[inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control)
the [template method pattern](https://en.wikipedia.org/wiki/Template_method_pattern).
The template method pattern exploits inheritance to make this
form of inversion of control implicit and (arguably) too-easy-to-implement.
Under the covers, inheritance performs invisible
downcast and upcast pointer magic so that every method's logic can depend on
`this` always pointing at the right piece of state.

There is another, better way to implement this pattern:
use [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection).
The base class's virtual method signatures become a Service interface.
The core logic in the base class assumes the Client role,
invoking the helper logic using object calls that comply to the Service interface.
The helper logic in each derived class is repackaged as a 
specialized Service that implements the Service interface.
Finally, a separate entity, the Injector creates
the services and client, and then injects the appropriate services
into the client at run-time.

Dependency injection is better because it offers a more versatile, transparent, modular design.
Since dependency injection doesn't use inheritance, 
our complex class hierarchies flatten considerably.
State dependencies are now explicitly marked, so when we call
another class's method, we can clearly see what data scope it might act on.
Specialized services are no longer tied to one specific base class
(or worse, to its particular *protected* implementation logic!).
As requirements evolve, our design can accommodate change with less breakage.
Changes to the Client (the former base class) no longer carry the risk
of breaking the services.
Services can be tested separately from the Client.
We can completely replace the client with a rewritten one,
or implement multiple different clients that use available services in different ways.
The code is easier for new people to learn, one piece at a time.
And, invisibly, we reduce the burden on the vtables to carry runtime
information needed to correctly (and implicitly) downcast `this` pointers
(Section 10 of [*C++ Multiple Inheritance*](https://pdfs.semanticscholar.org/35a5/01c412162dba797ce75f857ce58bc55d211c.pdf?_ga=2.115934534.1787507037.1563320962-1987026637.1563320962)).

I want to be careful here not to oversell the benefits of dependency injection.
We have not eliminated coupling. That's not possible.
What we have done is to *lessen* it, by forcing all communications
to go through public interfaces.
There is still a risk we may have to alter these interfaces over time.
When we do, this change will ripple out to every piece of logic that depends on them.
However much work this requires, it is still likely less risky than 
doing so when multiple layers
of class logic are interwoven with hidden spaghetti dependencies.

Remarkably, changing our code from the template method pattern
to dependency injection doesn't appreciably grow our code size or worsen performance.
This means our design architecture gets significant 
flexibility and stability benefits at nearly no added cost.

Given this assessment, the choice is simple:
prohibit stateful base class methods from accessing derived class
state or methods. 
This is easily accomplished by eliminating virtual, overrideable methods.
If a derived method implements a method by the same name
(and it might for the sake of interface polymorphism),
the base class's invocation of a method by that name
will call the base class's method, and never the derived class's version.
Faced with this limitation, programmers wanting inversion of control
will learn to reach for dependency injection instead.

## Protected Implementation Access ##

Languages with inheritance typically offer a third isolation attribute
called *protected*, which lies somewhere between *private* and *public*.
Base class methods and fields marked as *protected*
are *public* to derived classes but *private* everywhere else.

*protected* was added to C++ in 1989, after documented complaints that
"inheritance breaks encapsulation".
These complaints resulted from noticing that programmers were
making fields public that should have been private,
simply because derived classes need access to these fields
to specialize the behavior of base classes.
I suspect this need was particularly strong when class hierarchies
implemented inversion of control, as specialized methods in derived classes
need direct access to state data and methods that we really don't want
to make part of the public interface of the overall object.

However, protected access feels unsafe to me.
We make fields private in large part so that we can
guarantee that, so long as every type method protects the invariants,
their state will always be valid.
If private fields are upgraded to protected,
we place the burden of protecting the invariants
on every subtype's implementation.
This increases risk significantly.

Were we to take away *protected* isolation, would a bunch
of fields and methods we want private be forced to go public?
I don't think so. Prudent design of a type's public interfaces
can go a long way towards ensuring that only the data that is
needed is exchanged, just when it is needed, while always
protecting invariants.

If, after doing that, we may still want some way to isolate
certain "internal" types so they are only 
visible and usable to the types whose access we want to be more public.
To accomplish this,
we simply wrap these internal types as private members of
the visible type's (or module's) namespace.

Given our decision to abandon inheritance-based inversion of control,
I believe the questionable need for giving
subclasses protected access to base class constructs,
is more than negated by the danger to invariants.
Accordingly, Cone will restrict derived types
to accessing only the public fields and methods of the types it inherits from.
This decision brings another important benefit:
The potential risk of complex and fragile coupling between base and derived classes 
is reduced further.

## Interim Summary ##

In this post, we have taken the bold step of removing three
of inheritance's traditional capabilities from Cone's design.
One, interfaces, has been promoted to its separate abstraction (traits)
and given more powers.
The other two, inversion of control and protected access,
have been excised altogether.

Excising these features places the following restrictions
in the relationship between a base class and a derived class.

* A base class may not invoke methods of any derived class
  (nor access any of its state).
* A derived class may only access the public methods
  and fields of its base class(es)
  
We lose nothing of consequence by these restrictions,
because we can still implement inversion of control using dependency injection
and we can use carefully-designed public interfaces 
and type/module namespace isolation to gain
ready, performant access to needed information and capabilities.

The benefit is massive. Nearly every complaint about inheritance
has been significantly mitigated:
Hierarchies flatten. Coupling reduces dramatically.
Base classes are much less fragile.
And, encapsulation equalizes and invariant enforcement strengthens.

Although coupling has not been eliminated,
it *has* been isolated to the public interfaces.
When changes are required, we can quickly identify with confidence
what is affected and what needs to be tested.

Have we gutted inheritance completely by these changes?
Not even close!
In the [final post](/post/delegated-inheritance/), 
we explore the one traditional inheritance
capability that is worth preserving: automatic delegation.