---
title: "Escape Analysis"
layout: post
---


In languages like Java and C#, most objects wind up on the heap instead of the stack.  Heap-allocated objects can only be cleaned up by garbage collection, while stack-allocated objects get cleaned up 'for free' when the function that they were created in returns.

If you have a method that constructs a new instance of a class, uses it a little, and then discards it, there's no real reason to put it on the heap.  It's not going to need to exist after we return, so we might as well put it on the stack and get the deallocation for free.

If you instead have a method that constructs a new instance of a class and *returns* that to the caller, we **definitely** have to put it on the heap, because we need to ensure it still exists after we return.  There are lots of things we might do with an object that would require us to use the heap.  We can return it, pass it to another thread, pass it to another function that passes it to another thread, etc.

The basic idea is that if, after your function exits, there could not possibly be any remaining references to that object, then it might as well be put on the stack and cleaned away without the need for garbage collection.

<a href="http://en.wikipedia.org/wiki/Escape_analysis">Escape analysis</a> is a compiler optimization that can be applied to languages like these.  If the compiler can prove to itself that the object can't "escape" beyond the lifetime of the method, it will use the stack.

Although escape analysis really just refers to this compiler trick, I like to use the term to describe a specific kind of refactoring effort.  Before I get to that, though, we need to talk about your arch-nemesis.

## Code Doesn't Just Rot

Over the long life of a software project, certain parts of the code will become more difficult than they deserve to be.  One area might be a frequent source of bugs, and the fixes applied by many hands may make it needlessly complex, for instance.

Sometimes it seems that even when an area doesn't go through much flux, it still rots.  Newer and better ideas are introduced to the project, and one area may be left behind the times.  By the time you come across it again, it feels like it's rotting away.

Code rot taken to its extreme gives you a <a href="http://miksovsky.blogs.com/flowstate/2010/09/every-app-has-a-scary-basement.html">spooky basement</a>, the part of the project *nobody* wants to touch.  It's a tangled mass of patches and awkward abstractions.

I don't really like the code rot metaphor because it makes the problem code out to be a passive byproduct.  I prefer to think of bad/awkward/old design decisions as being an **active and malicious adversary hell-bent on escaping from the confines of its files** with the goal of making the rest of your project rot along with it.

Code doesn't just rot.  The rot escapes and spreads.  This thing is out to get you.

## Escape Analysis at Refactoring Time

When I'm working in a clean area of a project, refactoring is an easy process.  You rename things to make them read better, extract a method or class here or there to organize responsibilities better, and the like.

When I'm working on a spooky basement, or even a not-so-bad eerie breakfast nook, I put on the Escape Analysis Hat.  Unlike the compiler optimization which asks "At runtime, how can this *instance* escape into the rest of the running system", I ask "How do *references* to this unwanted method/property/field/class escape into the rest of the code?"

This starts with ReSharper, but tools like this can only get you started.  You find usages of the unwanted thing's identifier, and then find usages of those usages, and usages of *those* usages, to start to get a feel for what other areas depend on it.

That's not enough!  Remember, this method/property/field/class is actively trying to escape into the rest of your system.  It wants to escape in any sneaky, devious way it can.  Maybe it gets AutoMapped to some other type, whose usages you'll need to investigate.  Maybe it gets serialized to JSON which in turn gets consumed by some javascript you'll need to investigate.  Maybe it gets accessed via reflection, or appears in a binding in a XAML file, or gets passed to a web service...

## A Particularly Tricky Example

I was recently refactoring an area of a project that wasn't a serious problem yet.  Left alone, the complexity would have eventually gotten out of control, so we decided to address a particular source of the growing complexity.  I encountered a property in a View Model class that I wanted to simplify; I didn't quite see the need for it at first.  After seeing how references to this property escaped into the rest of the system, my first thought was "Oh, this really does get used in a lot of places, and not just in the UI layer.  It needs to exist."

After looking into the escape paths again, though, I realized it was largely unnecessary.  It satisfied a UI requirement, but that could have been satisfied in a simpler way and in only the UI layer.

The escape path was pretty intricate.  This property had a few real and meaningful usages.  Additionally, it got AutoMapped to another type, which ultimately got serialized and later deserialized so it could be AutoMapped *back* to the original type.  This circuitous route was not actually used for anything (anymore) and could be removed.

The next time you find yourself refactoring, and you reach a point where it's not obvious what the next step should be, find a suspicious identifier and see where it's escaping to.
