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

    ?&i32, f32    // a tuple: optional reference to an integer and a float
	
The grammar rules of most programming languages vary significantly between value expressions
and type expressions. Value expressions often support dozens
of unary, infix and other operators with well-defined precedence.
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
When types are nominal, it means a type expression is pretty simple, oriented around the name.
The parser could handle a type name the same way as a variable name,
and then resolve it properly during semantic analysis.
This is not possible with Cone, because
Cone supports a rich collection of structural types
which employ various operators to form a type expression.

There are several approaches one could take to handle this syntactic ambiguity.
One could use a back-tracking parser. One could add extra syntax
to mark when an expression should be treated as a type expression
(e.g., preceding a type expression with a colon).
Those approaches do not appeal to me, as they increase complexity for everyone.
Instead, I would like to examine whether it is possible to
design a converged syntactic grammar that gracefully supports both
type and value expressions in as synergistic and simple way.


## Unification for Cone ##

To unify these grammars, let's examine each kind of type expression, one after another,
and propose how to resolve any grammar conflicts with value expressions
that employ similar syntax.

### Nominal Types ###

Let's begin with the various nominal types: struct, tuple, enum, number types, etc.
As already mentioned, unification here is quite straightforward,
as nominal types cleanly distinguish between the name-based definition of the type
and its use. The **definition** of a nominal type is a statement.
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
and then distinguish which it has later, during semantic analysis.

Further solidifying their isomorphism, is that both type tuples
and value types appreciate being wrapped in parentheses when they
need to preserve their coherence after being thrown into a sea of commas
that serve a different master.

Would that they all would be this easy!

### References ###

References in Cone are fortunately another straightforward win.
Here is the syntax for a reference type:

    ref-type ::= '&' ('<' | [])? region-type? perm-type? lifetime? type
	
Let's walk through these elements which are unique to Cone:

- The ampersand signifies a reference type expression. There are three kinds
  of reference types:  regular ones ('&'), virtual references ('&<') and
  array references ('&[]'). The latter two make use of fat pointers that
  include a pointer to the vtable or the number of elements in the slice.
  
- The region type specifies the memory management region responsible for
  allocating the object pointed-to by this object (e.g., `so` for single-owner).
  If a region is not specified, this corresponds to a borrowed reference.
 
- The permission constrains how a reference may be used, such as indicating
  whether the object it points to may be read or mutated. Specifying this
  is also optional, and a context-sensitive default is assumed when omitted.
  
- The lifetime is an annotation (e.g., `'a`) that can be placed on any reference
  (though typically a borrowed reference). It helps ensure memory safety.
  
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

In Cone, a closure is sugar for a struct that both holds the state of the closure
and implements the callable method named `()`.
Since a closure holds state, one certainly can move the value piece of it around locally,
particularly when we can take advantage of type inference.
However, given the ackwardness of explicitly specifying the full type
for a closure, a value-based closure usually stays local.

When one does wish to move a closure around to other functions,
this is best done with a virtual reference, built around an ad hoc
trait that only captures the signature of the function.
This allows any closure whose function signature matches, regardless
of the structure of the state, to type match.

The type expression for defining a virtual reference to a closure looks like
a reference to a function:

    &<fn (i32, i32) i32

### Lifetime-constrained type ###

It is not just references which may need lifetime constraints.
They can also be valuable for constraining struct-based values which
contain references and even can be applied to certain number types!
To accomplish this, Rust uses the `+` operator to apply a lifetime
annotation to a type: `Context + 'a`

Cone's type expression syntax allows the use of a lifetime annotation
as effectively a prefix operator: `'a Context`

    life-type ::= lifetime type

Since lifetimes are not used to calculate values, 
there is no comparable grammatic structure in value expressions.
	
### Pointer ###

Pointer type expressions are simple, especially compared to references:

    ptr-type ::= '*' type

From a C heritage standpoint, the syntax of the
type expression differs from that of a type constructor for a pointer.
C uses `&` to construct a pointer. Cone uses it to construct a reference.
Cone provides no constructor for a pointer, per se.
Instead, a pointer is created by coercing a reference to a pointer.

That said, value expressions do use `*` as a prefix operator for
dereferencing a reference or pointer. Fortunately, even if the semantics
are not perfectly aligned when using the `*` as a prefix operator
for types vs. values, the grammar here is basically the same.

### Optional type ###

Cone provides convenient sugar for optional types. Instead of having to
specify `Option[&i32]`, it is possible to abbreviate this to `?&i32`.
This means that type expression grammar includes this rule:

    option-type ::= '?' type
	
Value expressions do not support use of the question mark as a prefix operator.

### Array ###

I have saved the most troublesome challenge for last.
My preference is to declare arrays with the element type following
the size contained in brackets: `[4] i32`.

However, this grammar clashes with the syntax that value expressions
use for array literals, which also begin with the '[' bracket.
Array literals do not permit an expression to follow the closing `]` bracket.

The most convenient way to reconcile these grammars is to adopt something
similar to Rust's syntax, which moves both the size and element type
within the square brackets, delimited by a semi-colon.

    array-type ::= '[' exp (',' exp)* (';' exp)? ']'
	
For a simple array type, it might look like this: `[4; i32]`, where the size comes first.
For a multi-dimensional array, it would look like this: `[3,3; f32]`.
This is a lot cleaner than we needed to use C's approach to multi-dimensional arrays:
`[3; [3; f32]]`!

This dual-partition array syntax also brings added value to array literals, 
offering support for three
complementary approaches to constructing a fixed-size array:

- `[1, 4, 9, 16]` which creates a four-element array initialized with those four values.
- `[6; 0]` which creates a six-element array with every element initialized to 0.
- `[100; closure]` which creates a large array and uses the closure to initialize
  each element.

## Implementation Strategies ##

In summary, it turns to be reasonably straightforward to merge Cone's
syntax for value and type expressions.
Sometimes, this will mean grammar rules that only apply to types (e.g., `?`)
and others that only apply to value expressions (so many, including nearly all infix operators).
These differences can be ignored during parsing, and then flagged as errors
during semantic analysis, when we know which is expected.

Another fortunate development is that the two grammars do not
differ in terms of operator precedence.

To change the compiler to support the fusion of expression grammars,
changes are needed to several passes:

- Cone's parser is recursive descent, with separate parsing functions
  for each value and type expression grammar rule. These need to be consolidated together
  so that any expression can be parsed, regardless of whether type or value.
  They also need to produce AST nodes that are driven by the operator parsed,
  and are agnostic about the kind of expression it is building.
  For example, instead of a separate DerefNode and PtrNode,
  parsing of a `*` prefix operator needs to produce a generic "star" node,
  which can later be specialized to either a DerefNode or PtrNode.
  
- The name resolution pass is when we determine whether a node is part of a type
  vs. value expression. Since any name resolves to its declaration,
  this is all we need to know whether the name is for a type vs. a variable.
  Similarly, a StarNode can be specialized to a PtrNode if the inner expression
  is a type, otherwise it specializes to a DerefNode.
  
- The type check pass is when we can verify whether any specific expression
  node is expected to be a type vs. value node, based on the context of where
  it has been placed. If it violates the expectation, a compile-time
  error can be produced.
  
This is not a trivial refactor, but it is not an overly complicated one either.
Ultimately, this grammatic flexibility opens the door to richer capabilities
down the road, including (god forbid) dependent types!