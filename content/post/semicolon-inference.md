---
title: "Semicolon Inference"
date: 2020-04-03T10:26:29+10:00
draft: false
# thumbnail: "img/placeholder.jpg" # Optional, thumbnail
disable_comments: true # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: false # Optional, enable MathJax for specific post
categories:
  - "Syntax"
tags:
  - "Lua"
  - "Kotlin"
  - "Javascript"
  - "Swift"
  - "Go"
  - "Scala"
  - "Python"
  - "Cone"
---

Many programming languages require that statements end with a semicolon.
Some languages, 
such as Javascript, Go, Kotlin, Swift, Scala and Lua,
make this requirement optional.
Although a semicolon may be specified, 
it is not required to terminate a statement.

This post explores the underlying rules various languages use to
make semicolon inference possible.
It also articulates the arguments for and against a language offering support for it.<sup>1</sup>

## The Challenge ##

The semicolon inference rules would be simple if every statement fit on a single line.
If that were the case, the lexer could just automatically 
inject a semi-colon token at the end of each statement-containing line.
However, long statements sometimes need to span multiple lines.

One simplistic solution would use a special punctuation to mark
when multiple lines form a single statement.
This continuation marker (e.g., a backslash) could appear at the end of
each unfinished line, or the beginning of each continuation line:

    printf("Solution results: %s %s %s\n",
	\ length, width, height)    // infer semicolon after ')'

This approach preserves rule simplicity, but it has drawbacks:
it is error-prone, as the coder can easily forget to type the continuation character,
and it visually breaks up reader's understanding of the statement's logic.

Language designers often prefer is to make semicolon inference
more intelligent, so that it can reasonably and automatically detect when a statement
runs across multiple lines.
One useful approach involves detecting when a statement is syntactically "unfinished".
In such cases, we can delay the semicolon until we achieve syntactic completion.
For example:

- A line ending with an operator that requires a follow-on term:

        int foo = a +     // No semicolon, as '+' operator requires a follow-on term
          2

- The line has unclosed parentheses, brackets or quotation marks:

        printf("Solution results: %s %s %s\n",  // Wait until parentheses close
	      length, width, height)
    
Unfortunately, sometimes this is ambiguous. Sometimes a statement is syntactically
complete at the end of the line, even though the coder has split it across multiple lines:

    int foo = a
	  - 2

Should this be eagerly treated as a single statement (`int foo = a-2;`) 
or conservatively as two (`int foo = a; -2;`)?
The eager approach can lead to a style guide recommending that certain statements
proactively begin with a semicolon to avoid it being accidentally merged into the
previous line's statements.
Furthermore, either approach bears the risk of a silent failure, where the compiler
may interpret it differently than the coder, but never flags the ambiguity as potentially problematic.

Let's examine how various languages deal with these challenges.

## Python ##

Technically, Python does not have semi-colon inference, but it does have
end-of-statement inference, which is close enough.

In most cases, the end-of-line is also the end of the current statement.
Multi-line statement are, however, possible:

- The backslash character may be used at the end of a line to indicate
  the statement continues on to the next line.
- If a statement has parentheses, brackets or braces that have not yet been closed,
  the statement continues on to the next line.

Semi-colons may be specified to separate multiple statements on the same line.
Intriguingly, semicolon has a lower precedence than other operators (e.g., `:`),
useful when one requires multiple statements on the same line as a control statement.
This performs both prints on each `i`:

    for i in range(10): print "foo"; print "bar"

A semi-colon found at the end of a line is allowed, but it serves no useful purpose
and is discouraged.

## Javascript ##

Javascript uses a [parser-based approach](http://www.ecma-international.org/ecma-262/7.0/index.html#sec-rules-of-automatic-semicolon-insertion).
The lengthy spec lists several rules and exceptions.
The heart of it says that whenever a token (called the offending token) is encountered
that is not allowed by any production of the grammar, 
then a semicolon is automatically inserted before the offending token if:

- The offending token is separated from the previous token by at least one LineTerminator.
- The offending token is }.
- The previous token is ) and the inserted semicolon would then be parsed 
  as the terminating semicolon of a do-while statement.
  
These rules are complex and error-prone. 
On first blush, the rules look eager, but then there are many exceptions
built around restricted grammar production rules that bring a conservative flavor.
The above link offers
several examples where semicolons are injected where one might not expect them,
due to these restricted grammar production rules.
This complexity is further highlighted by the last guideline, indicating that whenever
"an assignment statement must begin with a left parenthesis, 
it is a good idea for the programmer to provide an explicit semicolon 
at the end of the preceding statement rather than to rely on automatic semicolon insertion."

