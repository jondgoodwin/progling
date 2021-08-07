---
title: "What is a Programming Paradigm?"
date: 2020-11-22T10:07:08+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Paradigms"
---

Have our conversations about programming paradigms grown stale? 

It seems so.
Paradigms like object-oriented programming and functional programming, the two most talked-about,
are decades-old. Nearly always, conversations about them devolve into reciting the same 
desiccated stereotypes.

Is this because the notion of a programming paradigm has outlived its usefulness?
Dr. Harper appears to [think so](https://www.cs.cmu.edu/~rwh/papers/paradigms/paradigm.pdf).

Even among those who find some value in one paradigm or another,
there is a notable wariness to engage in fruitful discussions across diverse communities.
After too many emotionally draining debates between OOP vs. FP partisans,
it is easy to fear that programming paradigms are simply too radioactive to explore.
I have certainly felt this way.

And yet, I still think programming paradigms matter to the future of our craft. 
More than that, I believe we have not yet finished our journey towards better paradigms.
Indeed, if we are on the early stages of a fruitful paradigm shift,
I would love to see us explore this possibility with richer conversations
that help move us forward.

However, before we can do that, we first need to restore historical meanings that will
serve as a solid foundation for such conversations.
That foundation begins with defining what we mean by a "programming paradigm".

### Wikipedia's Unhelpful Definition ###

Wikipedia is not very helpful. It says: 
"[Programming paradigms](https://en.wikipedia.org/wiki/Programming_paradigm)
are a way to classify programming languages based on their features."
That's pretty much all it has to say about what a programming paradigm is.
The rest of the article just fleshes out representative examples of
this taxonomy without providing any unifying semantic meaning to explain
what qualifies as a paradigm and why it matters.

Humorously, if I subdivide languages whose names begin with vowels vs. consonants,
have I invented two new language-classification paradigms? 
(Vowel-Oriented Programming, anyone?) Paradigms-as-taxonomy seems foolish and shallow.

I do not fault Wikipedia for this. It accurately captures modern convention.

However, this definition strips away the useful history of the term. 
In doing so, it leaves us ill-equiped
to recognize a programming paradigm when we run across one, particularly a new and better one.

Even worse, it fails to recognize the invaluable role that programming paradigms play 
quite independently of programming languages. 
When we conceive of paradigms as only a taxonomy for programming languages, 
we end up misunderstanding and oversimplifying the relationship that exists 
between some paradigm and various programming languages, just as Harper does.

### Thomas Kuhn: Paradigm Shifts ###

The English word "paradigm" has been around in English for hundreds of years,
with notable special use by linguists. In its general sense, it carries this meaning:
"a typical example or pattern of something; a pattern, model or examplar". 

The 1962 publication of [Thomas Kuhn's](https://en.wikipedia.org/wiki/Thomas_Kuhn) highly influential book,
The Structure of Scientific Revolutions, added a new congruent meaning to the word: 
"a paradigm is a distinct set of concepts or thought patterns, including theories, 
research methods, postulates, and standards for what constitutes legitimate contributions to a field."
Or, more simply: "A world view underlying the theories and methodology of a particular subject."

What he noticed was that scientific progress was not simply an accumulation of theories proven
or disproven by experimental evidence. Instead, it is periodically punctuated by
"paradigm shifts" in the underlying model that govern the work that scientists collectively perform.
Such shifts, such as the shift in Physics from Newtonian classical mechanics to Einstein's
relativistic theories, fundamentally alter what problems scientists want to tackle,
how they go about doing so, and even the vocabulary and concepts they use to talk about it.

The catalyst for such shifts can vary. A common scenario is that some well-established
paradigm starts to accumulate notable flaws. The initial response is to paper over these
flaws with ever more complex work-arounds. Eventually, the complexity of the work-arounds
becomes so unwieldy that one or more visionaries begin searching for deeper and more valuable
concepts that better explain not only what we know, but also all of what does not seem to fit well,
and even phenomena we have not yet observed but can now predict. 

Some shifts are evolutionary, sometimes leading to subspecialization within a domain.
Other shifts, however, may be revolutionary in nature.
Such significant paradigm shifts can lead to
"incommensurability" disconnects between the adherents of different paradigms in the same field.
As a result, colleagues sometimes struggle to communicate or collaborate effectively
because they use different words (or worse, the same words in different ways).
and because they value and reward different normative goals as more worthy of pursuit.

Because of differences in concepts and subjective values, declaring one paradigm superior to another
can be difficult or impossible to do on a purely objective basis. 
That said, Kuhn enumerated several objective criteria can could aid in that assessment:
accuracy, consistency, broad scope, Occam simplicity, and fruitfulness.

### Robert Floyd: The Paradigms of Programming ###

Kuhn's observations had a profound effect on a broad range of domains beyond
science, including economics and the humanities. It is not surprising that someone
would want to apply this analytical framework to the then-emerging domain of programming.

This was the subject of Robert Floyd's 1978 Turing Award lecture:
[The Paradigms of Programming](https://www.ias.ac.in/article/fulltext/reso/010/05/0086-0098).
Although the paper makes no mention of object-oriented<sup>1</sup> or functional programming
(as neither had yet gained mainstream awareness),
he nonetheless made a number of valuable observations that still resonate for me.

He begins with a remarkably clear explanation of what our programming paradigms
focus on and facilitate, using structured programming (Dijkstra, Wirth, et al) as an example.

> "In the first phase, that of top-down design, or stepwise refinement,
  the problem is decomposed into a very small number of simpler subproblems.
  [...] This gradual decomposition is continued until the subproblems that arise
  are simple enough to cope with directly. [...] Yet further decomposition would yield
  a fully detailed algorithm.
>
> "The second phase ... entails working upward from the concrete objects and functions
  of the underlying machine to the more abstract objects and functions used throughout
  the modules produced by the top-down design. [...] This approach is referred to as
  the method of *levels of abstraction,* or of *information hiding*.
  
This closely tracks with the Kuhnian perspective: an organizing worldview
that strikes at the heart of how we approach complex programming challenges:
the design strategy for formulating the architectural/component structure of
some complex system. Paradigms like structured programming, OOP, FP, actor-oriented programming,
etc. offer different prescriptive approaches to this design process,
often pursuing different values and priorities along the way.

Intriguingly, Floyd goes on to use the term paradigm in a plural and non-worldview sense:
more akin to what we might today call a design pattern or abstract algorithm.
He mentions as examples his discovery of recursive co-routines as a control structure,
Cocke's dynamic programming algorithm for parsing context-free languages,
state-machine paradigm,
the value of using a rules-based paradigm that makes it easier for an expert non-programmer
to incrementally improve the effectiveness of Mycin, a medical diagnostic aid, etc.

Floyd encourages programmers to invest in sharing and expanding their
repertory of such paradigms. To this end, he suggests that programmers study other people's code
and enrich their contact with programs written under alien conventions
(for which he gave, as an example, 
the homoiconicity lessons he learned by visiting the Lisp-dominated MIT lab).

Similarly, he argues that teaching programming should involve the imparting of
paradigms more than specific programming languages. He wants instruction to focus more
on how to solve design problems, versus focusing excessively on "grammar, semantic rules,
and memorizing finished algorithms".

Finally, Floyd offers words of wisdom about the nature of the relationship between paradigms
and programming languages. Instead of promoting Wikipedia's and Harper's distorted understanding of
paradigms as a taxonomic categorization of languages, he offers a worthy challenge:

> "To the designer of programming languages, I say: unless you can support the paradigms
	I use when I program, or at least support my extending your language into one that does
	support my programming methods, I don't need your shiny new languages; like an old
	car or house, the old language has limitations I have learned to live with. To persuade
	me of the merit of your language, you must show me how to construct programs in it. I
	don't want to discourage the design of new languages; I want to encourage the language
	designer to become a serious student of the details of the design process."
	
This too, Floyd illustrates with examples, such as the generate/filter/accumulate paradigm,
a pre-cursor to the iterator combinator abstractions one finds in functional languages,
Rust and even .Net (Linq). When programming languages make it easy to support broadly
applicable design patterns, such iterator combinators, state machines, and co-routines,
programmers becomes more productive and code becomes more readable and maintainable. 
We see this evolution unfold in the continued growth of powerful language abstractions: 
pattern matching, channel-based message passing, asynchronous i/o (async/await) and so many more.

### Revitalizing Kuhn and Floyd in the 21st Century ###

So, how well do these observations apply to the programming challenges that face us now,
in our world of git, microservices, agile, open source, and more?
I think they hold up quite well for this reason: good design remains central to our success. 

Writ large, in the Kuhnian worldview sense, Floyd's notion of programming paradigms still applies
to the systematic methodologies we choose for decomposing a programming problem into its modular components,
stitching them together into ever-larger modules, and then improving the system design
with various abstractions that facilitate reuse and flexibility. 
Our popular programming paradigms, such as OOP (both classical and Kay), FP, Actor-oriented
programming recommend notably different approaches to these decomposition, modularization, and
abstraction activities.

Writ small, Floyd's notion of gathering a rich and growing reportoire of paradigms, or design patterns,
still sounds eminently wise, and immensely more practical than paradigm partisanship.
Also wise is his advice that the design of programming languages should facilitate
the concise representation of code that exploits and composes together these design patterns.
In this way, our paradigms, our design patterns,
have an independent and vital conceptual role that flows forward to influence both our tools
(including programming languages) and the complex systems we build and maintain.

Kuhn's notion of incommensurability is very evident in many of the conversations we witness
between adherents of different paradigms. Too often, they are littered with
frustration ("how do you reason about your program?"), miscommunication
("that's not what subtyping means!") and evangelical partisanship ("Haskell's type classes
bear no similarities whatsoever to OOP's subtype polymorphism").
But for those programmers who are genuinely eager to expand their repertoire of programming
strategies, as Floyd recommends, such conversations and explorations can be both enlightening
and inspiring.

Finally, Kuhn's analysis helps use realize that when Harper tells us to abandon 
programming paradigms (clades) for types (genes),
he is in fact not-so-subtly hawking another paradigm (and book!) which claims that 
use of certain type systems, backed by theory, improves the effectiveness of our designs
in ways we should care about.
We can explore this paradigm and learn from it, while rejecting the harsh partisanship
that discourages the study of any other kind of design strategy.

### Summary and Next Steps ###

I began with a question: What is a programming paradigm?

It is about how we design our complex systems:

- As a world-view, it describes how we should decompose, modularize and abstract
  the working components of our systems.
- At a lower level, it is a versatile collection of design patterns we can use to
  help us quickly write code that solves a wide range of specific design challenges.

Paradigms help us build great designs more easily and they positively influence
improvements to our programming tools.

When we view programming paradigms the way Floyd does, and I think we should, 
we end up with a metaparadigm for programming paradigms that is far more useful than
Wikipedia's metaparadigm of paradigms-as-language-taxonomies. As Kuhn would insist,
Floyd's metaparadigm improves accuracy, consistency, scope, simplicity, and 
(most important to me) fruitfulness.

So, where do we go from here?  In future posts, I want build on this foundation,
by reviewing in more specific detail where we are, paradigmatically speaking.
This means exploring topics like declarative programming, object-oriented programming,
and functional programming in light of the insights raised in this post.
Once we have done that, we are ready to speculate about our journey towards
emerging and future paradigms.

------------
<sup>1</sup>Although I know that Alan Kay invented the term object-oriented programming
sixteen years earlier, I do not know which influential person first called it a paradigm.
I suspect Floyd's paper played a triggering role, as the famous Byte issue on
Smalltalk and object-oriented programming was published only three years later, and
the first OOPSLA conference was held five years after that.

