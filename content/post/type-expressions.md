---
title: "Unifying Type and Value Expressions"
date: 2020-04-21T11:27:31+10:00
draft: false
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

I seem to be on a roll with syntactic concerns lately, which is not
where I typically spend most of my time.
Today's issue is, at least, more unusual than stylistic preferences
regarding [semicolons](/post/semicolon-inference) and
[significant indentation](/post/significant-indentation).
This post explores the issues that arise when
teaching the compiler to be agnostic as to whether it is parsing
a value expression or a type expression.

A value expression computes some typed value when evaluated:

    2 * radius * pi

A type expression denotes some specific type:

    struct Point {
	    x: i32;
		y: i32;
	}
	
The grammar rules of most programming languages vary significantly between value expressions
and type expressions. Value expressions often support dozens
of unary, infix and ternary operators with well-defined precedence.
Type expressions typically use fewer operators, if any. 
Even when type and value expression support the same operators,
they may not do so in syntactically equivalent ways,
nor be subject to the same precedence rules.

Most of the time, a language's grammar clearly distinguishes when it expects a
value expression vs. a type expression. For example, the conditional expression on
an `if` or `while` statement expects a value expression.
Similarly, the declaration of a variable supports the specification
of a type expression before or after the variable's name,
thereby declaring the variable's type explicitly.

For some languages, however, it is impossible or inconvenient
for a parser to know, based on the grammatic context, whether
the expression it is parsing is value-based or type-based.
For such languages, we need to converge the syntactic
structure of expressions such that the parser can handle either gracefully,
and then resolve which it is during semantic analysis.

