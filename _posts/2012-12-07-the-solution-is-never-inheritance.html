---
title: The Solution is Never Inheritance
layout: post
---

That's how fellow-Headspringer <a href="https://twitter.com/scichelli">Sharon Cichelli</a> started a conversation today:  "The solution is never inheritance."  It's a controversial title.  Afterall, one must never, <em><strong>under any circumstances</strong></em> speak in absolutes.  "The maintainable solution is quite oft not inheritance" may be closer to the truth while missing the point entirely.  Judicious use of inheritance in small chunks can be useful, but implementation inheritance in the wild is usually a mess that gets out of hand quickly.

The general advice to <a href="https://www.google.com/search?q=composition+over+inheritance">favor composition over inheritance</a> has been written about countless times.  In <a href="http://www.headspring.com/patrick/spaghetti-alla-code/">Spaghetti alla Code</a>, I blamed the problem on the way that <code>virtual/override/base</code> keywords achieve code reuse at the cost of littering your implementation with subtle <code>goto</code>-like complexity: you have to execute every line of a class hierarchy in your head in order to know what any one class is actually going to do.  When favoring composition, you're more likely to understand what a class does by just reading that class.

I don't always use implementation inheritance, but when I do, I use the Template Method Pattern and the Bucket of Small-Scope Helper Methods Pattern.

<h2>The Template Method Pattern</h2>

In this pattern, an abstract base class provides a basic algorithm, leaving a detail or two undefined as abstract methods.  A single-level of child classes completes the story by providing implementations for the undefined steps.    I never decide up front to use this pattern.  Rather, I refactor toward it when it's the clearest way to remove some code duplication.

<h2>The Bucket of Small-Scope Helper Methods Pattern</h2>

Sure, this isn't an official Gang of Four pattern, but it comes up often enough that it deserves a name.  I arrive at this kind of inheritance incrementally.  If I have an <em>interface</em> whose implementations take on suspiciously-similar private helper methods, and the duplication mounts up, I promote the interface to an abstract class and move the common private methods up to the base class.

When the scope of these helper methods is very small, the base class is an appropriate home for them, but watch out: you may be hiding a new concept here that would be more useful to the rest of your system as a separate public class.

I use the BoSSHM Pattern when the thought of a new public class feels like overkill or overengineering.  I don't want to distract the reader with rarely-used classes, just for the sake of having <em>small</em> classes.

<h2>Evolve Towards Inheritance</h2>

Off the top of my head, I can't think of any other time I use implementation inheritance and later think to myself, "That was a good idea!"  Also, the way I arrive at these patterns is just as important as the result: rather than being the product of up-front design, I arrive at them in small steps as code duplication pains present themselves.  Each solves <em>small</em> kinds of code duplication problems, so the resulting inheritance hierarchies are also small and therefore easy to understand.