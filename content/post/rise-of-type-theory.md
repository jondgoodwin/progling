---
title: "The Rise of Type Theory"
date: 2020-04-25T18:02:18+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "General"
  - "Types"
tags:
---

After three dreary posts on syntax, let's change the pace
and pursue an entirely different, deeper and more fun topic!

In our community's #theory channel on [Discord](https://discord.gg/yqWzmkV), 
someone asked: "Is there a basis for some of the mathematics related to type theory 
and its relationship to its usage in programming languages?"
This triggered a spirited dialogue about the historical antecedents
of type theory, how type theory evolved from that, and the interplay
between programming languages and type theory.

My dim memory was that this process unfolded like a bubbling stew
between mathematicians and computer science practitioners,
colleagues throwing flavors back and forth, 
exploring different ways to formalize interrelated concepts.
Very little of it proceeded in a straight line, and sometimes it more
closely resembled a drunken walk.
To confirm this memory, I scavenged through Wikipedia
and assembled a ordered, dated list of some seminal events from the
20th and 21st centuries.

This post expands on that list with additional contextual commentary and links
to relevant pages that go into considerably more depth.
I freely acknowledge that my treatment here is woefully incomplete
and simplistic, and always welcome suggestions.
Nonetheless, I hope others find this history as intriguing and inspiring as I do.

## The Murder of Hilbert's Program ##

*In the early 20th century, prior to World War II, seminal advances
to Mathematics unfolded, whose core concepts still play
a dominant role in type theory...*

