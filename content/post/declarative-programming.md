---
title: "The Siren Song of Declarative Programming"
date: 2021-01-04T07:48:18+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Paradigms"
---

Let's continue our journey of discovery through the wild jungle of 
[programming paradigms](/what-is-a-programming-paradigm)
by clearing away more obstructive vines.
This post tackles the taxonomic distinction that many
attempt to make between declarative and imperative programming.

What becomes clear, after some study, is that "declarative" and "imperative" do not
have singularly simple and clear meanings. Wikipedia offers [three incompatible definitions](https://en.wikipedia.org/wiki/Declarative_programming#Definition)!

The many strands of these confusing concepts entangle themselves into a 
[Gordian knot](https://en.wikipedia.org/wiki/Gordian_Knot)
that resists all attempts to neatly untie it.
Pursuing the siren song of declarative programming risks crashing into the rocks of cognitive confusion.
To stay safe, we will need to use Alexander's sword (and likely some Odyssean earwax)
to navigate a safe path past the dangers.

Over the course of this post, this is the approach I will take:

- Begin with common, helpful definitions
- Examine languages commonly identified as declarative
- Address whether functional languages should be classified as declarative

## Declarative vs. Imperative - 3 Definitions

The terms declarative and imperative are not original to programming.
They have a pedigree in linguistics, the study of human languages.
They describe the [modality](https://en.wikipedia.org/wiki/Grammatical_mood) 
of (say) a verb or sentence.

- An imperative sentence is a command or request: "Go buy milk.".
- An interrogative sentence asks a question: "Did you go buy milk?"
- An indicative (declarative) sentence claims a proposition:  "You went to buy milk."

Computer science is not required to define their terms the same ways as linguistics, 
but I find that interdisciplinary conceptual similarity helps lead towards clear, insightful conversations.
For this reason, I prefer definitions whose meaning hews closely to common sense.

An oft-cited distinction between imperative vs. declarative programming
is captured by this quote:

> In mathematics we are usually concerned with declarative (what is) descriptions,
  whereas in computer science we are usually concerned with imperative
  (how to) descriptions. (*The Structure and Interpretation of Computer Programs* -
  Abelson and Sussman, 1996)

This shifts the meanings in an easy-to-follow way:
imperative programming still involves commands (to the computer!),
but the purpose of those commands is understood to be telling the computer 
*how to* perform some computation.

Distinguishing between "what is" and "how to" is extraordinarily compelling to programming agility.
Programming is a complex and labor-intensive discipline.
Who would not prefer that the computer do *what* we ask it to do,
without our being forced to tell it exactly *how* to do it?

Pursuing this line of thinking leads to another shift in how to define these terms.
This shift results from wondering: how can we tell the difference between a
"what is" vs. "how to" program?
A common criteria uses the presence of control flow as a signal for the imperative:

>  Declarative programming ... expresses the logic of a computation without describing its control flow.
  ([Wikipedia](https://en.wikipedia.org/wiki/Declarative_programming)).

By this point, we have now collected three related, but distinctly different, conceptualizations
of these terms: the linguistic, the "how to", and the "control flow" definitions.
This is nowhere close to an exhaustive list (as we will show later).
Already, I hope you are already noticing the invasive, mutating, Hydra-like root stock that
has become the source of all these entangling vines we need to slash away.

More definitions will arrive later, but for now we have a good enough starter
set for examining whether certain "declarative" languages actually comply to these definitions.
We will discover that these definitions are not at all as equivalent as we feel they should be.

## HTML: Content and type declarations ##

Markup languages like HTML, XML, JSON and others are clearly declarative,
regardless of whether we apply the linguistic, "how to" or "control flow" definitions.
An HTML file specifies the structure and content of some document ("what is").
It lacks any sense of commanding or of control flow.
The same can be said for type declarations specified in many programs.

Although HTML is a declarative language, can we legitimately claim it is a declarative *programming* language?
From my perspective, I really don't consider an HTML file to be a program.
It's just data. On its own, it carries no clear-cut implication for its computational result.
For that, we require some external program to process this data in an intentional way:
render it with a browser, search it using XPath, etc.

## SQL: Library APIs ##

Some consider (a subset of) SQL to be declarative.
When we look at a SELECT statement in isolation, it seems a golden example of
showing how to ask for "what is" without needing to specify "how to".
The absence of control flow constructs in SQL appear to solidify the case that
SQL is a declarative language.

And yet, it does fail the linguistic test: every SQL statement is
an imperative command or request by intention and design.
Programs use SQL statements as commands or requests to be performed by a database management system.

Is there a conceptual perspective we can use to reconcile these contradictory views?
Yes! We can correctly conceive of a SQL statement as if it were a complex function call to
some system's API. Syntactically, it may not resemble the syntax of function calls in most
languages, but semantically, this is exactly what we are doing:
calling specific functionality in another module, passing it rich parametric information,
and expecting a certain kind of result to be returned.

Suppose we had a program in some imperative language that just did this:

    array.Sort();

It says what to do, not how to do it. It has no control flow.
In isolation, this looks every bit as declarative as SQL SELECT, but I think that is deceiving.
If we were to add just one more statement to our program, we will notice more clearly
the imperative nature that has been there all the time,
because the fuller program is more clearly specifying **how** to do its work
and has a well-defined flow of execution.

The lesson here is important: let's not give our definitions more authority than they have earned.
We need to acknowledge that the "how to" and "control flow" criteria poorly handle
the degenerate case of a "program" that consists of a single function call.
Languages that are really just a formalized syntax for API requests are fundamentally imperative,
not declarative, in nature.

There is another, deeper lesson we can learn here.
If one of our worthy programming goals is to maximize "what is" programming over "how to" programming,
we should be pursuing modularity with far more vigor than so-called declarative languages.
The whole point of modularity and APIs is exactly this: to isolate away the "how to" logic (library package)
from the "what is" request (API/function call).

## Prolog: Constraint languages ##

Unlike HTML and SQL, I consider Prolog to be an fascinating example of a declarative programming language.

A pure Prolog program is a collection of fact and rule clauses.
When a query is issued against a program, the logical consequence for the program is calculated which indicates
the instantiation(s) which can be proven for the query.
Impure, imperative effects can be added to program clauses, which enable it to do more than 
simply derive provable instantiations.

What makes a pure Prolog program declarative? 

- Linguistically, fact and rule clauses are declarative. They are neither commands nor requests.
- All clauses express "what is". No "how to" is observable.
- A program contains no explicit control flow, not even execution order. The order of clauses in a program makes no difference.

In what way does the execution of a Prolog program differ from an imperative program?
In Prolog, the order of executing clauses is determined by the inference engine which evaluates the logical
dependencies between clauses. It uses these calculated dependencies to figure out how to dynamically "wire together"
the program's workflow as required to satisfy a particular query.
Another way to look at it, is that each clause establishes constraints. It is the engine's job to satisfy
the needs of the query without violating any constraints.

When looked at this way, there are a broad range of programming languages built around the dynamic solution
of data-driven dependencies or constraints:  

- build or package management systems (e.g., Makefile and many other DSLs),
- spreadsheets, 
- automated workflow (or logistics) frameworks able to determine the optimal order to schedule asynchronous
  tasks based on availability or other constraints of their input data dependencies.

As with the section that talked about declarative HTML, interpretation of such a declarative program
requires the existence of some runtime program
that treats the declarative program as "data" to process. A prolog program needs an inference engine to run.

## Are Functional Languages Declarative? ##

Now that we have a sturdy foundation for distinguishing between declarative vs. imperative programming,
we are ready to tackle the persistent claim that functional programming languages (like Scheme, SML and Haskell) are declarative,
unlike most other programming languages that are rightly considered to be imperative.

Confusingly, however, if we apply the three definitions given so far, 
functional programming languages turn out to be imperative, not declarative:

- A functional program specifically tells the computer what to do.
- Functional programs not only express "what is", they also very clearly specify "how to"
- Functional programs explicitly specify the order for executing functional operations, complete with constructs for
  conditional and repetitive evaluation.
  
What an odd state of affairs! What is going on?

It is surprisingly difficult it is to obtain a clear, crisp, authoritative answer to this Gordian knot puzzle.
People do not raise this issue, much less resolve it with any clarity.
This is particularly strange coming from academia, who normally insists on categorical rigor.

One reasonable hypothesis for this state of affairs is that yet another definition for declarative programming is being used.
This clearly shows up in SICP which, after baiting us with the "what is/how to" distinction I quoted above, switches later 
in Chapter 3 to:

> In contrast to functional programming, programming that makes extensive use of assignment is
  known as *imperative programming*.
  
By context, it is clear that SICP considers adherence to [referential transparency](https://en.wikipedia.org/wiki/Referential_transparency) 
to be a crucial distinction between declarative/functional programming and imperative programming.
Wikipedia also subscribes to this theme with the second of its three definitions for declarative programming languages:

> Any programming language that lacks side effects (or more specifically, is referentially transparent)

Despite Wikipedia saying that its three definitions "overlap substantially", this claim is neither obvious nor justified.
It takes a lot of cognitive gymnastics and squinting of the eyes to arrive at a casual conviction that
these definitions are largely equivalent (it reminds me of how the emperor ended up proud of his fine new clothes). 
Unfortunately, this confusion has been baked into the cultural zeitgeist of computing for so long,
it is unlikely to go away any time soon.

It took me way too long to realize that the real heart of this issue revolves around the close, symbiotic relationship
between functional programming and mathematics. Computer science and programming arose out of mathematics's deep roots.
Programming language and type theory have a particularly tight affinity to mathematics (as illustrated
in my post about the [rise of type theory](/rise-of-type-theory)).

Mathematical equations and logical propositions are declarative in nature, as assessed by all definitions given so far.
If you view programming languages through the mind-set of PL theory, lambda calculus, and the Curry-Howard correspondence,
and many academically trained people do, the type system of functional programs maps cleanly to proof logic.
Functional programming is just another way to do math, and if math is declarative, then so is functional programming!

You can see this reasoning in the way that
SICP regularly connects the functional programming model back to the declarative nature of mathematics.
Wikipedia's third definition for declarative programming languages makes it even more explicit:

> A language with a clear correspondence to mathematical logic.

Where so-called imperative languages stumble, in this view, rests in their support for variable mutation and side-effects.
This violates referential transparency, which as SICP and this 
[historical survey of functional programming](https://ccrma.stanford.edu/~jos/pdf/FunctionalProgramming-p359-hudak.pdf) explain,
destroys equational reasoning which is built around our ability to substitute "equal for equal".
If you view programming purely as a rich branch of math for arriving at proofs of correctness,
you will understandably view imperative languages as an existential threat.
The paradigm wars are largely fueled by the emotional strength of this biased agenda.<sup>1</sup>

I am sympathetic to the themes being raised here.
The correspondences between programming languages and mathematics are real and important.
The issues surrounding the lack of referential transparency are also real and worthy of deeper discussion.
But I think we are better off being clear, balanced and non-judgmental in how we treat these issues.

I do not believe it facilitates understanding to overload and complicate the common sense meaning of "declarative".
There are other words we can use to talk about the presence and value of referential transparency.
There are other, gentler ways to assess which languages do, or do not facilitate, proofs of correctness.
Passing value judgments on the supposed superiority of "declarative" over "imperative" languages (or vice-versa)
provokes and antagonizes. I would rather we focus our energies on facilitating a better synthesis of great ideas
from divergent legacies.

## Summary

Declarative programming is indeed a valid, distinct programming paradigm, as exemplified by the languages
and systems mentioned in the Prolog section of this post. This sort of programming is particularly
valuable in areas where we can cleanly separate a domain's declarative constraints from the inference engine
that is used to perform meaningful work based on the collection of rules it is fed.
It has been historically tempting to generalize this approach to a richer domain of rules.
However, experience shows that scaling it up can lead to accelerating complexity for both the rules
and the runtime engine, to the point where it eventually becomes too hard and slow to use and too costly to maintain.

When one feeds functional language programs into some sort of proof assistant, this use case
could also be considered declarative in the common sense of the word.
However, whenever we write a functional program for execution, it is best to acknowledge that 
its evaluation instructions are every
bit as imperative as for any program written in a "non-functional" language.
Let's not confuse the meaning of declarative programming by burdening it with the additional
connotations of referential transparency. Use other words to make these distinctions.

Finally, there are good reasons why imperative constructions and languages still dominate most of programming.
We should feel no shame in engineering solutions with care and precision 
using powerful general-purpose tools, leaning on the APIs of pre-built libraries to
deliver complex functionality with speed and quality.

As engineers, let's praise the practical utility of imperative programming
and talk more about its useful paradigms in follow-on posts.

-----------------------

<sup>1</sup>No one illustrates the nobility and prejudice of this perspective better than the eloquent preacher Dr. Harper.
His first post, ["What, if anything, is a declarative language"](https://existentialtype.wordpress.com/2013/07/18/what-if-anything-is-a-declarative-language/),
lists and then criticizes many contradictory and ambiguous definitions for declarative.
However, this is a rhetorical trap, one he springs with glee in his follow-up post
["There is such a thing as a declarative language, and it's the world's best DSL"](https://existentialtype.wordpress.com/2013/07/22/there-is-such-a-thing-as-a-declarative-language/).
Here he begins by correctly pointing out that the term "variable" has distinguishable meanings
in mathematics ("an unknown that is given meaning by substitution") 
and imperative programming languages ("given meaning by assignment (mutation)").

Based on this observation, this zealot quickly deduces that the results from imperative programming languages are
"a miserable mess of semantic and practical complications".
He goes on:

> Declarative languages, being grounded in mathematics, allow for the *identification*
  of the "desired behavior" with the "executable code" ...
  [Functional programming languages] are an integral part of science itself,
  inseparable from our efforts to understand and master the workings of the world. ... 
  Imperative programming .. is doomed in the long run to obsolescence, an artifact of engineering,
  rather than a fundamental discovery on a par with those of mathematics and science.

If you share Dr. Harper's faith in the supremacy of mathematics over every other human endeavor,
you may well find his reasoning to be sound and compelling.
But if you highly value the contributions that engineering makes to the quality of our lives,
you may be less eager than he is to simply discard its artifacts into the dustbin of history.

I value and enjoy mathematics, but I never forget that its substantial power is limited, because
it can never be more than a simplistic model that captures
fascinating patterns in the world we live in.
Engineering's role in human affairs is as important, but it is necessarily guided by different practices and goals,
especially the governing principle that since perfection is impossible,
our job is create an optimal design for the situation, which leverages the strengths of the materials used,
and minimizes the costs and risks of their shortcomings.

Harper's mathematics may find itself temporarily impotent in the face of assignables,
but that does not mean that engineers need to hold off on using this valuable tool
until mathematics can catch up. It is often true that engineering builds something useful
long before math and science can deeply explain how it works and show how else to leverage it.
This is not something mathematics (nor functional programming languages) need to be ashamed of!
In my mind, it is a better world when engineering, science, and math build synergistic partnerships that benefit humanity,
even when they don't necessarily value the same priorities in the same proportions.