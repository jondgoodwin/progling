---
title: "Favor Composition Over Inheritance?"
date: 2019-07-18T06:07:56+10:00
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
  - "C++"
---

I need to decide what sort of inheritance capability Cone will offer.

None! I can hear some of you insist.

"Inheritance has recently fallen out of favor as a programming design solution,"
claims the [Rust language book](https://doc.rust-lang.org/book/ch17-01-what-is-oo.html).
"Favor object composition over class inheritance," recommends the
*Design Patterns* book in 1994. 
"[Inheritance is Evil](https://codeburst.io/inheritance-is-evil-stop-using-it-6c4f1caf5117)."
insists Nicolò Pignatelli.
It is not hard to find cogent, hard-hitting critiques of inheritance,
complaining about costs incurred from fragile base classes, excessive coupling,
and broken encapsulation.
If you have ever found yourself wrestling with the bewildering complexity 
of multi-layered, interdependent class hierarchies in C++, Java or C#,
you surely understand the desire to lay waste to inheritance altogether.

And yet, inheritance is an incredibly popular feature in programming languages.
Nearly every one of the most commonly used languages supports inheritance
(C being a notable exception). 
It's not just legacy languages;
one can find some form of inheritance in many rising stars, such as
Swift, Go (embedded types), and D.
Even Rust, which claims not to support inheritance, offers a limited
form of it and seriously entertains prominent proposals
to improve on this support.
Is this just historical inertia, or does it reflect some legitimate programmer requirement?

Over the next few posts, I want to forward the argument that sensible and fruitful
conversations about inheritance are difficult and painful because people often treat it
as if it were one monolithic abstraction against which simplistic conclusions can be drawn.
By breaking apart the complex, interwoven features of inheritance
into their distinct mechanisms, I believe we will find it easier to talk about.
More importantly, doing so will make it easier 
to assess which of its mechanisms are worthwhile to keep,
and which features we are better off supporting in some other way.

## What is inheritance? ##

Wikipedia defines 
[inheritance](https://en.wikipedia.org/wiki/Inheritance_(object-oriented_programming))
as "the mechanism of basing an object or class upon another object or class, 
retaining similar implementation."
This definition largely evokes its essential purpose (hinted at by the name),
but it leaves a wealth of mechanical details to the imagination.
To get serious about this, we need a categorical model for organizing its key capabilities.

For the purpose of these posts, let's narrow our gaze
to focus only on the forms of inheritance that might be offered
by a systems programming language (as Cone is).
This means we won't be talking about inheritance features
found in dynamically-typed languages, such as prototypical inheritance,
first-class classes, message-passing dispatch, Ruby mixins, or
[C3 linearization](https://en.wikipedia.org/wiki/C3_linearization).

Here is my proposed model for categorizing the inheritance features
supported by C++ (which offers the most comprehensive inheritance capabilities):

1. **Composition**. A derived class incorporates the state of its base class(es).
2. **Polymorphic Interfaces**. Dynamic dispatch applies to all classes implementing a base class's set of method signatures.
3. **Delegation**. Method calls to a derived class are satisfied fully or partially by base class methods.
4. **Subtyping**. Where variance allows, a derived class may be used where a base class is expected.
5. **Protected Access**. Derived classes may access protected implementation details in base classes.
6. **Inversion of Control**. Generalized base class logic is specialized by calling derived class methods.

These categories are still broad.
As each is discussed, we will look at more detailed variations.

## Inheritance Begins With Composition ##

The *Design Patterns* book presents composition and inheritance
as if they were two very different techniques for reusing functionality.
This is potentially misleading.
It is conceptually more accurate to view inheritance as
pure composition plus "extra magic" (points 2-5 above).

This is easily illustrated. Consider these C++ classes.

	class Base {
	  int a;
	  int b;
	};
	class Derived : Base {
	  int m;
	  int n;
	}

What the compiler understands is that the Derived class's state looks like this:

    class Derived {
	   Base base;
	   int m;
	   int n;
	}

A derived class effectively includes the state of any of its base classes,
much as if each of their state had been explicitly specified compositionally as fields.
Indeed, this intuition corresponds to how *Design Patterns* describes composition:
"[its] functionality
is obtained by assembling or composing objects to get more complex functionality.
Object composition requires that the objects being composed have well-defined
interfaces."

Examined from a purely compositional point-of-view, multiple inheritance of state
poses no real design headaches, any more than it does for a struct to have
several fields of different types.

## What's Next? ##

The next two posts take a deeper dive into the "extra magic" of inheritance:

* [Disinheriting Abstract Classes](/post/disinheriting-abstract-classes/)
  dissects the problematic role of abstract classes
  in supporting **Polymorphic Interfaces** and **Inversion of Control**.
  It also re-examines the issues that arise from **Protected Access**.

* [Delegated Inheritance](/post/delegated-inheritance/)
  explores the value of, and best approach for, gracefully supporting **Delegation**.
  It also address **Subtyping**.

