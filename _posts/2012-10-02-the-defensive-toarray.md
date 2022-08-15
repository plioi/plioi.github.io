---
title: "The Defensive ToArray"
layout: post
---


In <a href="http://patrick.lioi.net/2012/09/27/a-game-of-throws/">A Game of Throws</a>, we saw a frequently-blogged-about gotcha regarding the `yield` keyword and exception handling.  This week, we'll see how there is more than one motivation for using the `yield` keyword, and when you opt into it for one you get the other whether you like it or not.  This leads to a code smell: littering our code with 'defensive' calls to ToArray(). 

## Generalizing the Exception Gotcha

The exception handling case from last week is a specific example of a larger potential gotcha: with lazy evaluation, some work *happens* at a different time than the caller might expect.  It allows exceptions to get "smuggled" through try/catch blocks, but exceptions are just one way you might be surprised by the change in the order of events: if your `yield`ing method has any kind of side effect, the caller is likely to experience the odd order of evaluation.

That can be good.  That can be bad.  It's a sharp tool.

## Complecting Brevity with Laziness

This week, I'd like to cover another problem with lazy evaluation.  Maybe it's really just a corollary of the order-of-events gotcha, but I want to call it out specially because it has to do less with the nuts-and-bolts of code execution and more to do with the emergent human behavior that results *from* people's understanding of the problem: the Defensive ToArray.

There are two reasons to use the `yield` keyword:  laziness and brevity.

**Laziness is the advertised feature:** you can produce a collection's items on demand, as the caller tries to loop through them.  Laziness enables awesome things like assembling LINQ queries over multiple statements before finally starting to iterate through the overall result.

**Brevity is the tempting, likely-unintended feature**: a really monotonous pattern can be shortened using the `yield` keyword:

```cs
public IEnumerable<string> TableOfContents(Book book)
{
  var result = new List<string>();

  foreach (var chapter in book.Chapters)
    result.Add(FormattedTitle(chapter));

  return result;
}
```

It's tempting to look at that explicit list create/Add/return pattern and say, "I can shorten that with `yield`!"

```cs
public IEnumerable<string> TableOfContents(Book book)
{
  foreach (var chapter in book.Chapters)
    yield return FormattedTitle(chapter);
}
```

Or even shorter:

```cs
book.Chapters.Select(FormattedTitle);
```

...but even that uses `yield`'s semantics under the surface.

The problem is that we opted into `yield` (or equivalently Select) for the brevity, but we *got* laziness along with it.  Now we all have to be wary every time we call a method that returns an IEnumerable.  My inner monologue starts to sound like the following:

> Is this call lazy?  Would it be unsafe to pass the result around before defensively *realizing* it with a call to .ToArray()?  If I *do* that .ToArray() call defensively, am I stomping all over the original developer's intentions?

Now I have to start looking at the implementation of a method I'm calling in order to know whether or not its result is to be immediately distrusted.  *Do I have to finish writing that other developer's method with a .ToArray because they were being particularly... er... lazy?*

I'm starting to think that most calls to .ToArray() are a code smell, or rather an in-your-face workaround for a code smell.  Even when it's doing the right thing, it's too wordy and puts the onus on the caller rather than the guilty callee.  I don't want to have to fix the result of another method, and I don't want to have to look at a thousand such fixes littered throughout a project.

## Let's Start Communicating Intent

In response to all this, I have two items on my wishlist.  One is a wish for an addition to C#, and the other we can use in our projects today.

First, I wish that if a `yield`ing method were declared to return an array or anything that has a constructor accepting an `IEnumerable<T>`, that the items be immediately iterated through and packaged as that type.  This way, you could get the brevity while communicating that you are not interested in the side effects of laziness:

```cs
//Hypothetical C#
public string[] TableOfContents(Book book)
{
  foreach (var chapter in book.Chapters)
    yield return FormattedTitle(chapter);
}
```

Second, something we can do today: I wish that we could start standardizing on a naming convention for extension methods similar to Select that are not lazy: Map is a common term in other platforms.  Select would still lazily produce one IEnumerable from another, but Map methods would always do a defensive .ToArray for you.  You'd have to write a Map overload for every collection type you care about, but there's really just a handful of collection types you'll care about in practice.  For instance:

```cs
public class MapExtensions
{
  public TOut[] Map<TIn, TOut>(TIn[] items, Func<TIn, TOut> map)
  {
    return items.Select(map).ToArray();
  }

  public List<TOut> Map<TIn, TOut>(List<TIn> items, Func<TIn, TOut> map)
  {
    return items.Select(map).ToList();
  }
 
  //...etc
}
```

Without these Map methods, every time you see a Select you have to wonder what the original developer's intention was.  **If instead Map was the first method you reached for, then you would only reach for Select when you actually wanted lazy evaluation.**    Reading through someone else's code, Select calls would communicate deliberate laziness with potential side effect gotchas, and Map calls would communicate deliberate brevity with no such gotchas.
