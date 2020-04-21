---
title: "Significant Indentation"
date: 2020-04-18T17:42:43+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Syntax"
tags:
  - "Cone"
---

The last post focused narrowly on evaluating various techniques for
[semi-colon inference](/post/semicolon-inference/),
thereby enabling a program to successfully compile even when the programmer 
omits statement-ending semicolons.
This post broadens this theme, looding at how the compiler can take
advantage of significant indentation when handling block inference and multi-line string literals.

I wander into this topic knowing that for many programmers,
compiler support for significant indentation is troubling and uncomfortable.
They prefer the explicit clarity of required semi-colons to end statements and
curly braces to delimit blocks. This keeps the grammar straightforward.
The parser is easily written. 
This black-and-white approach leaves no room for grammatic ambiguity;
the compiler will not interpret tokens differently than the programmer intended.
Advocates for explicit token markers do not understand why anyone would risk such errors
just for the sake of sparing the programmer a few keystrokes.

I have no wish to persuade anyone to embrace significant indentation,
if they are confidently against it.
Realistically, I know I am in the minority for wanting
to leverage significant identation.
This awareness is why it has long been my intent to ensure that the Cone
compiler gracefully supports programs written in the free format style,
where semi-colons are always explicitly specified at the end of statements,
and curly braces are explicitly used to delimit blocks. And so it does.

If this is what you prefer, Cone will follow your lead.
So, unless you are curious to explore another perspective, 
there is no need for you to read the rest of this post.

## The Pros and Cons ##

These are the reasons why I find significant indentation appealing:

- **It allows more of a program's logic to be visible in an editor**.
  The editors I use typically fit only two or three dozen lines on the screen at a time.
  Common layout standards waste a whole line for the closing curly brace of each block.
  Some format standards waste yet another line for each opening curly brace.
  When blocks are small, these wasted lines add up quickly.
  Ultimately, this can constrain how much of the logic I can examine without scrolling.
  I don't like that, particularly with this common idiom:
  
        if a > 0         vs.      if a > 0:
	    {                            break;
	        break;
	    }
  
- **It improves the likelihood that a user's program will compile successfully**.
  People are rarely as disciplined as the computer. They make mistakes,
  including carelessly forgetting required semi-colons or curly braces.
  Most compilers won't let you go forward until you correct these mistakes,
  which can get annoying, especially when the compiler
  really can figure it out with a little more intelligence.
  I prefer it when compilers are less demanding and more helpfully friendly.

- **Logic that omits semi-colons and semi-colons is less visually noisy**.
  Program logic feels easier to scan when the screen displays 
  fewer symbols that need to be decoded.

Notice that I have said nothing about saving the programmer some keystrokes.
It does, but such a consideration feels comparatively insignificant to me,
as the effort to key these symbols is really not that great.

Significant indentation takes advantage of the fact that it is generally
considered best practices to indent block statements and subsequent
lines of a multi-line statement.
This approach makes code far easier to read for people.
We even build linters to make sure we do it correctly.
Why not allow the compiler to take advantage of the same visual
layout to make the same sort of rapid deductions about where
statements and blocks begin and end?

The biggest danger with this approach is the risk of ambiguity,
where the compiler comes to a different conclusion than the programmer intended.
Whatever approach is taken needs to significantly minimize this risk,
as such misunderstandings can be hard to notice.

For a few years, I have been seeking an approach that
satisfies seemingly contradictory goals: 

- it delivers on the benefits mentioned earlier,
- it eliminates the risk of confusing ambiguity, and 
- it supports programmers that prefer explicit delimeters, as well as
  those that allow delimeters to be inferred from significant whitespace
  (as Haskell also does).

## How to do it the Wrong Way ##

A few years ago, I came up with what seemed to be a promising approach.
I implemented it in two languages, first Acorn and then Cone.
It worked out well for Acorn, but the more I played with it in Cone,
the less I liked it. It turned out to be error-prone in frustrating ways,
and the rules for switching between modes were not simple and obvious.
In short, it kept getting in the way of a smooth programming experience.

There is no reason for me to go into great detail about where
it went astray, other than to point out where the conceptual flaw lay.
The mistake I made was to try to embed the intelligence of the
technique completely into the lexer. The lexer used some simplistic indentation
heuristics to determine when to automatically inject semicolons and
curly braces into the parser, as if those tokens had been explicitly specified.
If it discovered the explicit curly braces, it would turn off
significant indentation, thereby switching modes from significant indentation
to free form.

I liked this approach because the parser only sees and follows
the formal grammar, ignoring all white space discarded by the lexer.

