---
title: Composable Operations
layout: post
---

Last week, we took a peek at the <a href="http://www.headspring.com/patrick/whittling-parsley/">low-level pattern recognition classes</a> that make up the core of Parsley.  The final example showed how you can walk through some untrustworthy text in small and deliberate steps, looking for expected input along the way.  At each step, the step either succeeds or fails.  On success, the remaining unparsed text is passed along to the next step.

This process was the epitome of imperative coding: *Do this, and then do this, and then do that.*  Despite this imperative core, I'd describe the intended usage of Parsley as declarative: *Here's what valid input looks like, figure out the details for me.*

I said I'd reveal this week how this shift from low-level imperative coding to high-level declarative coding can be implemented, but I've decided to split it into two posts.  This week, we'll see how Parsley builds on the frustrating foundation by baking in a few fundamental and composable imperative operations.  Next week, we'll see how that more-useful layer can be extended into a high-level <a href="http://martinfowler.com/bliki/DomainSpecificLanguage.html">Domain Specific Language</a> (DSL).

**A declarative API needs to give your end users a suite of useful and familiar words, in such a way that the words can be easily combined to form larger declarations.**  We're going to take some important imperative operations, give them useful names, and package them up in such a way that they will compose well with each other.

## Pattern Recognition Fundamentals

Parsley can recognize patterns not unlike those recognized by regexes.  When working with regexes, we're constantly composing three fundamental operations: repetition, choice, and sequence.

With **repetition**, we say, "I've got this regex here, and I want it to be repeated."  The regex "A" recognizes a single letter "A", and we can produce a more interesting regex "A*" to recognize zero-or-more As, or "A+" to recognize one-or-more As.  A more complex pattern like "(A+B+)" can likewise be made more interesting via repetition: "(A+B+)*" which recognizes zero-or-more occurrences of As-followed-by-Bs.

With **choice**, we can combine some small patterns into a larger one by saying, "Any one of these subpatterns could happen next."  The regex "A|B" recognizes either an "A" or a "B".

With **sequence**, we can combine some small patterns into a larger one by saying, "The first pattern, followed by the second, followed by the third."  Regexes achieve this by just squishing them together in order: the regex "ABC" recognizes an A, followed by a B, followed by a C.  They all have to appear, and in that order.

These three operations *compose* very well.  The regex @"[_a-zA-Z]+[_a-zA-Z0-9]*" recognizes variable names.  We have repetition: the pattern has one-or-more leading characters and zero-or-more trailing characters.  We have choice: each character is an underscore or a letter or a digit.  We have sequence: to enforce that names don't start with a digit, we have a two-part sequence of leading characters *followed by* trailing characters.  We say these operations "compose well" because:
<ol>
<li>They can be applied to any pattern.</li>
<li>Doing so takes very little code.</li>
<li>Most importantly, **the result is also a pattern to which the operations can be applied again.**</li>
</ol>

If our API can exhibit these three properties, we can say it is "highly composable", which should go a long way to making the API declarative as well.  Instead of telling the machine what to do, step by step, we'll just glue together a bunch of small words into a larger statement.  The small words and the glue operations we perform on them will hide all the imperative details.

## What Hurt Last Week

The big example at the end of the <a href="http://www.headspring.com/patrick/whittling-parsley/">last post</a> just had to tell the user whether or not a string was a single letter surrounded by parentheses.  Despite being a simple pattern, the code involved was quite large and annoying to write.

Recall the Parser&lt;T&gt; interface:

{% gist 4727988 %}

A Parser of Things consumes some input tokens with the intent to construct a Thing.  On success, the Parser of Things produces a Thing and a reference to the remaining unconsumed tokens.  On failure, you get an error instead of a Thing.

To write the Parser of Parenthesized Letters, we first needed to write an implementation of this interface, just so that we could say "I expect the next token to be of a certain kind, would you kindly try to consume one such token?"  We then made a convenience method to construct instances of this class, to save on keystrokes later.  Finally, we could write the big, ugly Parser of Parenthesized Letters.

It hurts to have to write so much for so little gain.  I'd rather Parsley contain a suite of useful Parser&lt;T&gt; implementations out of the box, so I could just dive in and focus on detecting the pattern I'm interested in.  This suite of useful Parser&lt;T&gt; implementations ought to include our 3 fundamental/composable pattern-recognition operations: repetition, choice, and sequence.

First, though, the suite should provide some primitive operations to get started with.  Compared to our regex examples, we need to be able to express things like "A" before we can augment them into interesting patterns like "A*".

## Parsley's Default Parsers