The confusion surrounding these complex rules divides style guide stances into different schools.
Some advocate explicit use of semicolons nearly always.
Others allow implicit semicolons, but suggest code style guidelines
that minimize the chance of incorrect semicolon inference.
Google "javascript semicolon rules" for examples of both.

## Lua ##

Lua's approach is largely eager. There is no spec that clearly describes
the rules. The reference manual offers only a brief mention.
However, a helpful insight can be found in this
[post](https://www.seventeencups.net/posts/how-lua-banished-the-semicolons/).

Lua's approach appears to depend on the statement grammar being more limited,
not allowing expressions-as-statements (except function calls).
Thus, when the parser fully hits the end of a statement, it knows it.
Then it can look for an optional semicolon.
Failing to find one just begins the next statement.
Unexpectedly, this need not happen at the end of a line!

Being so eager, even the reference manual warns against the parser
consuming too much. To help prevent against this, it suggests that any new statement which begins
with a parentheses should be preceded with a semicolon:

    a = b + c
    ;(print or io.write)('done')

## Go ##

Go's lexer uses a [simple conservative ruleset](https://medium.com/golangspec/automatic-semicolon-insertion-in-go-1990338f2649).
It automatically injects a semicolon token when the line ends with one of the following tokens:

- an identifier
- an integer, floating-point, imaginary, rune, or string literal
- one of the keywords break, continue, fallthrough, or return
- one of the operators and delimiters ++, --, ), ], or }

This approach would presumably handle the trailing operator scenario well,
since unfinished operator tokens (e.g., `+`) do not trigger a semicolon.
It would also handle any unfinished parentheses or brackets, so long as the
coder was disciplined enough to end lines with a separator delimiter,
such as a comma. 

But this approach can fail when there is no such separator delimiter:

    placeValue(row, column,   // No semicolon here, due to trailing comma
      calculateTheAverage(oldValue, newValue)  // Oops! Semicolon injected here
    )	  

	
## Scala ##