This approach went wrong in too many disturbing ways.
A multi-line statement might be artificially divided into
several statements, one per line. Some statements do not
end with a semi-colon, but get one anyway.
Sometimes you want to put a block inside a statement,
and then have it carry on afterwards.

Most of the time the algorithm worked as intended, and I had decent workarounds
for some of the problems, but I knew in my heart it just was
not going to cut it. For lack of a better approach, I put it on
hold, but then I was left reluctant to write or show much code in Cone,
because I knew I was not settled on this basic syntactic convention.

## Grammar-driven Approach ##

It turns out that the better way to do this is to have 
significant indentation rules
be driven by the parser and the grammar it follows.
The parser can then asks the lexer for help, as needed.
By using the language's grammar as a guide,
the parser only needs to consider significant indentation
in certain very narrow circumstances, where there might be ambiguity
or where the grammar asks for it.

Let's take a look at each of these situations in turn.

### Blocks Without Curly Braces ###

As with many other languages, curly braces `{}` are widely usable in Cone
to mark some collection of statements as belonging together.
They are used in a syntactically consistent way for logic blocks, 
type definitions, modules, and more. They facilitate structured modularity.
If you are a fan of explicitly using curly-braces to delimit and wrap
some collection of statements, the Cone compiler will serve you well.

But Cone also supports an alternative syntactic style for blocks.
Similar to Python, one may begin a block with a colon and then
have all the block's statements indented on subsequent lines.
Wherever the added indentation ends, so does the block.

This example shows (again) both options:
  
        if a > 0         vs.      if a > 0:
	    {                            break;
	        break;                b = 5;
	    }
		b = 5;
  
When curly braces are used, indentation is basically ignored.
Indentation only becomes "significant" when the programmer explicitly asks
for it to be used to signal the end of the block.

### Statements without Semi-colons ###

As the prior post summarized, 
statement-ending semi-colons may be explicitly specified.
However, whenever the statement ends at the end of the line,
the semi-colon may be omitted.

The only place that this could potentially go wrong is
when it is potentially ambiguous, based on valid grammar.
Is this one statement or two?

        a = b + c + d
        - e + f

This confusion can only crop up when possible continuation lines
begin with an operator that *could* be a prefix operator or
it *could* be an infix/postfix operator, as is true for `-`.
Even this confusion is avoided if prior lines have open, but
not yet closed, parentheses, square brackets or curly braces.

How then does the programmer mark which interpretation the
compiler should make? By following a simple rule,
continuation lines should be indented after the first line of
a multiline statement, like so:

        a = b + c + d
          - e + f

The latter example is understood to be one multi-line statement.
The earlier example (without indentation) is understood to be
two separate statements.

Note that this rule is no more risky than requiring semi-colons to terminate
every statement. The absence of a desired semi-colon at the end of the first line
creates the same unfortunate compiler error as neglecting to indent continuation lines.

### Multi-line String Literals ###

There is only one other place that indentation plays an interesting
role in the Cone compiler: for multi-line string literals.

I think it is visually better when the lines of a long string can be indented
in harmony with the surrounding lines that contain the string.
Cone makes this possible by discarding indentation at the start of every line
within a string:

        print("
           This is a line.\n
           Another line.\n")

This is equivalent to:

        print("This is a line.\nAnother line.\n")

If you want any spaces at the start of a line to be included in the string,
precede the first of them with the backslash escape character.

## Summary Assessment ##

I scrapped the earlier algorithm for significant indentation,
replacing it with this new approach. It works exactly the way I want it to, finally.
I updated all code examples in the playground and reference documentation,
and added a few examples to explain these simple rules and guidelines.

I really am quite happy with how it is working out so far:

- It accomplishes my primary goals, in allowing vertical space to be used
  more efficiently and tolerating errors when semi-colons are omitted unintentionally.
- It gracefully supports both free form explicit annotations and significant
  indentation-based approaches without the need for any extra annotations.
- It only rarely cares about indentation: when explicitly requested for blocks
  and to disambiguate multi-line statements. And when it does,
  it does so in alignment with indentation best practices for people, 
  documented in many style guides and enforced by many linters.
- The rules are simple and easy to explain to any programmer.
- It is no more susceptible to compiler errors than explicit use of
  semicolons and curly braces.
- The parser remains straightforward and largely grammar-driven,
  based largely on lexer-produced tokens which discard white space.
  Only in a few places does the parser ask the lexer to help
  make syntactic decisions based on significant indentation.
  
On to the next challenge...