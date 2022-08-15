---
title: "Polaroid Mutation"
layout: post
---


Last week, we saw how <a href="http://patrick.lioi.net/2011/12/16/clojures-world-view/">Clojure's approach to syntax</a> can influence the way we think while using other languages.  Today, we'll see how Clojure's strong opinions about state change can similarly affect the way we think in our language of choice.

Some languages embrace mutation (the changing of variables' values at runtime).  Python and Ruby classes have publicly-settable *everything* and although this can get you into trouble it can also lead to some pretty slick tools like Ruby on Rails.  Clojure, on the other hand, fears mutation.

## .NET Strings are Immutable

.NET's `System.String` is a good example of Clojure's approach to collections.  Once you have a string instance, you cannot change its content.  If you want to change the string you are currently referencing, you have to instead create a whole new string (often based on the original), and then point your reference variable at the new one.

Although potentially-wasteful with memory, this approach has saved us from a lot of potential complexity.  When I pass a string around, I know it's not going to be altered by some rogue library call.  I can expose a string across threads without fear that it will be in mid-change when I try to look at it.

This immutability can be problematic for large strings that are being frequently 'changed', but in practice most of the change we deal with is with small strings where the performance hit is trivial.  In those cases where it *really* matters, we can use something like `StringBuilder` to avoid being wasteful with memory and time.

## .NET Lists Mutate

.NET's `List<T>`, on the other hand, is mutable.  We can think of strings as an immutable collection of `char` values.  A `List<char>`, on the other hand, can be **changed** rather than simply **replaced**.  You can add, remove, and alter each item in the collection, *and anyone looking at the object will see those changes right away, whether they like it or not*.

## Clojure Collections are Immutable

The built-in data structures in Clojure are immutable.  The two main types here are lists (corresponding with .NET `List<T>`), and maps (corresponding with `Dictionary<K,V>`).

In Clojure, when you have a list containing 1000 elements and you wish to append item number 1001, you don't change the contents of the original list.  Rather, you get a whole new list containing 1001 elements.  This is exactly what we would do with a .NET string to append a single char.

For maps, adding a new key/value pair likewise starts by copying the original collection and including the new pair.  Anyone who is still holding a reference to the original sees the original contents unaffected.

That may seem wasteful, since the use cases for lists are far more varied than for strings.  Aren't we going to feel the pain of this waste too easily in practice for it to be a good idea?

Thankfully, Clojure collections are clever.  Behind the scenes, <a href="http://blog.higher-order.net/2009/02/01/understanding-clojures-persistentvector-implementation">lists and a maps are really trees</a>.  The items are stored in the leaves, and whenever we need to create a "whole new collection", the new collection will share as much of the original tree as it can sneakily get away with.  The new collection is safe to share data with the original tree because, after all, nobody is *allowed* to change it.

Despite storing items in a tree, Clojure lists can still provide the *near* constant-time lookups we expect from lists.  These trees are exceptionally *wide*, but not very *deep*.  Even a fully-populated tree, with more items in it than you'd ever care to work with, is only about 6 levels deep.  *Slower than a .NET array lookup, for sure, but still O(1)*.  Well, O(1-*ish*).

## Course-Grained Mutation

Clojure does allow some mutation.  It would be difficult to use a language if it had *no* notion of assignment statements.  A small number of special reference types are available.  These are objects which can point at some other object, and this pointer can be requested to change to point to some new object.  These types are each opinionated about how this re-pointing can act with respect to threads, but the basic idea is that they allow for course-grained mutation.

Let's say I have one of these reference objects currently pointing at a map containing many name/value pairs, and that I'm using <a href="http://clojure.org/refs">a reference object that ensures transactional re-pointing in the presence of multiple threads</a>.  If I have to perform a significant change to the collection, **I'd like to be able to take my time to calculate the new version of the collection, without having to lock the original**, so the other threads can keep on trucking along.  Once I've calculated my new collection, sharing a subtree behind the scenes with the original, I'll ask my reference object to re-point to the new collection as an atomic operation.  Now, all the other threads will see the new state.

*C# represents state change by physically altering an object in-place.  Clojure represents state change by tracking a series of object snapshots.*  **The river isn't flowing by you in front of your eyes.  Instead, you have a stack of river-photos representing the key moments of its flow.**

## Minimizing Mutation in Your Language of Choice

Clojure's weird, and weird is good.

Some of the worst spaghetti code I've ever dealt with was hard to untangle **precisely** because of the fine-grained state change involved.  The first thing I do when faced with a tricky mess like this is to start identifying those parts of the code that are already lacking mutation, pulling them out into <a href="http://en.wikipedia.org/wiki/Pure_function">pure functions</a>.  Next, I look for ways to rephrase the existing mutation-heavy bits with equivalent pure alternatives.  **Often, the percent of your codebase that actually needs to deal with mutating state is remarkably small.**