Scala uses a somewhat different lexer 
[ruleset](http://jittakal.blogspot.com/2012/07/scala-rules-of-semicolon-inference.html).
A line ending is treated as a semicolon, except when:

1. The line in question ends in a word that would not be legal as the end of a statement, such as a period or an infix operator.
2. The next line begins with a word that cannot start a statement.
3. The line ends while inside parentheses (…) or brackets […], because these cannot contain multiple statements anyway.

The first rule operates similarly to Go's rules, allowing continuation when the statement is unfinished.
The second rule opens up continuation when the next line could not be a statement on its own.
So, it handles when an infix operator either ends the previous line or starts the next line,
in many cases. However, if an infix operator is also a valid prefix operator, it will break:

    let list2 = list1           // Oops! semicolon injected here
      |> myListFunction         // ... and here
      |> myOtherListFunction

Ahnfelt has proposed a more eager [variation](https://jsfiddle.net/ahnfelt/mdjp4bto/) 
of these rules that addresses this example.
The revised rule states: If two consecutive tokens a and b are on different lines, 
and a is in the beforeSemicolon category, 
and b is in the afterSemicolon category, insert a semicolon.
This would inject a semicolon, for example, if both tokens were identifiers,
or if one line ended with `)` and the next started with `(`.

As for Scala's third rule, it offers additional versatility, as it recognizes
that a statement should consume as many lines as needed
until all its open parentheses or brackets have been closed (like Python).
This protects against the problem shown in the Go example earlier.

## Swift ##

Swift's approach is eager, but in a less complicated way than Javascript.
It is very grammar-driven: statements that end with a semi-colon make that semi-colon optional.
Furthermore, if you want to start a new statement on the same line that ended
a previous statement, you must put a semi-colon in between them.
In this way, optional semi-colons are effectively only ever injected
at the end of a line.

Like any eager approach (e.g., Lua), the downside is that a statement may consume more lines
than intended. If a follow-on line is a grammatically correct continuation of the
statement begun on previous lines, it will be consumed as part of the statement.
Avoiding this silent failure involves either not creating statements that
could be a continuation, or else explicitly specifying the semi-colon to avoid it.

Unlike in some of the other approaches, the lexer plays a minor role
by providing a way to detect whether it is currently at the "start" of a new line.
The rest of it is handled by the parser.

## Kotlin ##

Kotlin's approach is similar to Swift's, being eager and 
[grammar-driven](https://github.com/antlr/grammars-v4/blob/master/kotlin/kotlin/KotlinParser.g4).
It has a specific "anysemi" production rule that expects to
find either a [new line or a semi-colon](https://discuss.kotlinlang.org/t/what-design-principles-contribute-to-the-effectiveness-of-kotlins-asi/15281/2)
as a way to separate statements.
Here is some additional [insight](https://www.reddit.com/r/Kotlin/comments/e2jr1z/how_does_kotlins_automatic_semicolon_insertion/).

As best as I can determine, it offers the same advantages and disadvantages as
Swift's approach from a coder point-of-view.
Kotlin recommends against the use of semicolons (and even offers a warning when they are redundant),
except under [two conditions](https://stackoverflow.com/questions/39318457/what-are-the-rules-of-semicolon-inference).

However, the grammar is more complex to handle it this way correctly, as the new-line
is a token the grammar must explicitly deal with.

## Tooling ##

Semicolon inference adds complexity to a language's tooling.
Any tool requiring awareness of the language's grammar,
such as editors or linters, has to also correcty comply with the language's semicolon rules.
In most cases, this means they cannot simply rely on straightforward context-free parsing.
Real-time edits will have to be responsive to the current context,
as well as any edit changes to that context.

One can imagine two possible proactive roles that editors could play
in facilitating semicolon inference:

- Have the editor automatically insert semicolons where appropriate,
  instead of providing semicolon inference as part of the language itself.
  
- Offer an option for the editor to display inferred semicolons
  wherever the language understands them to be located.
  This way the programmer is able to get a visual indication showing
  where the compiler understands the optional semicolons to be placed.
  This would be useful during editing to notice when a statement has absorbed more (or fewer)
  lines than the coder intends.

## What about Cone? ##

I have done this bit of research, so that I can make an informed choice
about whether to support semicolon inference as part of
[Cone](http://cone.jondgoodwin.com/).

I have long wanted to, but certainly not for any overwhelmingly essential reason.
It see it as a nice piece of sugar for the programmer, offering two benefits:

- The code is slightly less cluttered with punctuation, making it marginally more readable
- The compiler will annoy the programmer fewer times because of the
  inadvertent absence of a required semicolon.

The latter is the more compelling reason for me, 
but it only carries the day when:

- the rules are simple for both the compiler and coder, and
- there is little risk of the semicolon being inferred in the wrong place

Of the conservative approaches, I prefer Scala's as being the most accurate,
but it still troubles me that some leading infix operators will not be
viewed as continuations. And the rules border on complicated for people and
the compiler, and would get even more complicated to handle
Cone's support for blocks-as-expressions.
The lexer would need a lexing stack to handle statements inside blocks,
which are themselves inside expressions that use parentheses or brackets. Phew!

Swift's eager approach, then, seems the most compelling for Cone.
The fact that Swift's approach hiccups rarely for their programmers is encouraging.
However, it is likely to hiccup more often with Cone because there
are a number of operators that offer both prefix and infix semantics.
Is a line that starts with `*`, `-`, `&`, `.`, `?.`, `<-`, `++`, `--`, `(.` or `[`
a continuation of the previous line (as an infix operator) or the start
of a new line (using a prefix operator)? If Cone adopts Swift's approach,
the former is selected, but that will reach the wrong conclusion in many cases.

As Matthieum suggests, a better way to resolve this ambiguity is to leverage the common style of 
slightly indenting continuation lines after the first line.
This style convention style offers a valuable visual signal to the human reader to indicate
that a statement spans multiple lines.
The Cone compiler can exploit the same convention to decide
whether a new line is a continuation or a new statement: 
any line beginning with an ambiguous
operator is considered a continuation (infix) if it is indented, otherwise it is
treated as a new statement (prefix). When there is no ambiguity,
the grammar determines whether to consume additional lines, with or without additional indentation.
This allows `else` to have the same indentation
as its `if` line and still be considered a continuation of the `if` statement.

I like this modified version of Swift's optional semi-colon rules very much, as it:

- conforms to (and leverages) best-practice style guidelines,
- is easily explained to programmers (indent continuation lines!),
- is relatively straightforward to implement in the compiler lexer and parser,
- is (probably) less error-prone than requiring programmers to specify semi-colons
  at the end of every statement

So, I think that's what I will try, and see how it works out.


----

<sup>1</sup> Another post that covers similar material is
[here](https://temperlang.dev/design-sketches/parsing-program-structure.html#asi).
This post also tackles a number of other syntactic challenges for C-like languages,
beyond the issue of semicolon inference.

