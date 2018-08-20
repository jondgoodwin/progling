---
title: "Gradual Memory Management"
date: 2017-08-06
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: false # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Memory Management"
tags:
  - "Rust"
  - "Pony"
  - "Cone"
---

Realtime, quality 3D is punishing on hardware and software alike.
At least every 17 milliseconds, the scene needs to be updated and redrawn.
Managing memory efficiently across the many large and small data structures
that make up an interesting 3D scene
is a well-known and important part of that challenge.

<!--more-->

Game engines wisely reduce or eliminate the use of 
[tracing garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection),
due to the unpredictable "stop-the-world" lag that can result.
Instead, high-performance game engines use other memory management strategies,
such as [reference counting](https://en.wikipedia.org/wiki/Reference_counting), 
[pools](https://en.wikipedia.org/wiki/Memory_pool) and 
[arenas](https://en.wikipedia.org/wiki/Region-based_memory_management).

Experts in memory management know this truth:  No single memory management strategy
is best for all situations. They each have pros and cons.
If you optimize for performance, you can lose out on data structure flexibility or safety.
If you optimize for data flexibility, predictability and performance can suffer.

My first attempt at an easy-to-use programming language for 3D, 
[Acorn](http://web3d.jondgoodwin.com/acorn/index.html), 
uses a low-latency tracing garbage collector.
Its [non-copying collector](https://github.com/jondgoodwin/acornvm/blob/master/src/avmlib/avm_gc.cpp)
minimizes stop-the-world lag time by regularly
scheduling tiny generational and incremental cycles.
It works, but wrestling with its design
taught me a lot about the limitations and multi-threaded challenges of garbage collectors.

As I pivot to [Cone](http://cone.jondgoodwin.com/),
my second and more performant attempt at a 3D programming language,
I want to find ways to move beyond these limitations.
My dream is to find a way for each programmer to choose the memory management strategy
that best addresses their needs. Even better, I want the programmer to be able
to mix-and-match multiple strategies in the same program.
They can employ high-performance strategies where the data structures were amenable,
and then fill in the gaps with slower, but more flexible techniques.

As fate would have it, good luck came my way...

I had heard about [Rust](https://www.rust-lang.org/en-US/)
and its innovative ownership-based memory management model.
Curious, I studied it to understand its promise, its limitations, and
how its mechanisms work.
Coincidentally, I had also recently been introduced to 
[Pony](https://tutorial.ponylang.org) and its innovative 
actor model and reference capabilities, which I also studied around the same time.

What I noticed was that both Pony and Rust offer strong memory and race safety guarantees,
but accomplish them in somewhat different ways. That provoked my curiosity.
Could they be variations on a deeper, more versatile memory management framework?

Patterns emerged from my studies, and I began to get excited.

It might indeed be possible to create a language able to safely support
plug-and-play memory management,
by simply marrying their mechanisms and adding a few modest extensions.

In September 2017, I published a paper called 
[A Framework for Gradual Memory Management](http://jondgoodwin.com/pling/gmm.pdf)
that describes, in some detail, how this might be accomplished.
I was fortunate to receive astute and helpful feedback from many perceptive reviewers.
I encountered additional challenges since writing that paper,
but I still consider the described framework largely intact and promising.

I am now working through how to implement its mechanisms as part of Cone.
Some of the syntax has been sorted out with regard to
[references](http://cone.jondgoodwin.com/coneref/refrefs.html)
and [permissions](http://cone.jondgoodwin.com/coneref/refperm.html).
Cone's compiler enforces some of the type system mechanisms,
but there remains a lot of work left to fully realize this promise.