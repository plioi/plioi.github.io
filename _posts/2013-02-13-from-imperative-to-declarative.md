---
title: "From Imperative to Declarative"
layout: post
---


Two weeks ago, we took a peek at the <a href="http://patrick.lioi.net/2013/02/01/whittling-parsley/">low-level pattern recognition classes</a> that make up the core of <a href="https://github.com/plioi/parsley">Parsley</a>.  Last week, we saw how pattern recognition involves 3 pivotal <a href="http://patrick.lioi.net/2013/02/07/composable-operations/">composable operations</a>: repetition, choice, and sequence.  We saw how Parsley implements basic repetition and choice operations, but sequence was still too hard to address.

Today, we'll finally see how to extract a useful sequence operation into a useful class, finally giving us the last of the necessary composable operations, and freeing us to finish our high-level <a href="http://martinfowler.com/bliki/DomainSpecificLanguage.html">DSL</a>.

## Manual Sequence Recognition

Recall the horrible imperative code necessary to recognize the trivial three-token sequence of [left-parenthesis, letter, right-parenthesis]:

```cs
public string IsParenthesizedLetter(Token[] resultFromTokenizerPhase)
{
    var tokens = new TokenStream(resultFromTokenizerPhase);

    var replyA = Require(LeftParen).Parse(tokens);

    if (replyA.Success)
    {
        var replyB = Require(Letter).Parse(replyA.UnparsedTokens);

        if (replyB.Success)
        {
            var replyC = Require(RightParen).Parse(replyB.UnparsedTokens);

            if (replyC.Success)
                return string.Format("Parenthesized {0}!", replyB.Value);
                
            return string.Format("Expected {0} at {1}.", RightParen, replyC.UnparsedTokens.Position);
        }

        return string.Format("Expected {0} at {1}.", Letter, replyB.UnparsedTokens.Position);
    }

    return string.Format("Expected {0} at {1}.", LeftParen, replyA.UnparsedTokens.Position);
}
```

To recognize a 3 token sequence, we need 3 nested if statements and 4 return statements.  To recognize an N token sequence, we need N nested if statements and N+1 returns.  Each if statement tests whether the last call to Parse(...) succeeded, and each if statement's body hands off the remanining unparsed tokens to the next step.  Clearly, this does not scale well.

## (Ab)using LINQ

Note how similar each step is: we always call Parse(...), then test for success, then pass the remaining unparsed tokens along.  The main thing that changes each time we dig deeper is the particular Parser object in play.  I'd rather just say "Require(LeftParen), then Require(Letter), then Require(RightParen), halting at the first point of failure".

Our goal is to extract the "success check and unparsed token hand-off" action so that we don't have to repeat it ourselves, and we want to do so in a way that lets us chain together any number of parsers representing a sequence of expectations, without having to indent deeper as the chain grows longer.  With no further ado, ParserQuery:

```cs
public static class ParserQuery
{
    /// <summary>
    /// Converts any value into a parser that always succeeds with the given value in its reply.
    /// </summary>
    /// <remarks>
    /// In monadic terms, this is the 'Unit' function.
    /// </remarks>
    /// <typeparam name="T">The type of the value to treat as a parse result.</typeparam>
    /// <param name="value">The value to treat as a parse result.</param>
    public static Parser<T> SucceedWithThisValue<T>(this T value)
    {
        return new LambdaParser<T>(tokens => new Parsed<T>(value, tokens));
    }

    /// <summary>
    /// Allows LINQ syntax to construct a new parser from a simpler parser, using a single 'from' clause.
    /// </summary>
    public static Parser<U> Select<T, U>(this Parser<T> parser, Func<T, U> constructResult)
    {
        return parser.Bind(t => constructResult(t).SucceedWithThisValue());
    }

    /// <summary>
    /// Allows LINQ syntax to contruct a new parser from an ordered sequence of simpler parsers, using multiple 'from' clauses.
    /// </summary>
    public static Parser<V> SelectMany<T, U, V>(this Parser<T> parser, Func<T, Parser<U>> k, Func<T, U, V> s)
    {
        return parser.Bind(x => k(x).Bind(y => s(x, y).SucceedWithThisValue()));
    }

    /// <summary>
    /// Extend a parser such that, after executing, the remaining input is processed by the next parser in the chain.
    /// </summary>
    /// <remarks>
    /// In monadic terms, this is the 'Bind' function.
    /// </remarks>
    private static Parser<U> Bind<T, U>(this Parser<T> parser, Func<T, Parser<U>> constructNextParser)
    {
        return new LambdaParser<U>(tokens =>
        {
            var reply = parser.Parse(tokens);

            if (reply.Success)
                return constructNextParser(reply.Value).Parse(reply.UnparsedTokens);

            return new Error<U>(reply.UnparsedTokens, reply.ErrorMessages);
        });
    }
}
```

