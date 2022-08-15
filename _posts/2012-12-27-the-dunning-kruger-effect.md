---
title: "The Dunning-Kruger Effect"
layout: post
---


This week, I was guilty of the dark side of the <a href="http://en.wikipedia.org/wiki/Dunning%E2%80%93Kruger_effect">Dunning-Kruger Effect</a>.  It's not a universal truth, but it does describe a very real phenomenon that we're all guilty of from time to time.  This happens when an unskilled or incorrect person is overly-confident in their own ability, while highly-competent individuals doubt their own ability.

Good thing it's not a universal truth, or else we'd all get stuck in an infinite loop: "I doubt myself, therefore I must in fact be perfect.  I think I'm perfect, therefore I must doubt myself.  I doubt myself..."  Since that train of thought isn't useful, the practical result of the Dunning-Kruger Effect is that **any extreme confidence should be treated as a sign that we aren't trying hard enough to improve.**  When you *don't* react to your own confidence by seeking out areas for improvement, you doom yourself to stagnate.  That would my worst nightmare come to life (aka the Freddy-Krueger Effect).

This time around, I had false-confidence around a series of refactorings I applied to <a href="https://github.com/plioi/parsley">Parsley</a> about a year ago.

Let's start by setting the scene.  Parsley is a tool for working with complex text patterns, when a single regex is insufficient.  You describe what a successful input looks like by breaking down the overall pattern into several smaller, named patterns called grammar rules.  For instance, if you want to <a href="https://github.com/plioi/parsley/blob/2d113754b6e3e68a43e624987fb7ed5c4dd99870/src/Parsley.Test/IntegrationTests/Json/JsonGrammar.cs">describe JSON documents with Parsley</a>, you could define rules named True, False, Null, Number, Quotation, Array, and Dictionary.  All these grammar rules are described in terms of the *tokens* of JSON, the tiniest substrings that still have meaning to a JSON parser: things like keywords, operators, numeric literals, and string literals.

Some tokens are defined with simple string literals:

```cs
Keyword @null = new Keyword("null");
Keyword @true = new Keyword("true");
Keyword @false = new Keyword("false");
Operator Comma = new Operator(",");
Operator OpenArray = new Operator("[");
Operator CloseArray = new Operator("]");
Operator OpenDictionary = new Operator("{");
Operator CloseDictionary = new Operator("}");
Operator Colon = new Operator(":");
```

Some tokens are defined with simple regexes:

```cs
TokenKind Whitespace = new Pattern("whitespace", @"\s+", skippable: true);
```

Lastly, some tokens are defined with nontrivial, multilined regexes:

```
TokenKind Quotation = new Pattern("string", @"
    # Open quote:
    ""

    # Zero or more content characters:
    (
              [^""\\]*             # Zero or more non-quote, non-slash characters.
        |     \\ [""\\bfnrt\/]     # One of: slash-quote   \\   \b   \f   \n   \r   \t   \/
        |     \\ u [0-9a-fA-F]{4}  # \u folowed by four hex digits
    )*

    # Close quote:
    ""
");
```

An important design goal here was to allow users of Parsley to define their regexes under a technically-untrue assumption: you get to pretend that your regex is being applied to the *start* of a potentially-long string.  You are trying to tell Parsley how to "peel off" the next few characters from the raw input.

If that's how it actually worked, though, we'd be stuck with performing wasteful substring operations: after matching a regex against the front of a million-character string just to peel off the first few characters, you **don't** want to actually perform a nearly-million-character substring operation just to represent progress through the input.  Multiplying that waste by the number of tokens you'd eventually peel off of the original million characters, the waste would add up quickly.  So, I took advantage of some special regex options as well as a "\G" modifier, allowing me to compare your given regex against the input, *starting at whatever input position I feel like.*  I didn't want the consumer to care about these details, so I wrote the TokenRegex class to hide these details:

```cs
public class TokenRegex
{
    private readonly string pattern;
    private readonly Regex regex;

    public TokenRegex(string pattern)
    {
        this.pattern = pattern;
        regex = new Regex(@"\G(
                                   " + pattern + @"
                                   )", RegexOptions.Multiline | RegexOptions.IgnorePatternWhitespace);
    }

    public MatchResult Match(string input, int index)
    {
        var match = regex.Match(input, index);

        if (match.Success)
            return MatchResult.Succeed(match.Value);

        return MatchResult.Fail;
    }

    public override string ToString()
    {
        return pattern;
    }
}
```

Excellent!  You can provide simple regexes intended to match the *start* of a given input, and I modify the pattern so that I can in-fact match at *any* position we've progressed to.  **Man, am I awesome or what?**

Introducing this class *was* a good idea.  It dramatically simplified the regexes that consumers needed to provide, and sufficiently abstracted away the details and reasoning behind this rarely-used "\G" regex modifier.  This refactoring was also a part of a larger refactoring effort, which has also been successful.  I wanted to make it so that you didn't *have* to use all of the moving parts Parsley provides: if you needed a smarter tokenizer, such as one to deal with Python-style indentation, that was ok: I provided useful interfaces to let you hook in your own code for that kind of customization.  TokenRegex was supposed to help you further by isolating all the regex stuff from the rest of the system; if the Pattern class wasn't good enough for your needs, you could provide your own and still leverage TokenRegex while doing so.

This week, fellow-Headspringer <a href="https://coderwall.com/ChrisMissal">Chris Missal</a> made a simple request: "What are your thoughts on giving parsley the ability to match tokens with a case insensitive regex?"

Armed with all the undeserved confidence in the world, I first replied that I already provided the right hooks for exactly that kind of customization: simply see class A, create a similar class B implementing the same interface while using a case-insensitive regex instead.

**Oh. Wait. No. That doesn't actually work.**  Not so awesome, afterall.

Even though the entire effort last year was to provide hooks for *exactly* this sort of customization, my TokenRegex abstraction was working against me.  I had created it in order to hide the use of the "\G" regex modifier as well as the convenient RegexOptions.Multiline option, which had the unintended consequence of *also* hiding the RegexOptions argument from the consumer.  Chris wants to include RegexOptions.IgnoreCase, but I'm preventing that for no good reason.  It's fine that the current behavior is the *default*, but it shouldn't be the *only* option.

<a href="https://github.com/plioi/parsley/commit/c33c63bf870253be503f900482a6d3a65f50180a">The fix was to update the TokenRegex constructor to optionally accept additional RegexOption values</a>, and to allow those new constructor arguments to be supplied by the consuming project:

```cs
public TokenRegex(string pattern, params RegexOptions[] regexOptions)
{
    var options = RegexOptions.Multiline | RegexOptions.IgnorePatternWhitespace;
 
    foreach (var additionalOption in regexOptions)
        options |= additionalOption;
 
    this.pattern = pattern;
    regex = new Regex(@"\G(
                               " + pattern + @"
                               )", options);
}
```

From now on, if I get a feature request on an open-source project, I should never have a knee-jerk reaction of "It already does that due to my infinite foresight."  Rather, I should go on the assumption that I really have left out or hidden something useful, and see what I can learn in the process.
