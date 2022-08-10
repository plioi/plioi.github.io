---
title: 'Correcting My Confidently Incorrect Span&lt;T&gt; Model'
layout: post
---
Posts in this series:

1. [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/)
2. [Detecting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/)
3. [Correcting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/10/correcting-my-confidently-incorrect-span-t-model/)
4. Benchmarking Parsley's `Span<T>` Upgrade

In the previous post, I [detected that I had a Confidently Incorrect Mental Model for `Span<T>`](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/). I had a series of similar ideas bubbling up, but since they were all built atop a weak foundation, they weren't helping. I was stuck in a loop of generating similar ineffective ideas over and over.

**The answer to the "thought bubbling" trap here is to deliberately and radically wrench yourself out of that train of thought by moving down the ladder of abstraction, way way down to the absolute nuts and bolts fundamentals you *do* know to be real.** Set your mental model aside. Take incredibly small steps. Make many tiny commits. Seek feedback for each tiny step. When feedback confirms your expectations, include it in the new mental model. When feedback rejects your expectations, stop the presses because this is an important moment of correction to the model.

# 300 Tiny Adjustments

In [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/), examples included:

- Building an atypical WPF UI interaction from very tiny easier-to-build interactions, using logging and continuous running of the app for visual feedback.
- Learning git by falling back to the simplest subset of commands, using an explicit-refresh history viewer to continuously test my expectations.

In this case, I had an existing nontrivial library with solid test coverage and a public API I largely liked already. A complete rewrite would have been doomed to failure. In this case, moving down the ladder of abstraction to well-known fundamentals took a different flavor than the earlier two examples:

1. Setting aside the big fancy architectural moves I had been trying.
2. Favoring instead incredibly tiny refactorings in the area I was trying to introduce `Span<T>`.
3. Using the tests and the compiler for quick feedback on these tiny changes.
4. Commiting frequently with descriptive messages, so I could witness and revisit my own train of thought as I made progress over several weeks.

Rename this variable. *Commit.*

Extract this method. *Commit.*

Rephrase this interface as a delegate type. *Commit.*

Split this class into two, when the original mixed non-span ideas with potentially-span ideas. *Commit.*

Adjust the parameter list on this `delegate` type. *Commit.*

On and on... for about 300 commits all told. After the first several such commits, I had at least successfully wrenched my "thought bubbling" train of thought out of the realm of high level (but poor) `Span<T>` usages and down to a level of abstraction that I *did* have a strong model for: safe refactoring towards simplicity. The more commits I made at this level, the more votes I cast that the next idea would be at that same low level.

Zooming out a bit, the overall movement in the code was one of simplifying what I wrote all those years earlier. I was simplifying pain points in the design which made my earlier attempts with `Span<T>` too all-or-nothing, finally uncovering a few tiny seams where I could introduce *just a little bit of high confidence `Span<T>` code*. Not enough to reap the benefits, but enough for a foothold and enough to start the foundation of my new and improved mental model for what these things are. By taking smaller steps with these tiny refactorings, I was finally able to spot those small foothold opportunities, and could then spot ways to let the span work spread from there to have a larger and larger impact.

Over those 300 commits, I also had the chance to arrive at the idea of spans from many directions, each time at a very small and constrained scope. Each time I aimed at spans in this tiny way, I improved my mental picture of what was going on, what I was allowed to do, what I was not allowed to do, and why all that was the case. After a while, I found myself encountering fewer and fewer compiler error messages and more naturally identifying places where I could use a span effectively. I found myself naturally climbing back up that ladder of abstraction to think about bigger architectural moves.

# Iterating `Parser<...>`

The changes over these 300 commits are of course a bit much to cover in detail. The point here isn't to gain insight into how the library works, or even insight into spans, but to see the nature of the tiny steps being taken: **small moves, made with high confidence, repeatedly simplifying away things that had been painful every time I previously tried to introduce spans.** It was general cleanup, but general cleanup in the intentional direction of the end goal.

To get a feel for the overall work, though, it helps to watch the evolution of the `Parser<...>` type over time. This type happens to be pivotal to the library and represents any function that does a very small amount of work parsing the original sequence.