**I apologize for this.**  This was hands-down the most difficult 12 lines of code I have written or will ever write.  The pattern in play here is so difficult, so abstract, that *I don't even know what to name some of the variables.*

This is the <a href="http://blogs.msdn.com/b/wesdyer/archive/2008/01/11/the-marvels-of-monads.aspx">Monad</a> pattern.  It can be performed in any language that has lambda expressions in which the lambda body can refer to variables in the surrounding scope, like C#, Javascript, and Python, but is only ever used in practice when the language *also* has special syntax for its usages.  In C#, the special syntax which makes this pattern compelling is the `from/select` LINQ syntax.

This special syntax lets us "flatten" our nested if statements into a series of 'from' clauses.  When using the complete Parsley DSL, we can rewrite IsParenthesizedLetter without any nesting:

```cs
public string IsParenthesizedLetter(Token[] resultFromTokenizerPhase)
{
    var tokens = new TokenStream(resultFromTokenizerPhase);

    var parenthesizedLetter =
        from left in Token(LeftParen)
        from letter in Token(Letter)
        from right in Token(RightParen)
        select letter;

    var reply = parenthesizedLetter.Parse(tokens);

    if (reply.Success)
        return string.Format("Parenthesized {0}!", reply.Value.Literal);

    return reply.ErrorMessages.ToString();
}
```

ParserQuery defines some extension methods that are treated specially by the compiler.  Their existence allows us to select from `Parser<T>` objects instead of the usual ability to select from `IEnumerable<T>` objects.  Note how instead of having 3 nested if statements, we have 3 from clauses *in sequence*.

Note how the query does little more than *list* the 3 expectations in the order we expect them to appear in the input.  We "flattened" the nested if statements into a vertical series of from clauses.

Note also how this sequence operation is operating on 3 `Parser<Token>` objects (the right hand side of each from clause) and the type of the whole query is *also* a `Parser<Token>`.  Since our sequence operation results in the same type as it acts on, it is highly composable with last week's repetition and choice operations.

Although complicated, there's no magic here.  If we ask ReSharper to translate the above select statement into plain old method calls, we see that ParserQuery's SelectMany method is being called N-1 times when there are N from clauses:

```cs
var parenthesizedLetter =
    Token(LeftParen)
        .SelectMany(left => Token(Letter), (left, letter) => new {left, letter})
        .SelectMany(@t => Token(RightParen), (@t, right) => @t.letter);
```

In other words, our SelectMany method's job is to hook a single from clause into the chain of execution.

C# rewrites a flat query into calls to the SelectMany extension method, which in turn does the success check and the passing along of remaining unparsed tokens, so all the same *work* is going on here as in the origial imperative code.  Thankfully, we're no longer the ones doing that work!

Rather than trying to make sense of ParserQuery itself, it may be more illuminating to read <a href="https://github.com/plioi/parsley/blob/cb69098da8135f7ac5fb1b0f84071e0e8b94b8a0/src/Parsley.Test/ParserQueryTests.cs">ParserQuery's unit tests</a>.

## So Where Are We?

2 weeks back, we started with the low-level imperative classes.  These let us deliberately walk through the input, pattern matching as we go along.  It was annoying, but it worked.  Like assembly language, we wanted to instead work with a high-level alternative, even if the high-level alternative was ultimately doing this imperative stuff behind the scenes.

1 week back, we identified 3 fundamental operations necessary for a pattern recognition DSL: repetition, choice, and sequence.  We saw how repetition could be implemented as a class, ZeroOrMoreParser.  We saw how choice could be implemented as a class, ChoiceParser.

This week, we see that sequence can be handled by ParserQuery and the from/select keywords.

**Now, we have all the tools we need to finally create our DSL.**

