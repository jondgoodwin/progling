---
title: "Unifying Type and Value Expressions"
date: 2020-04-06T11:27:31+10:00
draft: true
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Syntax"
  - "Types"
tags:
  - "Cone"
---

A value expression computes some typed value when evaluated:

    2 * radius * pi

A type expression defines some specific type:

    struct Point {
	    x: i32;
		y: i32;
	}
	
For many programming languages, the grammar rules vary significantly between value expressions
and type expressions. Value expressions often feature dozens
of unary, infix and ternary operators with well-defined precedence.
Type expressions typically use fewer operators, if any. 
Even when type and value expression support the same operators,
they may not do so in syntactically equivalent ways,
nor be subject to the same precedence rules.

In many cases, a language's grammar clearly distinguishes when it expects a
value expression vs. a type expression. For example, the conditional expression on
an `if` or `while` statement expects a value expression.
Similarly, the declaration of a variable often supports the specification
of a type expression just before or after the variable's name,
thereby declaring the type of the variable.

For some languages, however, it is impossible or inconvenient
for a parser to know, based on the grammatic context, whether
the expression it is parsing is value-based or type-based.
For such languages, it becomes necessary to converge the syntactic
structure of expressions so that the parser can handle either gracefully,
and then resolve which it is during semantic analysis.

This post discusses what conditions might provoke a language to
pursue syntactic unification of value and type expressions.
It then demonstrates techniques for accomplishing that unification
on a specific systems programming language 
([Cone](http://cone.jondgoodwin.com/)).

## Why Unify? ##

One must unify type and value expression syntax for
any language that supports first-class types (types as values you can pass around).
This holds true for languages that support dependent types
(e.g., notice the Pies syntactic production rules for
[Idris](http://docs.idris-lang.org/en/latest/reference/syntax-reference.html)).
It also holds true for most dynamically-typed languages (e.g., Ruby),
whose types are not just first-class, but are also mutable!

You do not need to unify type and value expression syntax if
the language's grammar makes it unambiguous when to expect
a value expression and when to expect a type expression.
For most of Cone's grammar, this is true. Type expressions
are relatively rare and happen only in certain places,
such as defining types, declaring the types of variables or functions,
and after the operators for converting a value's type (e.g., `number into u32`).

It was when I added generics to Cone that I noticed I had a problem.
I use square brackets instead of angle brackets for a generic's arguments.
This means the semantics for `label[expr1, expr2]` varies depending on
whether label is a variable, function, type or generic.
The compiler has no way to know what label is until semantic analysis,
which means the parser has no way to know whether expr1 is a type or a value expression.

Even that would not be much of a problem if all types were nominal (e.g., Java).
When they are nominal, it means a type expression is really just the name of the type,
which means it could just be treated as if it were a variable by the parser,
and then resolved properly during semantic analysis.
However, Cone supports a rich collection of structural types,
with a variety of operators that may be used to form a type expression.

## Unification for Cone ##

There are alternative paths I could take with Cone to resolve this challenge
some other way. I could use a back-tracking parser. I could add extra syntax
to mark when an expression should be treated as a type expression
(e.g., preceding a type expression with a colon).
But for the purpose of this post, let's examine what
it might take to come up with a converged syntax that the parser
could use to gracefully accommodate either a type expression or a value expression
without backtracking.

To accomplish this, let's examine each kind of type expression, one after another,
and analyze how to resolve any grammar conflicts with any value expressions
using similar syntax.

### Tuples ###

Let's begin with tuples, because this is an encouraging instant win.
A type tuple is simply a list of types separated by commas. Its EBNF is:

    type-tuple ::= type (',' type)*
	
Good news! That's exactly the same syntax used by value-based tuples.
So, the parser can parse a type tuple or a value tuple in exactly the same way,
and then distinguish which it has later on during semantic analysis.

Would that they all would be this easy!

### Structs ###

Structs are nominal and not allowed in type expressions, except by their name.

### References ###

&region lifetime permission vtype

Single node, allow lifetime, but is an error for a value expression

### Array ###

Cannot use [index] type. Instead use, comingles with array "literal"

### Generic ###

### Function / closure ###

fn () 

## Precedence ##