For instance, a parser function might recognize and consume a single digit character, or might recognize an repetition of some other parser. You might then combine those two parser functions to get yet another little function the recognizes and consumes a series of digits.

Although the real work happens in tiny functions, this type started out as an interface wrapping that one function. It would accept an abstraction over the original text and produce a `Reply` representing the successful matching of an expected `T` value, or failure to match such a thing.

```cs
public interface IParser<out T>
{
   Reply<T> Parse(TokenStream tokens);
}
```

I renamed `TokenStream` to `Input`, as an old "token" concept was getting phased out:

```cs
public interface IParser<out T>
{
   Reply<T> Parse(Input input);
}
```

As the `Input` and a related `Text` type both got simpler, it became clear that I was repeating myself. `Input` was deprecated and I moved towards using `Text` throughout. By this point, I knew of several concepts that were in the way of introducing spans, even if I didn't quite see how spans would arrive. Even this removal of `Input` in favor of `Text` was just an intermediate step towards phasing out `Text` as well. All that mattered in the moment was that the number of obstacles reduced by one:

```cs
public interface IParser<out T>
{
   Reply<T> Parse(Text input);
}
```

Representing these little functions as an interface with small implementing classes got a little verbose. I really wanted to have a delegate type and tiny lambda expressions instead. That was a big pivot, so to minimize the number of manual touches to the code in that commit, I started with commit containing only an automated rename of `IParser` to `Parser`. In the moment, this was violating the C# naming convention, but it was a brief moment in history and eased the next step:

```cs
public interface Parser<out T>
{
   Reply<T> Parse(Text input);
}
```

Now I could more aggressively pivot to a delegate type. The previous rename let this commit focus on the monotonous removal of interface/class boilerplate without mixing the effort with frequent `IParser`/`Parser` renames. **One kind-of-idea at a time** made the work easeful despite the size of the change over the whole solution:

```cs
public delegate Reply<T> Parser<out T>(Text input);
```

Finally, things were starting to get simple enough, and my awareness of spans and related concepts like `ref` and `ref struct` starting to grow confident enough, I could start to get them into the mix. As `Text` pivoted from a `class` to a `ref struct`, the impact showed up here in the parser type:

```cs
public delegate Reply<T> Parser<out T>(ref Text input);
```

The old `Reply<T>` concept was feeling its age. It was an old way of dealing with concepts that are better handled today with nullable reference types. In addition to letting me drop a lot of annoying code, this let all the usages benefit from better compiler checks. The compiler could *know* about success (true) vs failure (false) and ensure that code paths at the call sites were sound as they tried to access the two `out` parameters formerly hiding in `Reply<T>`.

```cs
public delegate bool Parser<T>(
   ref Text input,
   [NotNullWhen(true)] out T? value,
   [NotNullWhen(false)] out string? expectation);
```

`Text` was a glorified `string` reference and 2D `Position` value, suspiciously similar in concept to a `Span<char>`. As a step towards simplifying that all away and enabling the phase-out of `Text`, I simplified `Text` and passed around the `Position` as a separate argument:

```cs
public delegate bool Parser<T>(
   ref Text input,
   ref Position position,
   [NotNullWhen(true)] out T? value,
   [NotNullWhen(false)] out string? expectation);
```

**Here, spans started to gain a foothold both in the code and in my mental model.** I was able to phase out enough old obstacles and simplify the remaining concepts to the point where spans could really show up as the representation of the text under processing. It was incredibly satisfying to conceive of this change, write it, and see the compiler was happy. My mental model for *this much span stuff* was finally right:

```cs
public delegate bool Parser<T>(
   ref ReadOnlySpan<char> input,
   ref Position position,
   [NotNullWhen(true)] out T? value,
   [NotNullWhen(false)] out string? expectation);
```

2D `Position` was needlessly complicating the inner workings of the library, with consequences around excessively recalculating substrings, so I reduced it to a simple `int` index:

```cs
public delegate bool Parser<T>(
   ref ReadOnlySpan<char> input,
   ref int index,
   [NotNullWhen(true)] out T? value,
   [NotNullWhen(false)] out string? expectation);
```

