---
title: "The Cobra Effect"
layout: post
---


Imagine for a moment that you are a member of the government in colonial India during the period of British rule.  You're concerned about the population of venomous cobras, so you attempt to impose a solution.  You offer a reward to anyone who brings you a dead cobra.  After a while, though, you start to get suspicious of some of the cobra hunters.  They seem to be bringing in far more cobras that any single person is likely to encounter, let alone trap and kill.  You discover that some individuals have begun raising cobras in order to turn a profit!  Clearly, your plan isn't working, so you stop offering the bounty.  Countless cobra farmers' snakes become worthless, so they are released into the wild, increasing the population of cobras.

You meant well, but your attempt to impose a solution to the problem actually made things worse.

The <a href="http://en.wikipedia.org/wiki/Cobra_effect">Cobra Effect</a> is similar to the more frequently-referenced <a href="http://en.wikipedia.org/wiki/Murphy's_law">Murphy's Law</a>.  Murphy's a bit of a pessimist: if something can go wrong, it will, as if the universe is out to get you.  The Cobra Effect, on the other hand, has more to do with human nature: you see a problem and attempt to impose a solution on a group of people, and their natural reaction to your solution ends up making the original problem worse.

Cobra Effect problems have 4 parts: a) recognizing a problem, b) imposing a solution on a group of people, c) people react naturally *to the solution*, and d) in turn that reaction makes the problem worse.  Years ago, I experienced these 4 stages on a software project.

## Farming Venomous Comments

**The problem:** Reading other people's code can be frustrating when it doesn't communicate enough of the intent.  You may not feel tempted to add lots of comments to your own code, but sometimes when looking at something ugly or unfamiliar, you find yourself wishing the other developers *were* writing lots of informative comments.  The more experienced developers on the team naturally fear the ocean of undocumented code that the rest of the team is sure to write in the coming months.

**Imposing a solution:** We need to ensure everyone is writing documented code, therefore commits will not pass code review unless each method has complete <a href="http://msdn.microsoft.com/en-us/library/aa288481(v=vs.71).aspx">XML documentation comments</a>.  Each method will tell you what it is, what each of its arguments is, and what the returned value is.  The user of your code will see this information as they type via IntelliSense.

**People react naturally to the solution:** It *hurts* to write those XML comments.  They are verbose, they are usually redundant, they are time consuming, and they are usually redundant.  If your product is a library to be used by other developers, you owe it to them to document the non-obvious details of your public interface, but beyond that writing these comments is absolutely painful.  Enter <a href="http://visualstudiogallery.msdn.microsoft.com/46A20578-F0D5-4B1E-B55D-F001A6345748">GhostDoc</a>, a Visual Studio extension that writes these comments for you by inspecting the names and types of your methods and parameters.  You start with an undocumented method like so:

```cs
public IEnumerable<Order> GetMatchingOrders(Guid customerId, Func<Order, bool> condition)
{
    ...
}
```

You place your cursor on the first line, hit a keyboard shortcut, and it automatically turns into this:

```cs
/// <summary>
/// Gets the matching orders.
/// </summary>
/// <param name="customerId">The customer Id.</param>
/// <param name="condition">The condition.</param>
/// <returns>Matching orders</returns>
public IEnumerable<Order> GetMatchingOrders(Guid customerId, Func<Order, bool> condition)
{
    ...
}
```

Code Review Accomplished.

**That reaction makes the problem worse:** GhostDoc was *encouraged* because meeting the code review requirement *hurt* so much.  Every time I used it, a sarcastic little voice in the back of my head said "Oh, is *that* what a customerId is?" or "Oh, is *that* what that method does?"  That voice started getting louder and I soon started to block out all XML comments whenever I read through anybody's code.  It wasn't even a conscious decision: my eyes started scanning right by the comments as if they were whitespace.  *When I actually needed one of these comments to communicate something real about my intent, nobody would ever notice it; it was a an intent-needle in an angle-bracket-haystack.*

Compare that last sentence to the original problem: "Reading other people's code can be frustrating when it doesn't communicate enough of the intent."  The imposed solution made that problem *worse* by making it even harder to see important information about intent when the need actually arose, and it made it harder to see the intent that was already present in the code itself because you could only fit a few real lines of code on the screen at a time.  

## People Smells

We talk about "code smells", simple things we see in a snippet of code that give us a negative gut reaction, suggesting a deeper problem.  There are also "people smells", where a solution to a human problem should give us a similar negative gut reaction.  With the notable exception of dubstep fans, people naturally try to avoid pain.  We seek out the path of least resistence.  When the solution to a human problem hurts more than the problem, you may get exactly what you asked for.