This post discusses what conditions might provoke a language to
pursue syntactic unification of value and type expressions.
It then highlights how to accomplish that unification
on a specific systems programming language 
([Cone](http://cone.jondgoodwin.com/)).

## Why Unify? ##

Any language that supports first-class types (types as values you can pass around)
must unify type and value expression syntax.
This holds true for languages that support dependent types
(e.g., notice the Pies syntactic production rules for
[Idris](http://docs.idris-lang.org/en/latest/reference/syntax-reference.html)).
It also holds true for most dynamically-typed languages (e.g., Ruby),
whose types are not just first-class, but are also mutable!

However, you need not unify type and value expression syntax if
the language's grammar makes it unambiguous when to expect
a value expression and when to expect a type expression.
For most of Cone's grammar, this is true. Type expressions
are relatively rare and happen only in certain places,
such as defining types, declaring the types of variables or functions,
and after various operators for converting a value's type (e.g., `number into u32`).

It was when I added generics to Cone that I noticed I had a problem.
I use square brackets instead of angle brackets for a generic's arguments.
This choice means the semantics for `label[expr1, expr2]` varies depending on
whether label is a variable, type or generic.
If label is a generic, then expr1 and expr2 are probably type expressions,
otherwise they must be value expressions.
Since the compiler has no way to whether label is a generic until semantic analysis,
the parser has to be flexible to correctly parse expr1 and expr2 as either
a value or type expression.

Even this would not be much of a problem if all types were nominal (e.g., Java).
When types are nominal, it means a type expression is just the name of the type.
The parser could handle a type name the same way as a variable name,
and then resolve it properly during semantic analysis.
This is not possible with Cone, because
Cone supports a rich collection of structural types
which employ various operators to form a type expression.

There are several approaches one could take to handle this syntactic ambiguity.
I could use a back-tracking parser. I could add extra syntax
to mark when an expression should be treated as a type expression
(e.g., preceding a type expression with a colon).
Those approaches do not appeal to me, as they increase complexity for everyone.
Instead, I would like to examine whether it is possible to
design a converged syntactic grammar that gracefully supports both
type and value expressions in as synergistic and simple way.


## Unification for Cone ##

To accomplish unify these grammars, let's examine each kind of type expression, one after another,
and prpose how to resolve any grammar conflicts with value expressions
that employ similar syntax.

### Nominal Types ###

Let's begin with the various nominal types: struct, tuple, enum, number types, etc.
As already mentioned, unification here is quite straightforward,
as nominal types cleanly distinguish between the name-based definition of the type
and its use. The definition of a nominal type is a statement.
As such, it is not a type expression, and cannot be used within one.

Type expressions need only refer to a nominal type by its name.
A nominal type's name may be qualified by the module it belongs to
(e.g., `Internet::HTML::Title`).
If the type is a generic, the name will be followed by generic parameters
that monomorphize the type (e.g., `Vec[i32]`). Here is the EBNF:

    qual-name ::= (modname '::')* name ('[' parm (',' parm)* ']')?

The good news is this exact syntax for module-qualified and generic-specialized
names applies in the same way when the name represents a variable or function,
instead of a type. So, the compiler can parse
either in exactly the same way, and then resolve later whether the name represents
some declared type, variable or function.

### Tuples ###

Tuples offer another encouraging win.
A type tuple is a list of types separated by commas:

    type-tuple ::= type (',' type)*
	
Good news! That's exactly the same syntax used by value-based tuples.
So, the parser can parse a type tuple or a value tuple in exactly the same way,
and then distinguish which it has later on during semantic analysis.

Further solidifying their isomorphism, is that both type tuples
and value types appreciate being wrapped in parentheses when they
need to preserve their coherence after being thrown into a sea of commas
that serve a different master.

Would that they all would be this easy!

### References ###

References in Cone are fortunately another straightforward win.
Here is the syntax for a reference type:

    ref-type ::= '&' ('<' | [])? region-type? lifetime? perm-type? type
	
Let's walk through these elements which are unique to Cone:

- The ampersand signifies a reference type expression. There are three kinds
  of reference types:  regular ones ('&'), virtual references ('&<') and
  array references ('&[]'). The latter two make use of fat pointers that
  include a pointer to the vtable or the number of elements in the slice.
  
- The region type specifies the memory management region responsible for
  allocating the object pointed-to by this object (e.g., `so` for single-owner).
  If a region is not specified, this corresponds to a borrowed reference.
 
- The lifetime is an annotation (e.g., `'a`) that can be placed on any reference
  (though typically a borrowed reference). It helps ensure memory safety.
  
- The permission constrains how a reference may be used, such as indicating
  whether the object it points to may be read or mutated. Specifying this
  is also optional, and a context-sensitive default is assumed when omitted.
  
- The final type describes the type of the object the reference points to.
  
Intriguingly, Cone's uses nearly the same syntax for reference constructors.
When the region is specified on a reference constructor, the constructor
effectively allocates a new object and returns a reference to it.
Otherwise, the reference constructor is used to borrow a reference to something.
Unsurprisingly, the type of the reference is isomorphical to the
specifications on the constructor.

There are two key differences between a reference type and a reference constructor.
A constructor expects an expression instead
a type as its last term. Also, it makes no sense for a reference constructor
to specify a lifetime annotation.

Despite these differences, they are similar enough grammatically that
the compiler can parse either in the same way and sort out the differences later.

### Function ###

Every function has a signature, which is effectively its type.
The signature portion of a function is:

    fn-sig ::= 'fn' name? '(' parm-dcl? (',' parm-dcl)* ')' type

Inside the parentheses are the names and types of the parameters.
The function's return type (which might be a tuple) follows.

In most cases, it does not make sense to have function signatures be part of
a type expression, because functions are not values we can move around a program.
However, it *does* make sense to allow function signatures as the referred-to
type of a regular reference or pointer. This defines a borrowed reference to a
function that takes two integers and returns one:

    &fn (x i32, y i32) i32

Wonderfully, this same syntax applies when using `&` as a reference
constructor, so long as we also provide the implementation.
This effectively creates a reference to an anonymous function:

    &fn (x i32, y i32) i32 {x + y}

### Closures ###

In Cone, a closure is sugar for a struct that not only holds the state of the closure,
and also implements the callable method named `()`.
Since a closure holds state, it is possible to move the value piece of it around
locally, but this is usually awkward. As a result, a value-based closure usually stays
local and works just fine, often with implicit typing.

However, when one does wish to move a closure around to other functions,
this is best done with a virtual reference, built around an ad hoc
trait that captures the signature of the function. 
The sugar for defining a virtual reference to this ad hoc trait is
very similar to that of a reference to a function:

    &<fn (i32, i32) i32


### Lifetime-constrained type ###

    'a Xyz
	
### Pointer ###

### Optional type ###

### Array ###

Cannot use [index] type. Instead use, comingles with array "literal"

## Precedence ##