---
title: "2030: Programming Language Trends"
date: 2020-02-24T10:45:01+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "General"
tags:
  - "Cone"
---

When the clock ticks over to a new decade, it is customary
to look back, to reflect on how much we have accomplished,
and then look forward, to sort out where we want to go.
Ten years is long enough that substantive progress should be visible
in the glacially-slow evolutionary pace of programming languages.
One can see this by noticing how many now-influential languages had
no notable marketplace presence only ten years ago: 
Rust, Go, Swift, Kotlin, Dart, and Julia.
Each of these languages brought enough new value to
garner significant market adoption vs. older, entrenched languages.

In January, several people polished their crystal balls and offered
[predictions](https://www.reddit.com/r/ProgrammingLanguages/comments/eljhdn/predict_the_state_of_programming_languages_in_2030/)
to this question: "what notable trends do you foresee unfolding over the next ten years 
in terms of PL theory, design, practice and marketplace adoption".
Although my crystal ball has shattered beyond repair,
this question provoked me to wonder what sort of improvements
I would like to see unfold in the commercial 
(vs. academic) evolution of programming languages.
In my thoughts, you will see that I am more interested in imagining how the overall capabilities
of PL ecosystems should improve over time, rather than
projecting which languages are going to win (or lose) as the marketplace shifts.

Before we can make anything, we must first dream it.
Well-grounded and compelling dreams are often a force for good,
as they change people's perceptions of what is possible and worth working towards.
Here are my dreams.

## Driving Forces for Change ##

Before diving into specifics, it is helpful to assess which
marketplace forces are compelling enough to fund tangible improvements
to programming languages and their ecosystems.
I would name these:

- **Developer productivity**.
  Corporations are as desperate to improve development speed
  as they ever have been, because this remains a notable lever
  for competitive advantage.
  Prior experience cautions us to be cynical, as improvements to PL design features
  can rarely be directly correlated to significant productivity gains.
  However, the richness of a language's overall ecosystem,
  its libraries, frameworks, tools, interoperability, 
  training, and marketplace clout, can offer significant leverage,
  as demonstrated historically by responsive IDEs, devops, and popular frameworks.
  Going forward, I see potential for further productivity improvements 
  resulting from tooling improvement opportunities
  in such areas as inference assistance, declarative programming, pattern refactoring, and cloud tooling.

- **Quality, safety and security**.
  The importance of these goals has rapidly escalated in importance, as our machines multiply,
  get interconnected, and become interwoven in the fabric of our lives.
  More and more, we entrust our life, health, finances and identity to programs
  that we expect to operate correctly, and be immune to invasive corruption.
  Rust's marketing campaign has been successfully built around these themes.
  Going forward, I see more languages jumping aboard this band-wagon,
  and bringing to the market additional research techniques for formally verifying the
  correctness of programs.

- **Throughput and latency**.
  Performance considerations are also increasingly important,
  particularly with [Moore's Law](https://en.wikipedia.org/wiki/Moore%27s_law)
  going on life-support.
  Corporations lose money whenever response times
  are not sub-second, more cloud servers must be bought,
  mobile devices run out of power and embedded device CPUs become overwhelmed.
  As software gets more complex and bloated,
  and hardware advances cannot keep pace, businesses are driven to use PL ecosystems
  that generate leaner, performant executables. This helped drive the adoption
  of Go, for example, to replace use of dynamic languages for server software.
  Going forward, I hope for further relief from PL ecosystems that
  offer ever more efficient use of memory and multi-CPU resources.

It is important to notice that these goals often act at cross-purposes.
For example, useful techniques for improving performance and safety can have a
negative impact on programmer productivity, due to added complexity.

## Type trends ##

A language's type semantics are central to everything about its ecosystem. 
It affects everything: library and program architectures, tooling, and interoperability. 
Therefore, improvements made to the utility of types ripples outwards with a magnifying impact.

### FP/OOP Static Type Fusion ###

We are already seeing so-called "imperative" languages increasingly
adopt valuable types and control flow patterns from the ML-family of 
functional programming languages.
In languages like Rust and Scala we are seeing features like:

- First class functions and closures
- Streaming iterators, with special support for map, filter, reduce, etc.
- Algebraic data types (sum types) and pattern matching
- Option<T>, which offers a safer way to work with null values.
- Result<T,E>, which improves on the safety of exception handling
- Tuples for constructing and destructuring multiple values

Over the next ten years, I hope this trend continues and accelerates,
resulting in a fusion of the most useful types and patterns of both OOP and FP paradigms.
In addition to the list above, additional synthesis opportunities should be pursued:

- Create more versatile [variant types](/post/unified-variant-type/)
  that marry together the advantages of sum types and inheritance-driven classes
- Tighten [inheritance](/post/favor-composition-over-inheritance/)
  to preserve its valuable form of reuse, while lowering the risk of problematic designs 
- Unify the power of subtyping-based polymorphism across generics, virtual dispatch
  and inheritance. This should include support for field/row polymorphism, structural subtyping,
  type extension, and multi-method capability.
- Use contracts on functions to help enforce invariants
- Offer sugar that reduces Option- and exception-handling code bureaucracy

I think there is a good chance we may also see some mainstream adoption of
research activity focusing on dependent types and effect systems.
Without getting into the weeds, [dependent types](https://cs.ru.nl/~wouters/Publications/ThePowerOfPi.pdf)
offer the potential to improve metaprogramming, 
by allowing more subtlety in the way we define data structures and
the logic needed to (de-)serialize, parse, format, manipulate and search that data.
Effect systems might help us to trace and manage the cause-and-effect chain of events
between some trigger, the program logic, and the resulting behavior.

Hopefully, the religious divide between OOP vs FP evangelists will fade away 
as the meaningful distinctions between them fall.
Accomplishing this brings a larger benefit: once future languages fully support a superset
of existing types and patterns, it becomes easier to automate the migration of legacy code
to stronger, modern language ecosystems.

### Dynamic Types ###

I anticipate static-typed languages will continue to gain ground over
dynamically-typed languages, at least for large, corporate software.
This is driven by several marketplace and technology trends:

- Static languages continue to have a demonstrable advantage with regard to throughput,
  latency, safety, multi-CPU support, and the productivity of long-term maintainability.
  As these drivers heat up, we will continue to see companies pivoting away
  from dynamic languages (e.g., node.js or Ruby) to static languages (e.g., Go).
- As static types become more flexible (e.g., variant types),
  this will blunt the advantage dynamic languages have had
  in their easy support for heterogeneous collections.
- Static languages are becoming increasingly higher-level and easier
  to use (e.g., inference), enabling new programs to be created more quickly.
- As IDEs increasingly integrate with responsive compilers,
  the wait-delay between code and test dramatically reduces,
  blunting the traditional rapid feedback benefit of dynamic languages.
- The emergence of WebAssembly offers static languages an expanding beachhead
  for challenging Javascript's monopoly on the browser platform.

Dynamic languages have been adapting to these trends as well.
Use of JITs and other optimization techniques have somewhat narrowed the performance gap,
but I doubt future gains will be significant.
Dynamic languages are reaching towards gradual typing to improve safety,
vaulting forward the market acceptance of TypeScript.
But gradual typing, as currently practiced, is a short-term bandaid, as it improves on the user
interface with semi-static types, but does not really address
the performance and concurrency challenges of dynamic types.
If anything, the success of gradual typing demonstrates that typing annotations
are viewed by many programmers more as a benefit than an impediment.

This is not to say that dynamic languages will (or should) fade away
in ten years. Marketplace momentum will carry them forward long into the future.
And dynamically-typed languages continue to have a sweet spot,
particularly for small to medium-sized "scripting" programs,
where flexibility and speed-to-run vastly outweigh performance and safety considerations.

The most exciting future opportunity that I envision for dynamically-typed
languages lies with applying gradual typing in reverse.
Instead of adding static type annotations to dynamic languages,
we should be looking at ways to inject dynamic types into static languages
(as C# has done [to some degree](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/using-type-dynamic)).
This way we get the safety and performance benefits when static type constraints make sense
(which is most of the time),
and the heterogeneous flexibility of dynamically-typed values when needed.

Doing so will help static-type evangelists shed their misunderstanding that dynamic types
are only about the absence of type annotations.
This opens the door to viewing them as the richest and most flexible kind of sum type,
one capable of supporting heterogeneous data structures,
dynamically-structured field-mapped data, and first-class types (monkey patching).
As Rich Hickey and others have [articulated](https://www.youtube.com/watch?v=2V1FtfBDsLU),
this sort of flexibility can be handy when working with data or logic
whose structure is dynamically in flux.

## Memory and Resource Management ##

Dynamic and managed programming languages often consume and churn memory prodigiously.
Objects are promiscuously created and destroyed in inefficient ways.
This degrades throughput, latency, and battery lifetimes, and makes it
hard to target constrained targets, such as WebAssembly and embedded devices.
Underneath these languages lies a tries-to-be-invisible, highly complex, tracing garbage collector.

We already know how to do better, using design techniques long-used
by game developers. In particular, [data-oriented design](https://en.wikipedia.org/wiki/Data-oriented_design)
arranges and accesses data in cache-friendly ways.
Additionally, use of arenas and pools offer noticeable performance
gains over traditional memory allocation and management strategies.

C++ and Rust, like Cyclone before, opened the door to re-thinking how programming
languages should offer fine-grained control over memory and other resources,
through the use of advanced reference types that ensure both
memory safety and performance. Particularly noteworthy are the use of:

- [Linear/affine types](/post/cyclone/)
  (move semantics) to ensure the deterministic
  and safe release and recycling of memory and other singly-owned resources.
- Bounded arrays and slices that allow secure and speedy access to
  an ordered collection of values.

Over the next ten years, I expect that other languages will not only
feature these capabilities, but will extend them in interesting ways,
particularly by offering polymorphic region support. 
Programs will be able to choose, for each allocated object,
which safe memory management strategy manages it:
single-owner, ref-counted, tracing GC, arena, pool, or other
custom-built region.

The primary obstacles to languages adopting better memory management strategies will
be ignorance and the added complexity of unfamiliar type annotations and rules
(as witnessed by programmer resistance to Rust's borrow checker).
Inference-driven tooling can help with this. Languages like 
[MLkit](https://link.springer.com/content/pdf/10.1023/B:LISP.0000029446.78563.a4.pdf),
[Lobster](https://www.reddit.com/r/ProgrammingLanguages/comments/edimm2/compile_time_reference_counting_lifetime_analysis/),
and [ASAP](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-908.pdf)
demonstrate that compilers can
read existing code and infer which memory management strategy better
addresses the way the program uses resources.

One approach to inference is to bake it into the compiler and make
memory management selection invisible, optimized by the compiler.
The potential downside to this is the buid time delay of doing this sort of
"whole program" inference on every compile.

I am more interested in an alternative approach where whole-program
memory inference is only run on demand. It can read unannotated
source code and analyze how the program's logic references memory,
looking at information about aliasing, lifetimes,
cache-crashing data access, etc., and then offers actionable suggestions to the programmer
on how to (re-)annotate the code with region and lifetime or re-factor data structures and access patterns,
in order to safely achieve better throughput or latency.
Such a tool could even be enriched with better insights
by examining execution profiler data on memory utilization and lifetimes.
Rather than having optimization only be an after-the-fact part of the build process,
I see tremendous potential in having the programmer participate in 
folding in explicit design insight back into the source code.

## Concurrency ##

Multi-CPU machines are ubiquitous;
efficient exploitation of their power is not.
Progress is held back by the safety challenge of race conditions,
programmer unfamiliarity with designing for concurrency/parallelism,
and the fact that most legacy languages are not well designed
to support safe, performant concurrency.

Given the large gap between CPU capability and common practice,
it is no surprise the last decade saw an explosive
growth in concurrency patterns, well beyond traditional locks, software transactions,
and the OS ability to multi-task process-isolated programs.
Emerging mainstream patterns include:

- [**Channels**](https://en.wikipedia.org/wiki/Channel_(programming))
  (Rust, Go, etc.) for passing messages between threads.
- [**Promises**](https://en.wikipedia.org/wiki/Futures_and_promises)
  and implicit continuations (Rust, JS, etc.) to ensure m:n threads
  are not blocked for I/O, while also facilitating responsiveness and developer ease-of-use.
- [**Permissions**](/post/race-safe-strategies/)
  (e.g., Rust and [Pony](https://tutorial.ponylang.io/reference-capabilities/reference-capabilities.html))
  that allow a program to polymorphically select the best combination of aliasing,
  mutability and synchronization constraints to prevent data races.
  Affine types and move semantics play a huge part here as well,
  enabling externally-isolated, mutable data structures to be safely moved locklessly from
  one thread to another.
- [**Actors**](https://en.wikipedia.org/wiki/Actor_model) (e.g., Pony, Erlang and Actix),
  an architectural model consisting of thousands of lightweight, concurrent
  actors who communicate with each other via queued messages.
  Sometimes, this can be scaled up to distributed computing.
- [**Concurrent data structures**](https://en.wikipedia.org/wiki/Concurrent_data_structure)
  which allow multiple threads to have [lock-free](https://en.wikipedia.org/wiki/Non-blocking_algorithm#Lock-freedom),
  concurrent, mutable access to the same data structure
- [**Structured Concurrency**](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/),
  to statically reason about and better manage concurrency control flow.

Although more patterns may emerge, 
I believe the primary concurrency challenge of the 2020s will be to make 
sense of out of this cornucopia of patterns.
In the 2020s, I hope to see:

- **Recipes and guidelines** that help designers select which
  pattern(s) are the best to employ for some problem domain.
  In the same way that diamond cutting is a difficult, but teachable art,
  so too should programmers be taught how to cleave a monolithic
  program across multiple CPUs and threads for optimal throughput, latency and safety.
  
- **Deadlock analysis tools** that can help diagnose and correct synchronization designs
  that prevent guarantees of forward progress. Even in a language without
  locks (e.g., Pony), it is still possible
  to design actors unable to move forward because they are waiting on each other forever.
  As with the [dining philosophers problem](https://en.wikipedia.org/wiki/Dining_philosophers_problem),
  a [promising approach](https://www.cs.cmu.edu/~balzers/publications/manifest_deadlock_freedom.pdf)
  may lie with analyzing whether a partial order is preserved across contested resources.
  
- **Concurrency inference tooling**, which can help programmers
  determine where a program can safely take advantage of concurrency or parallelism.
  As with memory inference, such tooling could pursue fully-automated,
  whole program inference (e.g., [Aeminium](https://www.cs.cmu.edu/~aldrich/papers/aeminium-toplas14.pdf)
  or [Automatic Parallelism in Mercury](https://paul.bone.id.au/pub/pbone-2012-thesis/)).

	  Again, I am more intrigued by the idea of responsive tooling that can make use of
	  data flow analysis and profiling statistics to offer meaningful suggestions on where
	  and how to inject permission, concurrency or parallelism annotations into largely serialized code.
	  As Paul Bone eloquently described it: ""Hey, your program would go MUCH faster if I put parallelism here, 
	  but it's unsafe because of this thing, could you move that thing for me?"
	  Even better would be the ability for it to refactor the logic to better exploit
	  various concurrency patterns.
	  And even better, what if the IDE were smart enough to recognize when
	  safety constraints were broken (trying to mutate an immutable value),
	  and it offered ways to automatically refactor the code so as to make it work.

## Tooling ##

So far, I have spoken largely about language feature improvements, 
highlighting how they can improve performance and safety, two of the three drivers listed earlier.
However, these features are unlikely to improve developer productivity.
If anything, they may well degrade it, due to greater programming complexity.
To increase developer productivity,
we should look to tooling for relief.

In the last decade, tooling has undergone seismic platform shifts:

- **Github** (and Git) now dominates the way we manage collaborative changes to source code.
  Increasingly, it is the hub for source-driven automation, such as
  continuous integration testing and deployment.
  
- **Responsive compilers** that are tightly integrated with IDE editors.
  These provide developers
  with real-time feedback on highlighted errors and solutions
  as source code is being altered. This capability is driven
  by the [Language Server Protocol](https://en.wikipedia.org/wiki/Language_Server_Protocol), 
  introduced by the highly popular VSCode editor.

Going forward, Github and IDEs will increasingly become the cloud-based control panel
for richer and more helpful tools. Here are several realistic directions our tools should take:

- **Legacy migration**.
  The biggest impediment to capitalizing on better language ecosystems
  is the enormous volume of legacy code written using inferior languages and libraries.
  One approach for overcoming this drag is incremental migration: rewriting a component at-a-time, 
  much as Mozilla has done with Servo within Firefox. This is a risky and expensive bet,
  exacerbated by the interoperability challenges of bridging components that do
  not share common type semantics. 
  
	  More useful would be a tool that can
	  semantically translate legacy code to a better language and libraries.
	  To be effective, the translation tools would not perform a literal 1:1 translation,
	  but should intelligently refactor the underlying intent to the new ecosystem's
	  idiomatic style. Instead of a perfect translation, I would want the translator
	  to mark every place where it struggled, so that a programmer can quickly resolve
	  the remaining gaps.

- **Inference and proof assistants**.
  I have already mentioned the value of memory and concurrency inference tools
  that can use whole-program data flow analysis to infer and suggest optimal static annotations
  and design patterns. No doubt there are other forms of inference that could also be helpful.
  Inference tools move us closer to the idea of gradual programming,
  where the programmer begins with a concise (perhaps even declarative)
  specification of the logic, and the tools help guide its specialized elaboration
  towards a safe, performant implementation whose core rules remain preserved
  and enforced.

	  In addition, we might also benefit from correctness proof assistants
	  which implement research-discovered techniques
	  that analyze code to determine where our programs are at risk of behaving incorrectly,
	  because they will violate declared invariants and constraints.
  
- **Architectural visualization and refactoring**.
  Most of our tooling helps us in the fine details.
  We are missing out on tools that offer useful visualizations
  and refactoring wizards with regard to architectural design patterns.
  Visualization tools have been tried before (e.g., [UML](https://en.wikipedia.org/wiki/Unified_Modeling_Language)),
  but were often rejected because of their extra work and cost.
  More effective would be architectural diagrams automatically generated from source 
  Similarly, how much more valuable would recipe cookbooks (and Stack Overflow) be,
  if wizards could auto-generate common design patterns
  or auto-refactor existing patterns based on structural changes?

Underlying these suggestions is the belief that we should never
build nanny-state tools that hide away pertinent implementation details, 
fight with programmers trying to get stuff done,
or attempt to eliminate the programmer.
Instead, our compilers and tools should be an invaluable amanuensis,
an idiot-savant assistant that we invite to help us solve problems,
thereby magnifying how much we can accomplish.
What we want is a partnership between agents with divergent talents:
tools that are best at gathering invaluable data and insight, 
and precise in automating consistency;
managed by professional people who are experts at knowing what needs to be built and how.

## Compiler Productivity ##

Before I let go of this topic, there is one more related area
I believe is worth raising: advances in the tooling used by 
people who invent new languages and ecosystems.

It is hard to overstate the influential role that LLVM
has assumed through the 2010s as a programming-language enabling technology.
A compiler-writer need only build a front-end that parses, semantically analyzes,
and generates LLVM IR. The LLVM backend then performs all the magic to translate
this SSA representation of the logic into highly-optimized executables across
over a dozen target architectures. It is impressive how many prominent languages
depend on this infrastructure: C, C++, Rust, Swift, and
[many more](https://en.wikipedia.org/wiki/LLVM).

Going forward, it is not unreasonable to hope for richer
frameworks that help us explore innovative PL capabilities forward more quickly, such as:

- Responsive compiler libraries that make it easier to architect a modern compiler
  built around demand-driven integration with the Language Server Protocol.
  
- Semantic analysis libraries that simplify the handling of name resolution,
  type checking/inference, and data flow analysis for rich type systems.
  One promising technology that might help close this gap is LLVM's 
  [MLIR](https://mlir.llvm.org/docs/Tutorials/Toy/Ch-1/),
  which provides a rich data structure and tools for flexibly capturing the semantics
  of logic according to some specific semantic dialect, similar
  to the role that mark-up languages and tools accomplish for data.

Imagine how much more quickly PL innovation might happen, were we able to
quickly compose a new responsive compiler that leveraged a variety of plug-ins:
EBNF/PEG-driven lexing/parsing, MLIR-based semantic analysis,
and the LLVM back-end code generation...

## Summary ##

Once again, my crystal ball is both cloudy and broken.
I make no promises that all of this is doable or will come to pass.
But these goals largely feel desirable and achievable to me over a ten-year journey,
driven by the relentless business pressures I cited early on.

I found it fascinating to contrast my list to the one
[Graydon Hoare](https://graydon2.dreamwidth.org/253769.html) 
published 2.5 years ago. 
Although there are some overlaps, they are largely on the fringes.

He focused more on the theory/research-driven side
of very complex, hard problems. He knows the state-of-the-art there
much better than I do, and offers a wealth of provocative links.
My list, by contrast, is less intellectually challenging, as it largely
elaborates on incremental refinements of known-art.
My focus and expertise lies more with the craft of engineering mainstream tools.

Most of all, this list is incomplete. I am eager to learn (more) about
your ideas. Let's keep turning our dreams into a better tomorrow.