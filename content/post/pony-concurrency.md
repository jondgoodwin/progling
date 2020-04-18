---
title: "Actor Concurrency in Pony"
date: 2019-09-02T15:01:35+10:00
draft: true
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "concurrency"
tags:
  - "Pony"
---

What's the best way for a programming language to support concurrency?

* How do we maximize thoughput and minimize latency?
* What approach mitigates the risk of race conditions?
* What concurrency patterns facilitate developer productivity and maintainability?
* Are packaged libraries sufficient,
  or should concurrency also be baked into the language?

To find insight, let's explore
the architectural choices made by other language designers,
beginning with the actor approach as exemplified
by the Pony language.
Most (but not all) of what's said here applies to Erlang and
some actor-based libraries, such as Akka (Java and .NET) and Actix (Rust).

## What is Pony? ##

For those unfamiliar with [Pony](https://www.ponylang.io/), 
it is a statically-typed language
designed and built in 2014-5 by a team led by Sylvan Clebsch
(now with Microsoft Research). 
Its type system supports classes, algebraic data types,
interfaces, traits, and actors.
Data is dynamically allocated and implicitly accessed by reference.
Programs use a small runtime to
handle automatic memory management and thread scheduling.

Pony guarantees both memory and race safety 
(based on [formally-proven semantics](https://www.ponylang.io/media/papers/opsla237-clebsch.pdf)).
Memory safety results from its unusual Orca architecture,
which blends together both reference-counting and tracing garbage collection.
Race safety is ensured by the absence of locks and
the use of reference capabilities on data.
These make it statically impossible
for two threads to mutate the same data at the same time.

## What are Actors? ##

An [actor](https://en.wikipedia.org/wiki/Actor_model)
is essentially a distinct thread
that performs its work independently from any other actor.
Each actor has its own message queue.
Every message on that queue represents a piece of work the actor must perform.
Execution of a Pony program consists entirely of actors
completing the work on their queues,
which may in turn involve sending work requests to
other actors. 

Pony has no concept of global data (no ambient environment).
Instead, each actor maintains its own state,
and can only send messages consisting of data it has
been explicitly told about, and only to actors it has been told about.
Actors are, themselves, objects, allowing references to them
to be passed as part of a message.

Actor types resemble classes.
An actor's fields define its state.
An actor defines *behaviors*, instead of methods,
each representing a specific kind of work it can perform.
A behavior specifies parameters for the data it accepts.
Unlike methods, behaviors neither return value(s)
nor return to their caller.

## Message Passing ##

Pony's message passing syntactically resembles a method call:

    actor.behavior(data1, data2)
	
However, its underlying semantic constructs the
message as some variant of the actor type's implicit sum type,
and then adds that to the actor's message queue:

    actorQueue.push(Actor::behavior{data1, data2})

The variants for an actor's implicit sum type are its
defined (and hidden) behaviors, with the structural fields
of those variants being defined by each behavior's parameters.

## m:n Thread Scheduling ##

Actors are threads, but they are not OS threads.
They are comparatively lightweight, requiring much less memory.
This allows a actor-based approach to scale up to support
many more threads than would be possible 
if actors were to simply use OS threads.

Actor threads yield execution control willingly and quickly.
This sort of cooperative concurrency improves latency
and lowers the throughput cost of OS threads that
use [preemption](https://en.wikipedia.org/wiki/Preemption_(computing))
to determine to switch from one thread to another.

It is the job of the Pony runtime to decide which of
some 'm' number of actors to schedule next on
a small 'n' number of OS threads.
Each OS thread has its own independent scheduler,
and a list of the actors it is dispatching work to.
If an OS thread runs out of work, it can 
[steal](https://en.wikipedia.org/wiki/Work_stealing)
whole actors with pending work from another thread.
This approach maximizes throughput, as all CPUs
are kept at busy as possible.

By stealing whole actors, Pony is able to preserve
[sequential consistency](https://en.wikipedia.org/wiki/Sequential_consistency).
This ensures that any actor can expect another actor to
perform the work it requested in the same order that messages were issued.

## Non-blocking Architecture ##

The execution of an actor's behavior never stops, waits, or gets suspended.
When it sends messages to other actors, it does not wait for a response.
It never acquires any sort of lock, mutex or otherwise
(making it impossible for a Pony program to deadlock).
It also never issues a blocking I/O request.
These restrictions are what make it possible for the scheduler
to expect a behavior to yield execution control willingly and quickly.

Garbage collection and the queue.

Behaviors never block. No locks (no deadlocks) MPSC sync. Even GC does not block.