1900 [Hilbert's problems](https://en.wikipedia.org/wiki/Hilbert%27s_problems). David Hilbert
challenges mathematicians with 23 problems to solve.
Following Gottlob Frege and Bertrand Russell, Hilbert sought to 
define mathematics logically using the method of formal systems.

1901 [Russell's paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox).
Russell demonstrates that propositions in Georg Cantor's set theory can lead
to contradiction. If R is the set of all sets that are not members of themselves,
R can neither be a member of itself, nor can it not.

1908 [Russell's theory of types](https://en.wikipedia.org/wiki/Type_theory#History).
One solution to the paradox is to create a hierarchy of types,
such that types can only be built from types lower in the hierarchy.

1910-13 [*Principia Mathematica*](https://en.wikipedia.org/wiki/Principia_Mathematica) (Whitehead and Russell).
Among its ambitious aims: precisely express mathematical propositions in symbolic logic using the most convenient notation.
Enormously influential, it showcased the power of symbolic logic applied to mathematics.

1920 [Hilbert’s Program](https://en.wikipedia.org/wiki/Hilbert%27s_program).
This challenge sought a formal, finitistic proof of the consistency of the axioms of arithmetic.
This is an outgrowth of Hilbert's second problem.

1925 *On the principle of the excluded middle* (Kolmogorov).
showing that formal logic statements can be formulated using 
[intuitionistic logic](https://en.wikipedia.org/wiki/Intuitionistic_logic).
Along with independent work, this is now known as the
[Brower-Heyting-Kolmogorov Interpretation](https://en.wikipedia.org/wiki/Brouwer%E2%80%93Heyting%E2%80%93Kolmogorov_interpretation),
a cornerstone principle of intuitionistic type theory.

1928 [Entscheidungsproblem](https://en.wikipedia.org/wiki/Entscheidungsproblem) (Hilbert and Ackermann).
An outgrowth of Hilbert's program, this decision problem challenge looks for an algorithm that considers, as input, 
a statement and answers "Yes" or "No" according to whether the statement is universally valid

1931 [Gödel’s Incompleteness Proof](https://en.wikipedia.org/wiki/G%C3%B6del%27s_incompleteness_theorems).
Kurt Gödel proves that any consistent formal system supporting elementary arithmetic 
is incomplete: statements in this system can neither be proved nor disproved.
Further, such a system could not prove its own consistency; 
it cannot prove the consistency of anything stronger with certainty.
This result was cataclysmic, resulting in the doom of Hilbert's program.

1932 *Symbolic Logic* (Lewis). This introduced five systems of modal logic to predicate calculus,
qualifying the truthfulness of a proposition by some notion of possibility or necessity.

1934 [Combinatory Logic](https://en.wikipedia.org/wiki/Combinatory_logic) (Curry).
Haskell Curry observes that the types of the combinators 
could be seen as axiom-schemes for intuitionistic implicational logic.

1935 [Natural Deduction](https://en.wikipedia.org/wiki/Natural_deduction) and 
[Sequent Calculus](https://en.wikipedia.org/wiki/Sequent_calculus) (Gentzen).
These equivalent systems substantially improved the notation and structure of proofs.
They proposed  ∀ for universal quantification, introduced normalization,
and showed that proof rules come in pairs (introduction and elimination).
Even though sequent calculus was invented to introduce cut elimination to natural deduction, 
both systems thrive.

1936 [Church-Turing Thesis](https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis).
Alonzo Church and Alan Turing independently reproduce Gödel’s result using their own formalisms.
Church, inventing [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus), 
proves that Hilbert's Entscheidungsproblem is unsolvable.
Turing's theorem, based on the [Turing Machine](https://en.wikipedia.org/wiki/Turing_machine), 
shows that there is no algorithm 
to solve the halting problem. Turing shows both results are equivalent.

1940 [Simply Typed Lambda Calculus](https://en.wikipedia.org/wiki/Simply_typed_lambda_calculus) (Church).
To avoid paradoxical uses, Church adds types to his untyped lambda calculus,
much as Russell used a hierarchy of types to resolve set paradoxes.

1942-5 Early [Category Theory](https://en.wikipedia.org/wiki/Category_theory) (Ellenberg and McLane).
Their work extended algebraic topology with categories, functors, etc.
Much later, this work was incorporated into type theory.

*Despite the dramatic failure of Hilbert's program, 
the notational and procedural advances made to formal logic and reasoning
were immensely valuable and are still relevant.
No less valuable was learning from Gödel that there are inscrutable
limits to how much we can expect to accomplish with formal logic.
We preserve this tradition when we pass nightmares on to those students
we caution about the danger of [Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness)
programming languages (and Conway's Game of Life)!
Let's return to this topic in a future post.*

## Programming Languages and Types ##

*[Eniac](https://en.wikipedia.org/wiki/ENIAC), the first general-purpose
general computer, arrived in 1945. After that, the proliferation of computers
was explosive. Given the tediousness of programming these computers in binary code,
it did not take long before the necessity of high-level programming
languages became widely recognized.*

*In the list below, I show the introduction of some notable programming languages.
Importantly, I also show how they varied in the datatypes they supported,
as types are inseparable from programming languages. Working with these languages
provided an invaluable laboratory for exploring the universe of practical types.
Importantly, many of the inventors of these programming languages
were either mathematicians or trained by them.
The inventors also collaborated and learned from each other.*

1942-5 [Plankalkül](https://en.wikipedia.org/wiki/Plankalk%C3%BCl) (Zuse). 
First high-level programming language to be designed for a computer.

1947 Curry describes an early high-level programming language.

1956 [Fortran](https://en.wikipedia.org/wiki/Fortran) (Backus).
This started with support for numbers, but later versions added
many data types, including array, complex numbers, and logical (1961).

1957 [Temporal Logic](https://en.wikipedia.org/wiki/Temporal_logic) (Prior).
This adds invaluable qualifiers to modal logic to express concepts like
always, eventually, and the sequential order of states.

1958 [Lisp](https://en.wikipedia.org/wiki/Lisp_(programming_language)) (McCarthy).
This lambda calculus-inspired dynamic-typed language 
offered support for numbers, symbols and heterogeneous lists.
In this, it improved on [IPL](https://en.wikipedia.org/wiki/Information_Processing_Language) (Newell and Simon).

1958 [Algol-58](https://en.wikipedia.org/wiki/ALGOL) (team).
This is the first language to combine seamlessly imperative effects 
with the (call-by-name) lambda calculus.
It started with numbers and arrays (supporting upper and lower subscript bounds).
It was followed later by Algol-60 (where Backus-Naur form was introduced) and Algol-68.  

1958 [System T](https://en.wikipedia.org/wiki/Dialectica_interpretation) (Gödel).
His motivation was to obtain a relative consistency proof
for Heyting arithmetic (and hence for Peano arithmetic).
It was later used to build a model of Girard's refinement of intuitionistic logic known as linear logic.
Pure math work continues and Gödel continues to deliver.

1958 *Combinatory Logic* (Curry). 
This book observes that Hilbert-style deduction systems (proofs) coincide 
to the typed fragment of combinatory logic (computations).
A decade later, Howard rediscovers and elaborates on this important correspondence.

1959 [Kripke Semantics](https://en.wikipedia.org/wiki/Kripke_semantics) (Kripke).
These considerably enriched the useful formalisms of modal logic.

1959 [Cobol](https://en.wikipedia.org/wiki/COBOL) (Hopper). Introduced records, along
with other familiar datatypes.

1962-7 [Simula](https://en.wikipedia.org/wiki/Simula) (Dahl and Nygaard).
The progenitor of object-oriented languages introduced classes, inheritance, etc.

1964 [Basic](https://en.wikipedia.org/wiki/BASIC) (Kemeny and Kurtz). 
A novice-friendly language supporting numbers, matrices and strings.

1966 [APL](https://en.wikipedia.org/wiki/APL_(programming_language)) (Iverson).
Array programming language with an unusual collection of succinct operators.

## From Type Algebra to the Lambda Cube

*The switch in sections reflects no pace change in the introduction
of influential languages and their type systems. It is sparked by
the explosively catalytic re-realization of the deep connection between
natural deduction (proof systems) and intuitionistic (calculation) systems and, later,
typed lambda calculus. The exciting implications of this result led to
a rapidly deepening formalization of type theory as a robust, distinct academic discipline.
Increasingly, we see insights flow back-and-forth not only
between type theory and programming languages (and theorem provers), 
but also between type theory and related branches of mathematics, 
such as category theory, topology, and abstract algebra.*

1969 [Curry-Howard Correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence).
Howard's notes highlight the tight correspondence between the natural deduction connectives &, ∨, ⊃ 
and the intuitionistic, computational types ×, +, → 
(product, sum, and lamba types),
as well as demonstrating the correspondence of cut elimination to normalization.
It goes on to propose new types that correspond to
the predicate quantifiers ∀ and ∃, now called dependent types.
Although not published until 1980, this was known by and influenced the work of
Martin-Löf, Girard, Coquand, and others. 
Wadler's paper [Propositions as Types](https://homepages.inf.ed.ac.uk/wadler/papers/propositions-as-types/propositions-as-types.pdf)
offers an invaluable summary of the historical context and impact of Howard's insight.

1969 [Hoare Logic](https://en.wikipedia.org/wiki/Hoare_logic).
A formal system for reasoning rigorously about the correctness of programs.

1971-79 [Intuitionistic Type Theory](https://en.wikipedia.org/wiki/Intuitionistic_type_theory) (Martin-Löf).
Building off the BHK Interpretation and inspired by the Curry-Howard Correspondence,
this establishes an intuitionistic (vs. classical) system for type theory.
It derives its power from just three types (0, 1, 2) and five constructors
(Σ, Π, =, inductive, and universes), making possible dependent type judgments.

1972 [C](https://en.wikipedia.org/wiki/C_(programming_language)) (Ritchie). 
This systems language supported pointers, struct, union, ,and array datatypes.
It took nearly three decades to formalize a logic around the safe use of pointers.

1972-4 [System F](https://en.wikipedia.org/wiki/System_F) (Girard and Reynolds)
adds universal quantification (polymorphism) to typed lambda calculus.
Intriguingly, this makes type inference undecidable.

1973 [ML language](https://en.wikipedia.org/wiki/ML_(programming_language)) (Milner).
ML is the progenitor of type theory-based programming languages.
Its types closely mirror those of type theory and its ongoing
improvements helped influence the evolution of type theory and 
the [LCF](https://en.wikipedia.org/wiki/Logic_for_Computable_Functions) theorem prover.
Among its initial innovations were the Hindley-Milner type system, pattern matching, 
first-class functions, sum types, and product type tuples.
Data types were later broadened to parameterized types (1978) and ref types (1980).

1974 [CLU](https://en.wikipedia.org/wiki/CLU_(programming_language)) (Liskov).
This language was first to introduce parameterized types with constraints (generics)
and abstract data types. It also featured variant types, classes, 
iterators, exception handling, and more.

1975 [Scheme](https://en.wikipedia.org/wiki/Scheme_(programming_language)) (Steele and Sussman).
This Lisp dialect strengthed the recursive nature of functional programming
with tall-call optimization. It also introduced first-class continuations,
later tied back using type theory to Pierce's law in classical logic.

1978 [Communicating Sequential Processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes) (Hoare),
a formal language for describing patterns of interaction in concurrent systems.

1980 [Calculus of Communicating Systems](https://en.wikipedia.org/wiki/Calculus_of_communicating_systems) (Milner),
which models indivisible communications between two participants. This was later expanded to become
[π-calculus](https://en.wikipedia.org/wiki/%CE%A0-calculus).
[Session types](http://simonjf.com/2016/05/28/session-type-implementations.html) for channels/protocols arise from process calculi.

1980 [Smalltalk](https://en.wikipedia.org/wiki/Smalltalk) (Kay).
Inspired by Simula, this dynamically-typed, object-oriented language facilitated
Xerox Parc's innovations that fundamentally transformed how we interact with computers.
Its message passing architecture helped inspire 
the [actor model](https://en.wikipedia.org/wiki/Actor_model),
which went on to influence π-calculus.

1983+ [Standard ML](https://en.wikipedia.org/wiki/Standard_ML).
This improved dialect added MacQueen's lauded module system (structures, signatures and functors),
as well as labelled records and unions.

1985 [C++](https://en.wikipedia.org/wiki/C%2B%2B) (Stroustrup).
Initially, a C dialect that added rich object-oriented abstractions.

1985 [Structure and Interpretation of Computer Programs](https://en.wikipedia.org/wiki/Structure_and_Interpretation_of_Computer_Programs)
(Abelson and Sussman) This influential textbook used Scheme to
teach about fundamental principles of recursion, abstraction, state, modularity,
and programming language design and implementation.

1986 [Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language)) (Armstrong).
A general-purpose, concurrent, functional programming language
which supports distributed, high-availability applications 
using immutable, garbage-collected data and message-passing processes (actors).

1987 [Linear Logic](https://en.wikipedia.org/wiki/Linear_logic) (Girard).
This substructural logic applies contraction and weakening rules to sequent calculus.
Its potential for deterministic resource management was heavily boosted in Wadler's papers
(e.g., *Linear Types can change the world*).

1987 [Behavioral Subtyping](https://en.wikipedia.org/wiki/Liskov_substitution_principle) (Liskov).
Influenced by Hoare Logic and inspired by object-oriented languages,
this logic formalizes the semantic interoperability of types in a hierarchy.

1988 [Effect Systems](https://en.wikipedia.org/wiki/Effect_system) (Lucassen and Gifford).
Proposes the use of annotations that support a compile-time check 
of the computational side-effects of a program.

1989 [Calculus of Constructions](https://en.wikipedia.org/wiki/Calculus_of_constructions) (Coquand).
Yet another formal system of type theory based on higher-order typed lambda calculus.
It forms the basis behind the [Coq](https://en.wikipedia.org/wiki/Coq)
interactive theorem prover.

1990 *Definition of SML* (Milner, Tofte, Harper).
The first formal specification of a widely-used language,
presented in terms of its grammar, typing rules and operational semantics.

1990 [Haskell](https://en.wikipedia.org/wiki/Haskell_(programming_language)) (Wadler)
A general-purpose, statically typed, purely functional programming language 
with type inference, lazy evaluation, and ad hoc polymorphism (type classes).
Haskell made monads, used for managing side effects in a pure way, famous.

1991 [Lambda Cube](https://en.wikipedia.org/wiki/Lambda_cube) (Berendregt).
This cube organized various lambda systems along three axes, corresponding
to the binding between types and terms: polymorphism (terms bind types),
dependent types (types bind terms), and type operators (types bind types).

1991 [Refinement Types](https://en.wikipedia.org/wiki/Refinement_type) (Freemand and Pfenning).
A predicate-based extension of behavioral subtyping for preconditions and postconditions.

1994 [System F<](https://en.wikipedia.org/wiki/System_F-sub) (Cardelli).
This expands System F lambda calculus with subtyping for parametric polymorphism
and record subtyping.

*At this point, much of the core foundation for type theory have been poured.
Type theory is revealed to be not a singular theory,
but a diverse and inclusive family comprised of many formal systems,
including not only several flavors of lambda cube systems (classical and intuitionistic),
but also process calculi, subtyping, linear logic, modal/temporal logic, effect systems,
and separation logic (future).*

## Going Mainstream ##

*When history gets recent, it becomes harder to discern what milestones
represent far-reaching influence. That said, let's highlight several events
that not only demonstrate further development of type theory and unique
application of type systems to programming languages and proof systems,
but also mention influential resources that broaden awareness of these rich and powerful tools.*

1997 [Region-Based Memory Management](https://en.wikipedia.org/wiki/Region-based_memory_management)
(Tofte and Talpin). This work, implemented using a dialect of ML called MLkit,
demonstrated new static-based inference and analysis techniques for rapidly allocating
objects in arena-based regions that outlive their static scope.

1999 [ATS](https://en.wikipedia.org/wiki/ATS_(programming_language)) (Xi).
This unifies programming with formal specification, combining compile-time theorem proving
with advanced type systems (including dependent and linear types).

2000 [Separation Logic](https://en.wikipedia.org/wiki/Separation_logic) (Reynolds et al).
This work formalized local reasoning about safe reference-based
use of shared, mutable memory areas, facilitating information hiding,
transfer of ownership, and virtual separation between concurrent modules.
This logic can help verify programming languages that offer versatile
reference-based memory and data race safety mechanisms.

2002 *Types and Programming Languages* (Pierce). 
This highly-recommended textbook provides a comprehensive introduction to type systems
and the basic theory of programming languages.
It covers simple types, subtyping, recursive types, polymorphism, and higher-order systems.

2006 [Homotopy Type Theory](https://en.wikipedia.org/wiki/Homotopy_type_theory) (Voevodsky et al).
This promising technique links topology to type theory,
using paths to explore deeper notions of equivalence.

2006 [Cyclone](https://en.wikipedia.org/wiki/Cyclone_(programming_language)) (Grossman et al).
This research project built a safe C dialect that used higher-order types,
linear logic, and region polymorphism to statically remove many safety holes in C.

2007 [Idris](https://en.wikipedia.org/wiki/Idris_(programming_language)) (Brady) 
A Haskell-like purely functional, lazy programming language that supports dependent types
and can be used as a proof assistant.

2007 [Agda](https://en.wikipedia.org/wiki/Agda_(programming_language)) (Norell & Coquand).
This is a dependently typed functional programming language that can also be used
as a proof assistant, where proofs are written in a functional programming style.

2010 [Rust](https://en.wikipedia.org/wiki/Rust_(programming_language)) (Hoare et al)
An increasingly popular and safe systems language that
heavily leverages linear logic, lifetimes, and fearless concurrency.

2012 *Practical Foundations for Programming Languages* (Harper). 
Another highly-recommended textbook that not only deeply covers type systems,
but also dynamics, classes and inheritance, control flow, mutable state,
parallelism/concurrency, and modularity.

2015 [Pony](https://www.ponylang.io/) (Clebsch).
This actor-based programming language offers a unique collection of
object and reference capabilities and a lockless design
to offer proven safety guarantees.
 
2020 [Granule](https://granule-project.github.io/).
A functional programming language based on the linear λ-calculus 
augmented with graded modal types, 
inspired by the coeffect-effect calculus of Gaboardi et al..