## Assembling Parsley's DSL

Parsley deals with defining grammars for languages.  You could define a grammar for a language like JSON, and use that to turn JSON text into .NET objects.  Parsley contains a Grammar class full of useful operations.  To keep things highly-composable, many of these operations both produce `Parser<T>` objects and operate on other `Parser<T>` objects.

First, the Grammar class includes a few helper methods for constructing the classes developed last week:

```cs
public abstract class Grammar
{
    public static Parser<Token> Token(TokenKind kind)
    {
        return new TokenByKindParser(kind);
    }

    public static Parser<Token> Token(string expectation)
    {
        return new TokenByLiteralParser(expectation);
    }

    public static Parser<IEnumerable<T>> ZeroOrMore<T>(Parser<T> item)
    {
        return new ZeroOrMoreParser<T>(item);
    }

    public static Parser<T> Choice<T>(params Parser<T>[] parsers)
    {
        return new ChoiceParser<T>(parsers);
    }

    ...
}
```

Next, we note that ZeroOrMore is useful, but not really enough in practice.  We may rather require that the input include OneOrMore occurrences of something.  Thankfully, we can *compose* sequence with ZeroOrMore to build OneOrMore.  OneOrMore things is the same as one thing followed by ZeroOrMore things:

```cs
public abstract class Grammar
{
    ...

    public static Parser<IEnumerable<T>> OneOrMore<T>(Parser<T> item)
    {
        return from first in item  //Require one item.
               from rest in ZeroOrMore(item)  //Collect zero or more other items.
               select List(first, rest);  //Combine the intermediate results.
    }

    ...
}
```

Where ZeroOrMore required a whole class full of imperative code, OneOrMore required *one 3-line expression*.

It's also very common to expect the input to contain a repetition of items *separated by* some other expected thing.  A JSON array literal contains zero or more items *separated by* commas.  Here, the third operation, choice, makes an appearance.

```cs
public abstract class Grammar
{
    ...

    public static Parser<IEnumerable<T>> OneOrMore<T, S>(Parser<T> item, Parser<S> separator)
    {
        //One-or-more separated things is a single required first item,
        //followed by zero-or-more occurrences of (separtor followed by item).

        return from first in item
               from rest in ZeroOrMore(from sep in separator
                                       from next in item
                                       select next)
               select List(first, rest);
    }

    public static Parser<IEnumerable<T>> ZeroOrMore<T, S>(Parser<T> item, Parser<S> separator)
    {
        //Zero-or-more separated things is EITHER
        //  one-or-more separated things
        //  OR
        //  zero things.

        return Choice(OneOrMore(item, separator), Zero<T>());
    }

    ...
}
```

Finally, a little payoff for all this infrastructure!  Imagine how horrifyingly verbose and error-prone these methods would be, if we had to implement them directly in terms of the low-level imperative style.

Lastly, there's one more built-in operation worth discussing as it directly affects the IsParenthesizedLetter method from the start of this post.  We often need to recognize that something important appears *between* two other, less important things.  A JSON array is zero or more items *between* the "[" and "]" symbols.  We care that the "[" and "]" are present, but the meaningful data we want to extract from that is the part in the middle:

```cs
public abstract class Grammar
{
    ...

    public static Parser<TGoal> Between<TLeft, TGoal, TRight>(Parser<TLeft> left, 
                                                              Parser<TGoal> goal, 
                                                              Parser<TRight> right)
    {
        //Expect three things in sequence, plucking out the one in the middle.

        return from L in left
               from G in goal
               from R in right
               select G;
    }

    ...
}
```

## Revisiting the Original Example

Now, in order to recognize a parenthesized letter, all we need is:

```cs
Parser<Token> parenthesizedLetter = Between(Token("("), Token(Letter), Token(")"));
```

Here, parenthesizedLetter is an object which can be handed the raw input tokens and can reply with one of two statements: "The input is not a parenthesized letter and here is why," or "The input is a parenthesized letter and here is that letter."  No nested if statements, no explicit progress through the token stream, no manual error checks at every step.

Most importantly, this says *what* it does rather than *how*.  **We've turned our imperative API into a declarative API by repeatedly extracting common and monotonous patterns into highly-composable operations, and by further hiding those helper classes behind a few simply-named helper methods.**
