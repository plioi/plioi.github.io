---
title: "Recognizing Covariance Issues in Your Code"
layout: post
---


My last three posts have covered the nuts and bolts of the little-used `in` and `out` keywords:

* <a href="http://patrick.lioi.net/2011/10/21/array-covariance-in-c/">Array Covariance in C#</a>
* <a href="http://patrick.lioi.net/2011/10/28/seamless-generic-type-casting/">Seamless Generic Type Casting</a>
* <a href="http://patrick.lioi.net/2011/11/04/seamless-delegate-type-casting/">Seamless Delegate Type Casting</a>


The examples have all been small, and a little contrived, so you may not yet recognize when you need to fetch this tool from your toolbelt. Today, I'll cover a real covariance problem I ran into while developing <a href="http://github.com/plioi/Parsley">Parsley</a>, an open-source text parsing library.

The goal of parsing is to convert plain text into a data structure that can be more easily traversed and reasoned about. You want to describe the patterns of what valid input looks like, as well as describe what objects should be created when those patterns are found.

In today's example, we'll be parsing <a href="http://json.org/">JSON</a> text into the corresponding arrays and dictionaries that JSON describes. Parsley's integration tests include <a href="https://github.com/plioi/parsley/blob/91394b0f17606aea9a583ae49249da3567c53298/src/Parsley.Test/IntegrationTests/Json/JsonGrammar.cs">a JSON parser</a>. **Without the `out` keyword, this sample parser would fail to compile with some pretty suspicious error messages.**

The most important types defined in Parsley are `Parser<T>` and `Reply<T>`:

```cs
public interface Parser<out T>
{
    Reply<T> Parse(Lexer tokens);
}

public interface Reply<out T>
{
    T Value { get; }
    Lexer UnparsedTokens { get; }
    bool Success { get; }
    ErrorMessageList ErrorMessages { get; }
}
```

A `Lexer` (aka tokenizer) inspects the input plain text and converts it to a series of tokens. If we're parsing JSON, tokens would be things like a constant, a keyword, or a puctuation symbol like [ or {. A `Parser` is any object which can consume some tokens and produce a `Reply`. A `Reply` contains the parsed object (if the input was valid), the remaining unconsumed tokens, and possibly some error messages.

Parsley lets you take *small* parsers that do part of the effort, and glue them together into larger parsers that do the full parsing effort. Like regexes, the glue involves repetition ("zero or more of *that* pattern"), alternation ("any one of these possible patterns"), and sequencing ("*this* pattern, followed by *that* pattern").

## Repetition

For repetition, we have helper functions that take in a `Parser<T>` and output a new `Parser<IEnumerable<T>>`. The new parser will repeatedly recognize whether the pattern appears again, consuming it until the pattern is no longer recognized. The overall `Reply` will contain an object for every occurrence of the pattern:

```cs
public static Parser<IEnumerable<T>> ZeroOrMore<T>(Parser<T> item) { ... }
public static Parser<IEnumerable<T>> OneOrMore<T>(Parser<T> item) { ... }

Parser<char> digit = ...;
Parser<IEnumerable<char>> integer = ZeroOrMore(digit);
```

## Alternation

For alternation, the `Choice` helper function takes in several `Parser<T>` and returns a new `Parser<T>` which recognizes when *any one* of those patterns appears next in the input text:

```cs
public static Parser<T> Choice<T>(params Parser<T>[] parsers) { ... }

//The result of a successful parsing of JSON text
//is the result of one of these other simpler patterns,
//whichever one is encountered first:

Json.Rule =
    Choice(True, False, Null, Number, Quotation, Dictionary, Array);
```

## Sequencing

For sequencing, I <a href="http://patrick.lioi.net/2011/09/08/the-compilers-treatment-of-linq-syntax/">redefined the `from` and `select` LINQ keywords</a> to provide the *sequencing* operation, gluing together small `Parser`s into a larger `Parser`.

In part of the JSON parser, we need to recognize the pattern of a single key/value pair for a dictionary. Such a pair is a quoted string, followed by a ':', followed by some other JSON value. The result of the Pair rule is an object containing the relevant parts of that sequence:

```cs
Pair.Rule =
    from key in Quotation
    from colon in Token(":")
    from value in Json
    select new KeyValuePair<string, object>(key, value);
```

The result of a query like this is *yet another* `Parser`. Each `from` clause consumes a little more input, and the `select` clause describes the combined result of those intermediate steps.

## The Problem

At one point, before the `Parser` and `Reply` types used the `out` keyword, I ran into some surprising compiler errors. I *wanted* to write the following:

```cs
GrammarRule<object[]> Array;
GrammarRule<Dictionary<string, object>> Dictionary;

...

Array.Rule =
    from items in Between(Token("["), ZeroOrMore(Json, Token(",")), Token("]"))
    select items.ToArray();
                
Dictionary.Rule =
    from pairs in Between(Token("{"), ZeroOrMore(Pair, Token(",")), Token("}"))
    select ToDictionary(pairs);
```

Just to satisfy the compiler, though, I had to write this instead:

```cs
//Rule declarations had to become less precise:

GrammarRule<object> Array;
GrammarRule<object> Dictionary;

...

//Select clauses needed explicit upcasting:

Array.Rule =
    from items in Between(Token("["), ZeroOrMore(Json, Token(",")), Token("]"))
    select (object) items.ToArray();
                
Dictionary.Rule =
    from pairs in Between(Token("{"), ZeroOrMore(Pair, Token(",")), Token("}"))
    select (object) ToDictionary(pairs);
```

In order to pass `Array` and `Dictionary` to the `Choice` helper method, they needed to have the same type as each other. In order to satisfy the compiler after that, I was having to also include explicit *upcasts* in the `select` clauses. **That is just *weird*!** Have you ever had to explicitly upcast, just to satisfy the compiler? As if an array or dictionary *might not* be an `object`!

## The Solution

> **When the compiler surprises you, the most direct path to getting a successful compile isn't necessarily the right answer. First, try to understand what the compiler is trying to tell you. Ask yourself, "What is it about my code that the compiler is getting tripped up on? Why isn't it making the decision I expected?"**

Taking a closer look, these queries result in values of type `Parser<object[]>` and `Parser<Dictionary<string, object>>`, respectively, and we're trying to assign them each to a `.Rule` property of type `Parser<object>`.

We *want* to treat any `Parser<Child>` as a subtype of any `Parser<Parent>`, whenever `Child` is a subtype of `Parent`. When we look at the `Parser<T>` and `Reply<T>` definitions, we see that the Ts always appear in the return types, never in the input types. The `out` keyword is therefore applicable here, with the desired results:

1. Using `Choice` no longer has the suspicious side effect of requiring less-descriptive types for each rule we happen to pass to it.
2. The compiler no longer needs us to include suspicious explicit upcasts in our select clauses.