The default parser implementations are as frustrating to write as the sample from last week, since they are written in terms of the awkward low-level API.  However, they make things easier on the end user, as we'll see next week.  They are found under the <a href="https://github.com/plioi/parsley/tree/cb69098da8135f7ac5fb1b0f84071e0e8b94b8a0/src/Parsley/Primitives">Primitives</a> folder.  Four of these are relevant to today's discussion:

**TokenByKindParser** - Last week we wrote a parser that takes an expected token *kind* and demands that the next token in the input be of that kind.  On success, the token is consumed.  For instance, if you are parsing C# you may reach a point where you expect the next token to be an identifier.  You don't care what specific identifier it is, only that it is an identifier as opposed to an operator or number.  The real implementation of that class in Parsley is called TokenByKindParser:

{% gist 4728010 %}

**TokenByLiteralParser** - Sometimes you have a more specific expectation of what will appear next in the input.  Rather than only caring about what kind of token appears next, you'll expect that the token will be a specific known string.  When parsing C#, you may reach a point where you know the next token must be a ";" character.  TokenByLiteralParser serves this purpose.  As with TokenByKindParser, we simply inspect the current token and return success or failure, advancing one token on success:

{% gist 4728017 %}

**ZeroOrMoreParser** - This class gives us our much-needed repetition operation.  Note that this class is constructed *with another Parser&lt;T&gt;* to be augmented, just as the regex "A" augmented by "*" becomes the regex "A*".  The parser works by attempting the given parser against the input.  On success, we repeat from the new position in the input, collecting each intermediate result into a list.  As soon as the item-parser finally fails, we are finished and can return a successful result containing all the items collected along the way.  Note that this is a Parser&lt;IEnumerable&lt;T&gt;&gt;, meaning that after consuming a bunch of tokens, the result you get is a collection of things, where each thing was recognized by the initial input Parser&lt;T&gt;.  That's our first hint at composabilty:

{% gist 4728052 %}

**ChoiceParser** - This beast gives us our much-needed choice operation.  When we want the next thing in the input to be one of several possible things, and each of those things is individually recognized by a different Parser class, this combines them into one operation that says "Any of these things could appear next."  The first one to match the input wins:

{% gist 4728061 %}

ZeroOrMoreParser and ChoiceParser make up a part of the composability we are shooting for here.  They take in Parsers as their input and are themselves Parsers.  That means you could feed one into the other: you might have two ZeroOrMoreParser instances, and pass them into a ChoiceParser: I expect zero-or-more As or else I expect zero-or-more Bs.

There's a subtle gotcha within both ZeroOrMoreParser and ChoiceParser.  By default, Parsley distinguishes between a parser that fails **without** advancing in the input and a parser that fails **after** advancing in the input.  If you are parsing C# and you know that the next thing in the input is a choice between a foreach statement or an if statement, but the input is "if $typo$", we don't just want to say that the choice failed; we want to say that your if statement is broken.  This is why these two classes care so much about detecting changes in position along the way.

## What About Sequence?

TokenByKindParser and TokenByLiteralParser give us our trivial starting point, like a regex that only recognizes a single letter "A".  ZeroOrMoreParser gives us repetition.  ChoiceParser gives us choice.  We still don't have a useful sequence operation to build on.  Our complicated example from last week is still complicated!

Recall my criticism from last week:

<blockquote>[IsParenthesizedLetter] is nearly as tedious as writing assembly. Each time I wanted to progress a little further, I had to indent again and declare some new local variables. I had to carefully pass along the remaining unparsed tokens at each step, and I had to concern myself with failure at each step.

To make matters worse, this stuff motivates having lots of returns from a single function, making it extremely hard to break the function apart into smaller parts. <a href="http://www.headspring.com/patrick/detect-reflect-decomplect/">Return statements are "Extract Method" fences.</a></blockquote>

With all the code we've seen so far, we're stuck with implementing all sequence patterns in this way, producing a convoluted morass of nested if statements.  We don't even have the option of Extract Method refactorings to ease the pain.  We've coded ourselves into a corner!

Fortunately, a little-used design pattern will save the day next week, allowing us to bottle up the subtle code duplication exhibited by last week's manual attempt at a sequence operation.  Once we have sequencing down, all of these peices will start to fit together smoothly, and our end user will finally have that suite of useful composable words at their disposal.

## Recap

Last week, we saw how Parsley represents progress through the input, and caught a glimpse of how a desired pattern can be recognized (or not) within that input.  The code to use those classes was verbose, ugly, and apparently refactor-proof.

To ease the pain, we've seen how a few composable operations could be provided on top of the original core classes, but we're still left with the tricky situation of sequencing.  Next week, we'll implement sequencing, completing our DSL and insulating the end user from having to work with the ugly core directly.