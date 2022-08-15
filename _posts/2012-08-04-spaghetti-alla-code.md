---
title: "Spaghetti alla Code"
layout: post
---


This is part 4 of an N-part series on identifying and resolving incidental complexity in your software projects.  This post assumes you've already read the earlier posts:

1. <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Essential vs Incidental Complexity</a>
2. <a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">Identifying Incidental Complexity</a>
3. <a href="http://patrick.lioi.net/2012/07/29/scope-creep/">Scope Creep</a>


## Recipe: Spaghetti alla Code

INGREDIENTS

* 1 cup 'virtual' keywords
* 3 cups 'override' keywords
* 1 pinch of 'base' keyword
* 1 generous dollop of moxie


DIRECTIONS

1. Combine ingredients in large mixing hierarchy.
2. Season to distaste.


SERVES: no one

## Composition Over Inheritance

Last week, I took a suspiciously-growing class, Scope, and used subclassing to split it into a class hierarchy.  The goal was to start separating some of the concepts that had gotten all mixed up, and to better-communicate how each *kind* of Scope was to be used.  I ended that post with some vague criticism about disliking the use of 'virtual', 'override', and 'base' keywords.  I felt that using subclassing was a useful tool to help separate responsibilities, but the result was more complex than necessary.

These keywords helped me to avoid copy/pasting code across all the subclasses, and allowed subclasses to customize that behavior where appropriate, so how bad could they be?  It all worked, and I didn't have to reach for my copy/paste shortcuts.

<a href="https://github.com/plioi/rook/blob/8155326c6c428cdd2bf63e9e4986796a8a20fac7/src/Rook.Compiling/Scope.cs">The code</a>'s a little hard to follow, though.  The base class contains some `virtual` methods with a default implementation.  That implementation is used by GlobalScope and TypeMemberScope because that happens to be the right thing to do for those subclasses.  LocalScope `override`s them, deliberately using the `base` implementation in *one* of those `override`s and deliberately *avoids* the `base` implementation in another.  Why?  To find out, you have to run the code in your head, and in order to do that you have to start looking around at all the implementations of all the methods in the hierarchy.  

> With virtual/override/base, you have to understand *everything* to understand *anything*.

These three keywords combine to produce the same kind of <a href="http://en.wikipedia.org/wiki/Spaghetti_code">spaghetti code</a> as a `goto` does.  The compiler knows what you mean, but the reader doesn't!

When possible, I prefer to favor composition over inheritance.  We still want to avoid copy/pasting logic around, but composing simple objects together is another way to accomplish that goal, and the results are easier to read.

## Applying Composition in the Scope Hierarchy

> In previous weeks, we've improved upon incidental complexity by removing bad abstractions and splitting up abstractions that were getting too complex.  This time, we'll improve upon incidental complexity by recognizing that one abstraction was hiding within another.

The code I didn't want to copy around appeared largely in the base class of <a href="https://github.com/plioi/rook/blob/8155326c6c428cdd2bf63e9e4986796a8a20fac7/src/Rook.Compiling/Scope.cs">the troubled implementation</a>.  I had a dictionary with a particular key type and value type, representing the types of the identifiers found in a given scope ("bindings" in compiler terms), and I wanted access to the dictionary to be strict about uniqueness of keys while not throwing exceptions (programmers mistakenly create duplicate identifiers all the time, so it's not an exceptional occurrence).

This special kind of dictionary deserves to be a first class citizen of my compiler.  Let's pull it out into its own class, separate from the class hierarchy but *used* by the class hierarchy to avoid duplicated logic.

I extracted the new abstraction, `BindingDictionary`, from `Scope`.  The abstract base class became empty, so I converted it to an interface:

```cs
public interface Scope
{
    bool TryIncludeUniqueBinding(Binding binding);
    bool TryGet(string identifier, out DataType type);
    bool Contains(string identifier);
    bool IsGeneric(TypeVariable typeVariable);
}
```

(On for-pay projects I follow the IFoo interface naming convention for consistency across the team.  I don't on this project because... because... because I say so that's why.)

Contrast <a href="https://github.com/plioi/rook/blob/8155326c6c428cdd2bf63e9e4986796a8a20fac7/src/Rook.Compiling/Scope.cs">the previous implementation</a> with <a href="https://github.com/plioi/rook/blob/59cb30404aba89cd8bc94a761e1194e79210afab/src/Rook.Compiling/Scope.cs">the new implementation</a>.  These classes make use of the new `BindingDictionary` abstraction in order to reuse that logic without inheritance and overrides.  To understand any class, you need only look at that one class.  You don't have to run it *all* in your head to know what it's actual behavior is going to be, because **each class is satisfying the `Scope` contract on its own**.  Also, I got to give more descriptive names to the member variables because the member variables are no longer shared across multiple classes with different meanings.

Subclassing isn't all bad.  I find myself using the Template Method pattern, for instance.  In that case, though, I'm only overriding methods that are abstract in the base class.  When you start overriding non-abstract methods, though, you start lying to the reader:

> "This virtual method here does *this*, unless it doesn't.  It probably doesn't."

## Criticism and Next Steps

Some of these classes actually got *bigger*, which can be a code smell, but in this case they are mostly one-liners, and in my mind I exclude one-liners when considering lines-of-code counts.  The world needs more one-liner methods.

Now that `Scope` is just an interface, it's easier to take a quick glance at it and know that one of the methods on it is standing out like a sore thumb.  Scopes represent which identifiers are visible at some point in the code being compiled, along with their declared types; the `TryIncludeUniqueBinding, TryGet, and Contains` methods serve that abstraction.  The `IsGeneric` method doesn't serve that concept, but lives in these classes in order to produce the right results.

Looks like we've got another poor abstraction here: a useful abstraction is missing completely, and needs to be added so that this 'IsGeneric' decision, *whatever the heck that means*, can have a good home.  We'll see what that's all about in my next post.
