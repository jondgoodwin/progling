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
Church and Turing recapitulated this result using their distinctive formalisms.
Intuitionistic type theory was deliberately designed to avoid these paradoxes.
And so it goes.

As an undergraduate, I had the pleasure of studying Gödel's
ingenious proof as part of a course on Predicate Logic.
Later, my pleasure was doubled by reading Douglas Hofstadter's
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

This post<sup>1</sup> begins by demonstrating these patterns using arithmetic functions.
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
exactly once, in order<sup>2</sup>:

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
we are forced to reach for some variation of runtime garbage collection,
based on reference-counting, tracing, or both.

So what does all this have to do with cycle weakening?
It's pretty straightforward:
when you allow multiple references to the same object,
reference cycles become possible, where one object references another, which references
another, eventually returning, via some reference, to the original object.
It is well known that reference counting struggles
in the presence of such [reference cycles](https://en.wikipedia.org/wiki/Reference_counting#Dealing_with_reference_cycles). 
Such cycles effectively keep every object in the cycle
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
using [Dining Philosophers](https://en.wikipedia.org/wiki/Dining_philosophers_problem).
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
such as Pony, can still experience deadlocks.<sup>5</sup>
By carefully modeling each dining philosopher and fork using distinct actors
that synchronize by sending messages to each other,
it becomes clear that at some point the philosopher actors just end up
silently waiting on resource acquisition messages that will never arrive.
This state of waiting and never progressing forward
is as much a deadlock as contention over locked resources.
And thus, even in a language without locks,
we need to break deadlocks using some form of statically provable
weakening of resource dependency cycles.

## Type Inference ##

With this background in hand, we can now see a clear pattern in the
cycle weakening solutions that theoreticians have applied 
to encountered paradoxes<sup>3</sup>:

- Russell addressed his [paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox)
  (*if R is the set of all sets that are not members of themselves,
  R can neither be a member of itself, nor can it not*)
  by formulating his ramified theory of types, which weakens allowed relationships between sets. 
  Essentially, this theory of types establishes a partial order on entities. 
  An entity of some type can only be built up from entities of a lower type,
  much like the solution used by factorial and the dining philosopher problem.
  Russell's axiom of reducibility is too stringent, as it effectively prohibits recursion,
  but it got the job done until more versatile approaches could be formulated.
   
- Alonzo Church invented 
  the [simply-typed lambda calculus](https://en.wikipedia.org/wiki/Simply_typed_lambda_calculus)
  to address the Kleene-Rosser paradox, which showed that certain systems of formal logic, 
  including formalizations using Church's untyped lambda calculus, are inconsistent.
  This too applies a Russell-inspired hierarchy of types to
  weaken cycles in the lambda calculus.
  
- The [intuitionistic type theory](https://en.wikipedia.org/wiki/Intuitionistic_type_theory)
  that arose from Brouwer, Heyting, Kolmogorov, and Martin-Löf 
  is designed to carefully avoid paradoxical constructions.
  Here are two examples. By rejecting the
  [Law of Excluded Middle](https://en.wikipedia.org/wiki/Law_of_excluded_middle),
  intuitionistic logic avoids having to represent paradoxes that are neither true nor false.
  Instead, it uses the 1-type to represent things we can demonstrate exist, and the 0-type to
  represent anything that does not exist, including anything unprovable (e.g, a paradox).
  
    Another example is its use of inductive types, enabling the definition of
	natural numbers as a partial order. Each number is a successor of the previous number,
	beginning with 0.
	Inductive types can be self-referential, 
	but usually only in a way that permits structural recursion, 
	which weakens recursion (cycles) to avoid paradoxes.
  
	Handling recursion successfully is so important,
	that mathematics and logic distinguish between predicativity and impredicativity.
	An impredicative definition is self-referencing. A predicative definition is not.
	Intuitionistic theory is designed to be predicative.

For my final example, let's discuss Standard ML's modules.
Modules are a separate language grafted on to the core ML language,
enabling considerable richness to modular constructions that support
improved reusability of code in an elegant, isolated way.
As great as they can be, people keep looking for ways to improve on
SML's modules, such as enabling support for the sort of ad hoc polymorphism offered
by Haskell's type classes.

Andreas Rossberg proposed a new language, 
[1ML](https://people.mpi-sws.org/~rossberg/1ml/1ml.pdf), that went further.
His objective was to integrate ML's modules and core language into a single language,
and make modules first class, so that they can be passed around like values.
This means there is no difference between functions, functors, and type constructors,
and similarly no difference between structures, records and tuples.
The paper observes that the syntactic distinction between the core and module languages
is just a coarse way to enforce predicativity for module types.
1ML replaces this syntactic weakening of cycles
with a more surgical semantic restriction.
To preserve predicativity (ensuring decidability in type inference),
1ML imposes the constraint that "during subtyping (aka signature matching) the type **type** 
can only be matched by small [monomorphic] types,
which are those that do not themselves contain the type **type**."

That sounds exactly like weakening cycles so that type inference becomes decidable,
while also ensuring a more powerful module and type system.

## The Moral of the Story ##

As with diamond cutting, the skill lies not with knowing it needs to be weakened,
but in knowing where to tap the cleaver to maximize its value.

So, the next time you encounter Turing setting out on another endless marathon,
offer to him a bicycle whose wheels have been skillfully weakened 
through the application of a finite partial order,
to ensure he will halt in time for tea.

## Coda: Formalizing the Partial Order Pattern ##

This post confused some readers. Some thought I claimed a way to overcome
Godel's and Turing's result. No, I stand by their work.
That said, Turing's result still allows for the fact that *some*
programs can be proven to terminate (or proven to never terminate).
It is these specific programs that I focus on (rather than the programs
that Turing based his results on and which Lawvere's proof unifies
as equivalent using category theory and the diagonal argument).

Another point of confusion revolved around the lack of a formal treatment
for the pattern I describe, making it seem as if the pattern (and examples) were
too vaguely described to warrant being an observation with substance.
Let's see if we close that gap beginning with a more precise description of
how to apply a "partial order" solution to weaken cycles so as to guarantee termination<sup>4</sup>:

1. Each cycle/loop has a testable exit condition that causes the loop to cease.
2. The testable exit condition operates on some varying state in the cycle.
3. The collection of all cycle-termination states can be mapped cleanly 
   to a finite set of abstract states.
4. The abstract states are organized as a partial order, such that each is greater
   than or less than the others transitively.
5. Every iteration of the cycle moves that abstract state from a greater value to a lesser value.
6. The cycle terminates when it reaches the lowest abstract state.

I suspect this description is specific enough that a formal proof
could be constructed (very different to Lawvere's!) showing that conformance to all of these
criteria would be sufficient to guarantee termination.
Furthermore, it should also be possible to demonstrate that certain
relaxations of these restraints could also guarantee termination.
For example, the fifth constraint could be relaxed from "every iteration" to
"A finite number of iterations".
Similarly, the fourth constraint could be reworded to allow multiple directed
paths to (possibly multiple) end state(s), so long as discrete forward progress
is nonetheless guaranteed over some finite number of steps.

Today, the Collatz conjecture is not proven to terminate.
However, were we some day able to formally map the rapidly vacillating integer
conditional states to a partial order, it should then be possible
to prove it.
However, it is also possible that some future proof of the Collatz conjecture
will not be equivalent to the terminating analysis on an abstract topology map.
Time will tell.

Usefully, we can use another variation of these criteria
to guarantee that certain programs will *never* terminate.
In particular, we could change the last two criteria
to indicate that progress along the directed graph of states
will never arrive at the lowest exit state for some values.
This can happen because we can show that forward progress requires
an infinite number of steps, or that certain paths never end
up at an exit state (as shown earlier with the lookForZero function).

One might wonder how this formalization of partial order applies to the
garbage collection examples I cited earlier. With tracing GC, it
is pretty straightforward.
If the tracing GC collector does not color memory as it traces and marks, 
it would follow the reference cycles forever and therefore never terminate.
The colors apply a partial order to the references, 
where the end state is that all references eventually turn black 
(the terminating black state) and the tracing GC terminates, despite the presence of cycles.

Applying the partial order to reference counting requires that we reframe the problem
around termination. Imagine that we run our ref-counting program within another program
called the Overlord. The overlord keeps track of all memory requested by our ref-counting program, 
and will not allow the ref-counting program to terminate until it has freed all memory it requested.
A ref-counting program with reference cycles and no/improper use of weak references will 
fail to terminate.
However, a ref-counting program that applies a partial order, 
ensuring every reference cycle has at least one weak reference, 
will terminate, because it will have successfully met our criteria 
and have freed all memory that it allocated.
So, when we look at reference-counted memory management in terms of getting the behavior we require
(no memory is leaked), 
and not just as a data structure requirement, 
then it becomes possible to appreciate how the use of a partial order 
to weaken all cycles guarantees the program terminates correctly.

-----

<sup>1</sup> Someone submitted this post to
[Hacker News](https://news.ycombinator.com/item?id=23185525).
How wonderful to read all the additional insights and connections shared by others!

<sup>2</sup> You did not think I was going to write a post and not reference
[Cone](http://cone.jondgoodwin.com/), did you?

<sup>3</sup> To my delight, @user has since shared an nlab link 
on the [diagonal argument](https://ncatlab.org/nlab/show/diagonal+argument).
It references Lawvere's proof, using category theory,
that these historical examples (and Cantor's diagonal theorem)
are deeply equivalent.
Yanofsky offers a [more-digestible explanation](https://arxiv.org/pdf/math/0305282.pdf)
of Lawvere's correspondence.

<sup>4</sup> I am grateful to Threewood, whose skepticism and clear thinking
challenged me to be more precise about these mechanisms and examples.

<sup>5</sup> Pony makes a strong claim about being deadlock-free: 
"It’s deadlock free. This one is easy, because Pony has no locks at all! 
So they definitely don’t deadlock, because they don’t exist." 

This part of the claim is true: it makes no use of locks, not in the runtime 
and none are surfaced to the Pony programmer. 
The only synchronization mechanism that Pony makes use of is the MPSC queues 
that make possible message passing between actors. 
Each actor has its own queue, and it is only "active" when it has messages 
on its queue that need to serviced. The work-stealing scheduler will give every 
actor with messages an opportunity to process at least one message. 
When it does so, it never blocks and (hopefully) finishes with no requirement 
to return a result to any other actor (although it may be pass messages to other actors if it wishes).

Here is Wikipedia's definition of a deadlock: 
"a deadlock is a state in which each member of a group is waiting for another member, 
including itself, to take action, such as sending a message or more commonly releasing a lock". 
In other words, deadlocks can occur not only because of locks (which Pony does not have), 
but also because of waiting on another actor to send a message. 
The wiki article lists four conditions for a deadlock, which (it turns out) 
can also occur in a message-passing architecture.

How would Pony trigger this sort of deadlock? 
Consider implementing the dining philosophers problem in Pony by having the main actor 
spawn five fork actors (with on-table state) and then five philosopher actors, 
telling each the fork actor to their left and the fork actor to their right. 
The philosopher actors all wake up hungry, so they reach for their left fork first, 
by sending it an acquire message and then effectively suspend. 
If the left fork A is in on-table state, it changes its state to indicate 
it has been acquired by a specific philosopher and sends a message back 
to the philosopher to say it has been acquired. 
That reactivates the philospher to now send an acquire message to the right fork B. 
But maybe by this point in time another philosopher has already acquired 
that right fork B as its left fork! So it sends a message back 
to the requesting philosopher to say it is not currently available to be acquired, try again later.

The implications of this are clear, although every philosopher 
is getting activated by messages regularly, at a certain point in time 
it is possible that meaningful forward progress can cease for at least one philosopher. 
At least some philosophers will starve, because they become deadlocked 
in not being able to get two forks needed to eat, 
because they are effectively contending over access to shared resources.

The situation is semantically equivalent to when we use locks for forks instead. 
And if we apply Dijkstra's partial order to how the philosopher actors 
acquire their forks in an actor-based message passing architecture, 
the deadlock risk vanishes and all the philosophers have a reasonable chance, 
eventually, to eat and therefore guarantee eventual forward progress.

Technically, one might choose to call this livelock, vs. deadlock,
but the overall damage is the same, and livelocks can be even harder to diagnose.