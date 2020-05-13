---
title: "Weakening Cycles So That Turing Can Halt"
date: 2020-05-11T16:27:45+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "General"
  - "Memory"
  - "Types"
tags:
---

It is notable how often paradoxes arose in the historical journey 
that led to the [Rise of Type Theory](/post/rise-of-type-theory).
Resolving Russell's paradox led to his theory of types.
Gödel's Incompleteness theorems rested its proof around a variation of
the [Liar Paradox](https://en.wikipedia.org/wiki/Liar_paradox).
Church and Turing reformulated this result using their variant notations.
Intuitionistic type theory was deliberately designed to avoid these paradoxes.
And so it goes.

As an undergraduate, I had the pleasure of studying Gödel's
ingenious proof as part of a course on Predicate Logic.
Later, I had the further pleasure of reading Douglas Hofstadter's
delightful book [Gödel, Eschel, Bach: an Eternal Golden Braid](https://en.wikipedia.org/wiki/G%C3%B6del,_Escher,_Bach),
which inventively ruminates on themes related to Gödel's result.
All this left me wondering (as it does many), is there an underlying pattern
to the trap that Gödel carefully laid down for us?
More importantly, how can we avoid this trap without giving up?

My initial sense was that the presence of two conditions
open us up to danger.
The first is recursion, the ability for things to reference themselves
directly or indirectly. The second is "self negation" in the presence of recursion:
when following the recursive path contradicts earlier-established truths.
The door to paradox seems to open when both are permitted.

In a recent flash epiphany, I came to understand that the problematic pattern 
is more basic and universal than this.
Far from being just a theoretical barrier, it is an omnipresent
challenge we must regularly overcome across many interesting programming domains.
And we not only run into it often, we regularly overcome it as well,
and do so without having to give up recursion!
Best of all, there is a pattern to how we overcome these "paradoxes".

This post begins by demonstrating these patterns using arithmetic functions.
It then illustrates how the same principles apply to memory management and deadlocks.
The final section shows how these principles have been invaluable
in helping us evolve the power of our theoretical tools.

## Recursive Arithmetic ##

Turing's version of a Gödelian paradox is called the 
[halting problem](https://en.wikipedia.org/wiki/Halting_problem):
Can we determine whether any given program will finish running at some point,
or will it run forever? Turing proved that no general algorithm
can be formulated that can answer this question across all possible programs.

Once we re-frame the challenge in terms of termination,
it is obvious that any program without loops or recursion will terminate,
since a program consists of a finite number of operations that are all executed
exactly once, in order<sup>1</sup>:

    imm x = 5
	imm y = x * 3
	x = x + y

Using loops or recursion, we can create programs
that will not terminate (at least for some values):

    fn lookForZero(n u32):
	  if n > 1:
	    lookForZero(1)
	  elif n == 1:
	    lookForZero(2)

Pass 0 to this function, and it stops immediately. However, any other
value will cause it to run forever, flipping between 1 and 2.
It is like a hamster getting on a wheel that never stops,
going round and round forever.

Recursion is not always problematic, however.
This function always terminates:

    fn factorial(n u64):
      if n < 1:
        return 1
      else:
        return n * factorial( n - 1 )

This is what we want! Recursive functions (or loops) that are
guaranteed to halt at some point, traversing the cycle
a finite number of times. What extra quality needs to be added
to recursive cycles to guarantee termination?

As factorial demonstrates, an effective strategy involves weakening cycles so they are guaranteed
to "break" at some point. How does factorial do this?
The "exit" from the cycle is based on a number ("n") which is always getting 
arithmetically smaller from its starting point, until it inevitable reaches its smallest
possible terminating value (0). One helpful way to visualize this terminating recursion is
with a spiral whose line must terminate inexorable at the center.

Before showing the usefulness of this weakened cycle pattern to other problems,
I want to make it clear that [termination analysis](https://en.wikipedia.org/wiki/Termination_analysis)
is significantly more complicated (and fascinating) than a binary choice
between guaranteed termination and guaranteed non-termination.
This example illustrates some of the uncertainty that lies between these extremes:

    fn f(n Bignum):
      while n > 1:
        if n % 2 == 0:
          n = n / 2
        else:
          n = 3 * n + 1

For every number tried so far, this function terminates.
However, no one has proven it always will for every value of *n*.
This challenge, the
[Collatz conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture),
has a fascinating history of its own, including an intriguing connection to
[John Conway's Fractran and Minsky Machines](http://raganwald.com/2020/05/03/fractran.html).
It is noteworthy to point out that, even though Conway never proved the Collatz conjecture,
he did prove that a generalized version of it is undecidable.

## Memory Management ##

As with Euclid's [fifth postulate](https://en.wikipedia.org/wiki/Parallel_postulate),
the holy grail search for the perfect automatic memory management strategy
has led so many to ignoble ruin. Rust's static single-owner strategy
seems so attractive, and yet it severely restricts the data structures we can build.
Arenas, used in inappropriate circumstances, can be unacceptably leaky.

When programs need multiple references to point to the same object, and we
cannot statically determine which alias will be the last to expire, we require
a memory management strategy that can determine this at runtime,
so that allocated memory can be safely recycled.
And thus, to satisfy our need for large, transient, complex, shared "ownership" data structures,
we are forced into reaching for some variation of runtime garbage collection,
based on reference-counting, tracing, or both.

So what does all this have to do with cycle weakening?
It's pretty straightforward:
when you allow multiple references to the same object,
reference cycles become possible, where one object references another, which references
another, eventually returning via some reference to the original object.
It is well known that reference counting struggles
in the presence of such [reference cycles](https://en.wikipedia.org/wiki/Reference_counting#Dealing_with_reference_cycles). 
These cycles effectively keep every object in the cycle
alive forever. And so memory leaks.

An effective solution to leaky cycles involves weakening every cycle, literally.
When you ensure that every reference cycle contains at least one "weak" reference,
memory leaks are entirely avoided. Unlike regular references, weak references
do not have the power to keep the object it points to alive.
This weakening is enough to allow any cycle to be broken and, ultimately, discarded.

Tracing garbage collectors use a different strategy
to weaken reference cycles: [color marking](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Tri-color_marking).
The collector begins with a root set of references,
and traces the path of all references to all objects that
need to stay alive. When references are cyclic, this poses a termination challenge,
as a dumb collector would chase around these references forever.
Smart collectors drop breadcrumbs as they trace, coloring every reference
as it is traversed. A reference starts off white, turns gray when we know it requires tracing,
and then becomes black when traced.
When the collector runs across a colored reference,
there is no need to trace it again. The ability to mutably color references
weakens cycles, and thereby guarantees termination.

## Deadlocks ##

Deadlocks are another intractible quagmire for program design.
When we have multiple independent threads that require shared, mutable
access to the same resources, we need to synchronize access to these
resources to avoid data races. Often, this involves the use of locks.
A thread waits until it can acquire an exclusive lock,
reads or mutates the resource, and then releases the lock as quickly as possible.

This is straightforward when only one resource is in contention.
However, we can run into real problems when threads need to acquire locks
on multiple resources at the same time. What if thread A acquires a lock to 
resource X and then tries to acquire a lock to resource Y,
but resource Y has been locked by thread B who then wants to acquire resource X?
Both threads will wait forever for the other to free the resource they need!
This is a straightforward example of a 
[deadlock](https://en.wikipedia.org/wiki/Deadlock).

What we have created is a resource dependency cycle, one that
can never terminate unless we weaken the cycle somehow.
In 1965, Edsger Dijkstra formulated a version of this challenge
in terms of the [Dining Philosophers](https://en.wikipedia.org/wiki/Dining_philosophers_problem).
Each philosopher needs to eat their bowl of spaghetti, but can only do so
using both forks placed on either side of their bowl.
The problem is each fork is shared with their neighboring philosopher around the table.
How do they arrange this wordlessly without starving?

Dijkstra's solution imposes a partial order hierarchy on the forks, 
the resources in contention. Each fork is numbered.
Every philosopher will only acquire their lowered numbered fork first,
followed by the higher numbered fork.
By forcing an acquisition order on contended resources,
we have effectively weakened the cycle of dependencies,
and thereby ensured that no philosopher is left endlessly waiting on
a required fork already acquired by other also-waiting philosophers.

It is worth pointing out that deadlocks may occur even in the absence of locks.
This is important because programming languages that make no use of locks,
such as Pony, can still experience deadlocks.
By carefully modeling each dining philosopher and fork using distinct actors
that synchronize by sending messages to each other,
it becomes clear that at some point the philosopher actors just end up
silently waiting on messages that will never arrive.
This state of waiting and never progressing forward
is as much a deadlock as contention over locked resources.
And thus, even in a language without locks,
we need to break deadlocks using some form of statically provable
weakening of resource dependency cycles.

## Type Inference ##

With this background in hand, we can now see a clear pattern in the
cycle weakening solutions that theoreticians have applied to encountered paradoxes:

- Russell addressed his [paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox)
  (if R is the set of all sets that are not members of themselves,
  R can neither be a member of itself, nor can it not)
  by formulating his ramified theory of types, which weakens allowed relationships between sets. 
  Essentially, this theory of types establishes a partial order on entities. 
  An entity of some type can only be built up from entities of a lower type,
  much like the solution used by factorial and the dining philosopher problem.
  Russell's axiom of reducibility is too stringent, as it effectively prohibits recursion,
  but it got the job done until more versatile approaches could be formulated.
   
- Alonzo Church invented 
  the [simply-typed lambda calculus](https://en.wikipedia.org/wiki/Simply_typed_lambda_calculus)
  to address the Kleene-Rosser paradox, which showed that certain systems of formal logic, 
  including Church's untyped lambda calculus, are undecidable.
  This too applies a Russell-inspired hierarchy of types to
  weaken cycles in the lambda calculus.
  
- The intuitionistic type theory that arose from Brouwer, Heyting, Kolmogorov,
  and Martin-Löf is built on a structure carefully designed to avoid
  paradoxical constructions. Law of Excluded Middle. arithmetic.
  
- 1ML and undecideable type inference


-----

<sup>1</sup> You did not think I was going to write a post and not mention
[Cone](http://cone.jondgoodwin.com/), did you?
