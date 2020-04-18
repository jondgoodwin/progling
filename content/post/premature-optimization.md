---
title: "Premature Optimization"
date: 2019-03-07T07:31:33+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Performance"
tags:
  - "Cone"
---

The decades-long golden age of [Moore's Law](https://en.wikipedia.org/wiki/Moore%27s_law) is fading.
Slow, bloated software will find it increasingly difficult to hide
behind the skirt of ever-faster computer hardware.
Given that businesses and users will not yield on their demand for speedy software,
developers will need to get a lot better at architecting for performance.

In his excellent article on [Performance Culture](http://joeduffyblog.com/2016/04/10/performance-culture/),
Joe Duffy begins: "*Performance is one of the key pillars of software engineering, 
and is something that’s hard to do right, and sometimes even difficult to recognize.*"
Later on, he warns: "*I've heard the 'performance isn’t a top priority for us' statement many times
only to later be canceled out by a painful realization that without it the product won’t succeed.*"
I highly recommend this post for its detailed, hard-won advice on the value of
insightful performance metrics and a relentless focus on performance improvement.

In my post here, however, I want to focus on something he does not address,
the seeming contradiction between his
insistence on instilling a strong performance culture right from the start of a project,
and Donald Knuth's memorable adage from 1974:  "*Premature optimization is the root of all evil.*"

By way of personal example:
when I told someone I was architecting the Cone compiler for performance,
Knuth's authority was invoked to warn me that I was going about it backwards.
Build it first, I was advised, then use profiler tools to determine where performance
optimization will yield in the greatest improvements.

Whose lead should I follow, Knuth or Duffy?

## What is Knuth getting at? ##

I am always suspicious of wisdom-in-a-sentence.
When complex design choices are distilled simplistically into a few words,
valuable context can often boil away in the process.
Let's recover it.

His famous dictum was published in two papers back in December 1974.
In ["Structured Programming with **go to** Statements"](http://citeseerx.ist.psu.edu/viewdoc/download;jsessionid=23230D8413EF633953BDF6102B9D513C?doi=10.1.1.103.6084&rep=rep1&type=pdf), 
the quote appears at the end of this paragraph:

*There is no doubt that the grail of efficiency leads to abuse. Programmers waste
enormous amounts of time thinking about,
or worrying about, the speed of noncritical
parts of their programs, and these attempts
at efficiency actually have a strong negative
impact when debugging and maintenance are
considered. We should forget about small
efficiencies, say about 97% of the time: premature optimization is the root of all evil.*

Knuth sensibly wants to avoid wasting programmer time on unfruitful optimization work.
He also wants to minimize the impact that optimization techniques can have on
making the program's logic harder to understand and debug.
Despite these misgivings, he does advocate strongly for optimizing wherever we *know*
that a significant improvement can be obtained:

*Yet we should not pass up our opportunities in that critical 3%. A good programmer
will not be lulled into complacency by such
reasoning, he will be wise to look carefully
at the critical code; but only after that code
has been identified. It is often a mistake to
make a priori judgments about what parts
of a program are really critical, since the
universal experience of programmers who
have been using measurement tools has been
that their intuitive guesses fail.*

Knuth considers even a 12% improvement to be significant:
"*In established engineering disciplines a 12% improvement, easily obtained,
is never considered marginal; and I believe
the same viewpoint should prevail in software engineering.*"

Since efficiency is not the focus of either paper,
the topic is only treated lightly.
Although these passages properly highlight the key trade-offs,
he offers only cursory advice on how to find and remedy the "critical 3%":
"*all compilers written
from now on should be designed to provide
all programmers with feedback indicating
what parts of their programs are costing the most [...]
After a programmer knows which parts of
his routines are really important, a transformation like doubling up of loops will be
worthwhile.*"

His advice to postpone efficiency tuning as long as possible
is recapitulated near the end of the paper:
"*In our previous discussion we concluded that
premature emphasis on efficiency is a big
mistake which may well be the source of
most programming complexity and grief.
We should ordinarily keep efficiency considerations in the background when we formulate our programs.
We need to be subconsciously aware of the data processing
tools available to us, but we should strive
most of all for a program that is easy to
understand and almost sure to work.*"

## Simple programs vs. complex systems ##

With such sensible advice from an esteemed authority,
how can Duffy (and I) justify advocating an aggressive approach to performance?
Are there legitimate grounds to challenge Knuth's argument?

Yes. Since his observations are not backed up by large-scale experimental data,
several thoughtful questions may be posed:

* Is the "critical" section really just 3% of the code?
  Although the [Pareto principle](https://en.wikipedia.org/wiki/Pareto_principle)
  almost certainly applies to performance tuning, the proportion of code that is critical
  very likely varies according to the program's underlying architecture.
* Would improving performance of the non-critical code truly not be worth the effort?
* Does performance optimization always result in code that is harder to understand and debug?
* Are performance profilers the most effective way to identify
  the code whose optimization would yield the greatest performance improvement? 

When it comes to small code bases and algorithms (where Knuth has devoted much of his work),
he may well be sufficiently right.
However, with large, complex, performance-critical software systems 
(e.g., compilers, virtual machines, operating systems),
I am not convinced it is wise to bet your success on
waiting until near the end of development work to 
optimize a paltry 3% of your code base.

Here, as with so much else about software development,
an ounce of prevention might well be worth a pound of cure.
If you allow yourself to get way behind the curve on performance,
the effort it will take to carve out all the accumulated fat is likely to become monumental.

## Good Design and Premature Pessimization ##

My biggest concern about Knuth's approach is its focus on
a certain kind of performance bottleneck, 
where inner loop(s) dominate the work.
A performance profiler is invaluable here, since it can quickly identify
which structured section(s) of the code consume the most CPU cycles.
A relatively low effort in optimizing these functions
can often yield a significant improvement to overall program efficiency.

This strategy struggles, however, when inefficient code patterns
are repetitively diffused throughout the code base.
Thousands of tiny paper cuts sprinkled in many different places are likely to be
largely "invisible" to performance profilers.
Even if tools could identify all such code fragments,
tuning their logic for better performance is going to take a lot more effort,
given how many different parts of the code are effected.

The most productive strategy for reducing this overhead is *prevention*,
achieved by thoughtful architectural design.
Herb Sutter makes the case for prevention in Rule 9 of the
"C++ Coding Standards":  [Don't Pessimize Prematurely](https://books.google.com.au/books?id=mmjVIC6WolgC&pg=PT72&lpg=PT72&dq=premature+pessimization&source=bl&ots=ceTmMRkKY8&sig=ACfU3U2_RCIAMOnK7w8I38bKsQdtQ0D0og&hl=en&sa=X&ved=2ahUKEwjs9PCB6PDgAhXCeisKHSjvCiIQ6AEwB3oECAQQAQ#v=onepage&q=premature%20pessimization&f=false).
"*All other things being equal, notably code complexity and readability,
certain efficient design patterns and coding idioms should just flow naturally
from your fingertips and are no harder to write than the pessimized alternatives.
This is not premature optimization, it is avoiding gratuitous 
[pessimization](https://stackoverflow.com/questions/15875252/premature-optimization-and-premature-pessimization-related-to-c-coding-standar).*"

Joe Duffy makes a similar argument in 
["The 'premature optimization is evil' myth"](http://joeduffyblog.com/2010/09/06/the-premature-optimization-is-evil-myth/):
"*I have heard the 'premature optimization is the root of all evil' statement 
used by programmers of varying experience at every stage of the software lifecycle, 
to defend all sorts of choices, ranging from poor architectures, 
to gratuitous memory allocations, to inappropriate choices of data structures and algorithms, 
to complete disregard for variable latency in latency-sensitive situations
[...] I do not advocate contorting oneself in order to achieve a perceived minor performance gain.
[...] What I do advocate is thoughtful and intentional performance tradeoffs being made 
as every line of code is written. 
Always understand the order of magnitude that matters, why it matters, and where it matters. 
And measure regularly!
[...] Given the choice between two ways of writing a line of code, 
both with similar readability, writability, and maintainability properties, 
and yet interestingly different performance profiles, don’t be a bozo: choose the performant approach. 
Eschew redundant work, and poorly written code. 
And lastly, avoid gratuitously abstract, generalized, and allocation-heavy code, 
when slimmer, more precise code will do the trick.*"

## Memory, Data and Synchronization ##

The words "*efficient design patterns ... should just flow naturally from your fingertips*"
or "*choose the performant approach*" make it sound like these choices should be easy and obvious.
They are not. Years of thoughtful experience, research and experimentation matter.

The two posts from Joe Duffy that I linked above offer a wealth of concrete wisdom
that should help put a good programmer on the right path.
In particular, Duffy's "myth" post has three detailed sections that offer
constructive advice regarding data structure design, memory management, and
synchronization strategies, where insightful architectural choices
can make a huge difference on performance.
(Additionally, implementation language and libraries choices can also
have a significant impact on performance.)

To his advice, I would like to add a few more thoughts:

* **Memory management**. malloc() and free() are surprisingly expensive. So, too,
  are runtime bookkeeping for reference counting and tracing GC, particularly with multi-threaded code.
  This is why I am so bullish about the prudent use of
  bump-pointer arena allocators, pools, single-owner (RAII) memory management,
  and borrowed references. That's also why I am baking these performant mechanisms
  into Cone, and working hard to make them explicit and easy-to-use.
  
* **Data copying**. Data is often copied around needlessly, potentially accompanied by
  costly additional allocations. Examples abound: string handling, immutable
  or persistent data structures, message passing, etc. The cost of copying can add up
  when it happens alot, especially for critical data structures that
  the code revolves around. This is why I am bullish about prudent use of slices and
  reusable, mutable data structures.
  
* **Indexing**. A similar case can be made for reducing data structure indexing costs,
  especially for hashed data, through the use of borrowed references and
  (string) interning.
  
* **Data Oriented Design**. Huge performance wins can be obtained by organizing
  data access in cache-friendly ways. Ideally, one wants to structure code to
  swiftly navigate a long cache line from front-to-back.
  This means avoiding certain common OOP design patterns, such as allocating
  each entity separately (vs. gathering them together in a single array object),
  virtual dispatch for handling each entity, 
  and the liberal and unnecessary use of getters and setters.

## Summary ##

I find it fascinating how much Joe Duffy and Herb Sutter
echo Don Knuth's warning about excessive optimization in the small,
especially for imperceptible improvements that make the code more complex, 
and therefore harder to understand and maintain.

Yet, they enrich Knuth's advice when insisting on the importance (when throughput matters)
of investing time early and often to architecturally design for performance,
particularly with regard to memory management, critical data structures, and
synchronization strategies. Poor decisions here can multiply the
widespread (and hard to detect) distribution of lots of tiny pieces of inefficient code.
Getting it measurably right as you grow your software may take more time upfront,
but it can potentially save you a lot of heartache and greater refactoring effort later on.

If you believe performance will ever matter,
don't let simplistic warnings about "premature optimization" prevent you from
paying careful attention to the performance implications of your design and coding choices.