**At this point, each new change was more significant, higher up the ladder of abstraction, and more confident in its use of spans.** See how we're assuming `char` sequences there? I wanted to generalize to recognizing patterns in any sequence, not just sequences of text.

However, a system wide change of `Parser<T>` to a proposed `Parser<TItem, TValue>` would have introduced a hundred compiler errors, each needing attention until I could finally see if the concept worked. No thanks!

As a step towards that big and uncertain move, I first duplicated the type. I could then opt into the new type bit by bit over several commits without breaking the whole library in the process. This enabled me to get **constant positive feedback about my progress** even during a "big" architectural change:

```cs
public delegate bool Parser<TValue>(
    ReadOnlySpan<char> input,
    ref int index,
    [NotNullWhen(true)] out TValue? value,
    [NotNullWhen(false)] out string? expectation);

public delegate bool Parser<TItem, TValue>(
    ReadOnlySpan<TItem> input,
    ref int index,
    [NotNullWhen(true)] out TValue? value,
    [NotNullWhen(false)] out string? expectation);
```

Once everything used `Parser<TItem, TValue>` instead of deprecated `Parser<TValue>`, I could drop the old one:

```cs
public delegate bool Parser<TItem, TValue>(
    ReadOnlySpan<TItem> input,
    ref int index,
    [NotNullWhen(true)] out TValue? value,
    [NotNullWhen(false)] out string? expectation);
```

At this point, the whole library used spans throughout.

So, what do we have after all that? A `Parser<TItem, TValue>` is a function that can accept a lightweight view into the sequence of items being parsed, an `int` representing the current position in that sequence, does work to decide if the pattern in question is being met, and uses the `out` parameters to report success (some matched value) or failure (some missed expectation). Most end user code doesn't have to worry about this contract directly, and instead works with it indirectly via more useful public entry points, but this contract does form the fundamental operation that the rest of the library is built on.

By taking incredibly small steps, I essentially performed open heart surgery on the library while safely introducing spans and building up my mental model for spans. The automated tests passed on each commit, and provided a much needed safety net throughout.

# Autopilot

After these many little wins at the lower level of abstraction, and after a number of little wins at the higher levels of abstraction once spans gained a foothold, things I would have expected to be difficult started happening more naturally. With essentially no effort now, I would "bubble up" an idea for what to try next, immediately "bubble up" the very closely related realization-idea of what the compiler wouldn't like about it, and immediately "bubble up" the next refactoring that would get that obstacle out of my way to try again.

At this point, I was technically thinking, and thinking about big complex things, but it was *easeful* as if someone else was at the wheel and I was just watching it happen. This is what it feels like when your mental model is helping and you're sitting on the most effective rung of the ladder of abstraction. Each little random idea that pops up is at least in the right ballpark, so the effort involved to evaluate ideas and course correct is light.

I could finally *sit back comfortably* and let more useful high-level ideas bubble up, like meaningfully generalizing from `Span<char>` to `Span<T>` to allow recognizing patterns in any sequence (useful when parsing a programming language), not just for sequences of characters. Having that idea back when I had a poor mental model would have just been yet another faceplant, but was now a breeze.

It's no great feat of mental strength: the singular moment of strength was days and weeks earlier when I wrenched myself down from the high rung of abstraction and insisted on performing just a few tiny commits at the rung of confidently *correct* work. Anyone is capable of that short term moment of mental strength. The rest is essentially autopilot where the main course correction you're looking for is whether the bubbled-up ideas result in *frictional* vs *easeful* progress. 

> Thinking hard is a bug, not a feature; a symptom, not a cure. We can unblock ourselves by reducing the need to think hard, doing the exact opposite until the sloppy thought-generating machine between our ears has a chance to do its job naturally again.

When the mental model and the level of abstraction are aligned, it no longer feels like work. Flow takes over. Your ego shuts the heck up. Hours of progress can come and go in the blink of an eye leaving you feeling refreshed. When the mental model and the level of abstraction are misaligned, we can recognize that friction and simply pivot to fundamentals and feedback systems until flow takes over and brings you back on course.

That's a lot of fancy talk, but did it really help? In the next post, we'll see benchmarks of Parsley before and after this application of `Span<T>`, using a few popular alternatives as a baseline.