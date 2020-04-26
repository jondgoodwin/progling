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
colleagues that often knew each other constantly throwing flavors back and forth, 
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
Along with independent work, this leads to the
[Brower-Heyting-Kolmogorov Interpretation](https://en.wikipedia.org/wiki/Brouwer%E2%80%93Heyting%E2%80%93Kolmogorov_interpretation).

1928 [Entscheidungsproblem](https://en.wikipedia.org/wiki/Entscheidungsproblem) (Hilbert and Ackermann).
An outgrowth of Hilbert's program, this decision problem challenge looks for an algorithm that considers, as input, 
a statement and answers "Yes" or "No" according to whether the statement is universally valid

1931 [Gödel’s Incompleteness Proof](https://en.wikipedia.org/wiki/G%C3%B6del%27s_incompleteness_theorems).
Kurt Gödel proves that any consistent formal system supporting elementary arithmetic 
is incomplete: statements in this system can neither be proved nor disproved.
Further, such a system could not prove its own consistency; 
it cannot prove the consistency of anything stronger with certainty.
This result was cataclysmic, resulting in the doom of Hilbert's program.

1934 [Combinatory Logic](https://en.wikipedia.org/wiki/Combinatory_logic) (Curry).
Howard Curry observes that the types of the combinators could be seen as axiom-schemes for intuitionistic implicational logic.

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
We preserve this tradition when we pass nightmares on to the students
we caution about the danger of [Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness)
programming languages (or even Conway's Game of Life).
Let's return to this topic in another post.*

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

1958 [Lisp](https://en.wikipedia.org/wiki/Lisp_(programming_language)) (McCarthy). 
Dynamic-typed language supporting numbers, symbols and heterogeneous lists.

1958 Algol-58, followed later by Algol-60 and Algol-68.  the first language to combine seamlessly imperative effects with the (call-by-name) lambda calculus.

1958 System T (Godel) whose motivation was to obtain a relative consistency proof
for Heyting arithmetic (and hence for Peano arithmetic).
It was later used to build a model of Girard's refinement of intuitionistic logic known as linear logic

1958 Curry observes that a certain kind of proof system, referred to as Hilbert-style deduction systems, coincides on some fragment to the typed fragment of a standard model of computation known as combinatory logic.

1959 Cobol Programming Language (Hopper). Introduced record/product types, etc.

1962-7 Simula (Dahl and Nygaard) supporting classes, inheritance, etc.

1964 Basic Programming Language (Kemeny and Kurtz). Support for numbers, matrices and strings

1966 APL (Iverson) Array programming language

## From Type Algebra to the Lambda Cube

1969 Curry-Howard Correspondence. Howard observes that a "high-level" proof system, referred to as natural deduction, can be directly interpreted in its intuitionistic version as a typed variant of the model of computation known as lambda calculus

1972+ C programming language (Ritchie). Integers, pointers, array, struct, union, etc.

1972-4 System F (Girard and Reynolds) Polymorphic Lambda Calculus. 2nd order lambda calculus

1973 ML language (Milner). Hindley-Milner type inference system, pattern matching, first-class functions, tuples, lists, integers. 1978: parameterized types (post-CLU). 1980 ref type 

1974 CLU programming language (Liskov). Parameterized types with constraints (generics), abstract data types, variant types, classes, iterators, exception handling, etc.

1975 Scheme (Steele and Sussman)

1979-82 Intuitionistic Type Theory (Martin-Lof) addressing dependent type judgments

1983+ Standard ML: labelled records/unions, module system (MacQueen) 

1984 Common Lisp

1985 C++ (Stroustrup)

1985 Structure and Interpretation of Computer Programs (Abelson and Sussman) Influential textbook, with Scheme at its core

1987 Linear Logic (Girard) 1990 Linear Types can change the world (Wadler)

1989 Coq (Coquand) Interactive theorem prover

1990 Definition of SML (Milner, Tofte, Harper)

1990 Haskell programming language (many) pure, lazy language, w/ type classes (Wadler)

1991 Lambda Cube (Berendregt)

1994 System F< (Cardelli) Lambda calculus with subtyping

## Going Mainstream ##

1996 OCaml (Leroy et al) SML with object-oriented capabilities

1997 Region-Based Memory Management (Tofte and Talpin) Also MLkit

1999+ ATS programming language (Xi) designed to unify programming with formal specification

2000-2002 Separation Logic (Reynolds et al)

2002 Types and Programming Languages (Pierce). Popular textbook.

2006+ Homotopy Type Theory (Voevodsky and many others)

2006 Cyclone (Grossman et al), A safe C dialect with regions (including linear)

2007 Idris (Brady) Pure FP language for dependent types

2010 Rust (Hoare et al)

2012 Practical Foundations for Programming Languages (Harper). Standard textbook
 
