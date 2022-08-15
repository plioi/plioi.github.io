---
title: "Ain't Any Is All Ain't"
layout: post
---


Today, ReSharper made the same suspicious suggestion to both my coworker and me.  We both had the task of checking a potentially-large collection for items that fail to pass a test, and we both phrased these like so: "if not any of the items fail, continue...".  In each case, ReSharper wanted to rephrase our statements, and it seemed like following the suggestion would be a change for the worse! 

## An Offensive Suggestion

Here's what each of our situations boiled down to:

```cs
//You write this:

if (!items.Any(v => v == null))
    Console.WriteLine("None of these are null!");


//ReSharper suggests this:

if (items.All(v => v != null))
    Console.WriteLine("None of these are null!");
```

We leaned on the `Any()` extension method, with the intention of "short circuiting" the underlying loop.  `Any()` starts at the front of the list, and tests each item until one of them passes the condition in the lambda expression.  Once we hit that item, there is no need to continue processing items, so `Any()` returns early.  This is similar to the way that the `||` operator works: in the expression `(a() || b() || c())`, we only bother to call method `c()` when `a()` and `b()` are both false.

So, it seems the ReSharper suggestion is only *technically* the same.  You'll get the same answer, but surely testing "All" the items is potentially wasteful in the event that the collection is large.

What a terrible suggestion!  So terrible, I wanted to write an angry letter to the editor of my local newspaper: "...these nogoodniks, these ne'er-do-wells over at <a href="http://www.jetbrains.com/resharper/">JetBrains</a> need to **turn that racket down and get off my lawn!**"

I'm glad I didn't, though, because I was completely wrong.

## Oops

While searching for a code sample demonstrating this ReSharper suggestion, I found the truth: <a href="http://stackoverflow.com/questions/9027530/linq-not-any-vs-all-dont">Not Any vs All Don't</a>.

The top answer reveals the actual implementation of both `Any()` and `All()`.  Naturally, *both* `Any()` and `All()` are implemented with the "short circuit" logic mentioned above.  `Any()` behaves like `(a() || b() || c())`, **and `All()` behaves like `(a() && b() && c())`**.  Both have the opportunity to stop looking at items as soon as they have enough information to have the correct final answer.

My mistake was to read too much into the use of the word "All" without actually thinking about how it could be best implemented.  Yet again, ReSharper has proven itself to be clearly smarter than me.
