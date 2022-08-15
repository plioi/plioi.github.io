---
title: "Architecture Resists Change"
layout: post
---


In a conversation about large software projects, the word "architecture" is bound to come up sooner or later.  Most programmers have a gut feeling for what this really means with regard to software, but if you asked 10 programmers to define it, you'd get 10 different answers.

In <a href="http://martinfowler.com/ieeeSoftware/whoNeedsArchitect.pdf">Who Needs an Architect?</a>, Martin Fowler covers a few definitions which he ultimately boils down to one simple concept: **architecture is the decisions that are hard to change.**

Depending on the tone you use when reading that definition, it's almost a *pejorative*.  Joel Spolsky even warned us of <a href="http://www.joelonsoftware.com/articles/fog0000000018.html">Architecture Astronauts</a>, people who effectively over-engineer and abstract things to the point that they're no longer solving real problems.

Yes, **an** architecture can be bad.  If the decisions that are hard to change form obstacles between you and the true goal of your project, you've got a bad architecture.  On the other hand, some decisions that are hard to change may be a driving force for your success.  It's just important to be aware of which decisions form your architecture as you make them.

## Example: The Onion Architecture

With the <a href="http://www.headspring.com/onion-architecture-part-4-after-four-years/">Onion Architecture</a>, the hard-to-change decision is to define realtively-unstable infrastructure on the outside, depending on a core application that is ignorant of infrastructure details like persistence.

This decision is hard to change, but it is justified.  As Jeffrey Palermo said during a presentation on the O.A., an assembly is only as stable as its least-stable dependency.  Therefore, for an application to stand the test of time, we need to maximize the stable portion and minimize our dependency on things that are too likely to change.  If your app integrates with Twitter today, it likely won't in 5 years after Twitter goes the way of the blue Dodo bird, so you are better off defining your Twitter integration outside of the core assembly, dependent on the core assembly rather than the other way around.

## Example: Parsley

A few months back I described a hobby project to illustrate <a href="http://patrick.lioi.net/2011/11/11/recognizing-covariance-issues-in-your-code/">covariance issues</a>.  This project lets you parse text by describing what a valid input looks like (similar to regular expressions).  There were two key decisions that would be impossible to change without starting from scratch. First, the user would describe what a valid input looks like by gluing together tiny <a href="http://domaindrivendesign.org/node/127">side-effect-free</a> functions. Second, I would use read-only data structures to represent progress through the input.

These two decisions are made frequently when using functional languages, but not so much in C# projects.  Despite being unusual, I found these decisions to be extremely helpful for this kind of problem, regardless of the language used to implement it.

The samples contain a <a href="https://github.com/plioi/parsley/blob/0b0b1cdddab040c1513344f3b768a8c348bed138/src/Parsley.Test/IntegrationTests/Json/JsonGrammar.cs">JSON parser</a>, which amounts to describing what valid JSON looks like.  The bulk of the description is summed up in a few lines:

```cs
Json.Rule =
    Choice(True, False, Null, Number, Quotation, Dictionary, Array);

Dictionary.Rule =
    from pairs in Between(Token("{"), ZeroOrMore(Pair, Token(",")), Token("}"))
    select ToDictionary(pairs);

Pair.Rule =
    from key in Quotation
    from colon in Token(":")
    from value in Json
    select new KeyValuePair<string, object>(key, value);

Array.Rule =
    from items in Between(Token("["), ZeroOrMore(Json, Token(",")), Token("]"))
    select items.ToArray();
```

Because of the two architectural decisions, all the details about tracking progress, success, and failure can be completely hidden.  The user can solve the task at hand by describing the task at hand, and the architectural decisions enable that ability.

## Making Your Own Decisions

As you make a decision that affects your project's structure, ask yourself, "Is this a part of our architecture?  Is it going to be hard to change later if we go forward with it?  If so, does it *earn the right* to be included anyway?"  If not, try to restructure it so that it *is* easy to change, or consider chucking it entirely.
