---
title: "Quashing Incidental Complexity"
layout: post
---


This post is the conclusion to a 7-part series on identifying and resolving incidental complexity in your software projects.  It assumes you've already read the earlier posts (linked in the sections below).

The problems I've faced can crop up in any project.  When a concept or responsibility gets applied to the wrong abstraction, we subconsciously start to add workarounds just to make things work at all.  These workarounds take up space, physically as excess lines of code and mentally as distractions from your true intent.  Workarounds beget workarounds, and the problem feeds itself until you think, "What a mess!  I need to rewrite it all!"

By placing responsibilities in their proper home, you can combat this feedback loop and keep your code clean.

## A Troubled Project

In <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Essential vs Incidental Complexity</a>, I admitted that my pet project, a compiler for an experimental language called <a href="https://github.com/plioi/rook">Rook</a>, was starting to run into problems.

In a healthy project, it can actually get easier to add new features the further along you go, as you discover and flesh out abstractions that help rather than hinder progress.  In Rook, it was getting harder and harder to add new features.  This difficulty was a sign that I was working with the wrong abstractions.

Two classes in particular were starting to grow wildly complex: Scope (a dictionary-like type that describes which identifiers are visible from the point of view of a line of input source code), and TypeChecker (a recursive process which determines the type of all expressions).

I'd like to think I'm pretty good at writing software, but here I am admitting I created something too complex for its own good.  **This can happen to anyone**.  The important thing to know is that you can always take steps to turn things around.

> It's tempting to chuck everything and embark on a rewrite, but I've done that before (with this project!) and I'd rather fix what I have over the course of a few *weeks* than rewrite it and find myself in the same spot in a few *years*.

This kind of problem has a direct connection to incidental complexity.  If your abstractions are working against you, you spend more lines of code serving them than they spend serving you.  It starts to get harder and harder to see the intent behind a given method because half of the characters in it amount to boilerplate. 

As I said, this can happen to anyone.  The solution is to identify which abstractions are faulty, and adjust them in small and safe increments until they start working for you again.  

Faulty abstractions can take many forms:

## Unnecessary Abstractions

In <a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">Identifying Incidental Complexity</a>, `TypeChecked<T>` was an abstraction that complicated things without adding anything of value.  Its existence required many methods in TypeChecker to waste a lot of time and concentration to recombine very small arrays of error messages, completely distracting the reader from the actual intent of the methods.  It ultimately gave me no benefit, and removing it helped to clean up a lot of ugly code.

## Abstractions with Multiple Personalities

In <a href="http://patrick.lioi.net/2012/07/29/scope-creep/">Scope Creep</a>, we saw that the Scope abstraction was right in its inception but had gotten extremely complicated over time.

One class absorbed many responsibilities as I began to use it in more and more scenarios.  In fact, I was using one Scope class to represent several different *kinds* of scope (global, local, etc).  The Scope class had multiple personalities, and I saw that splitting such a class into several classes with a common interface would make it easier to see the actual domain concepts I was trying to represent.

Specifically, I used inheritance and lots of `override` methods to accomplish this improvement.  I don't enjoy doing that, but implementation inhertiance in this case was a *tool* to nudge things in the right direction in small steps while solidifying what separate classes I needed.

## Brittle Abstractions

In <a href="http://patrick.lioi.net/2012/08/04/spaghetti-alla-code/">Spaghetti alla Code</a>, we saw how implementation inheritance can motivate some horribly convoluted code that just *happens* to work.  Scope was separated *physically* into different classes now, but the method implementations were all tied up in knots.

With lots of `virtual/override/base` keywords, you have to read *all* of the code to understand *any* of the code.  Making changes here without unintended side effects becomes too difficult, making the abstraction brittle in the face of inevitable change.

> Implementation inheritance is like writing in assembly language: it *works*, but good luck ever *editing* it with confidence.

The solution here was to favor composition over inheritance.  Each individual class can be understood by only looking at that class.  The new implementations read a lot more like an English description of the domain being modeled.

## Anemic Abstractions

In <a href="http://patrick.lioi.net/2012/08/11/primitive-obsession/">Primitive Obsession</a>, I had an abstraction that existed in name only: TypeVariable was a simple wrapper for an int.  I was so obsessed that the TypeVariable class was "just a glorified int" that it never occurred to me to put more smarts into it.

Not realizing that this class was a good home for some important concepts, I had to throw those concepts *anywhere* that would work, and that just happened to be the unfortunate Scope class mentioned before (as if it didn't have enough to worry about).

> Scope was a bit of an Idea Sponge at this point.

The solution was to recognize that the TypeVariable abstraction was incomplete, and lots of things got simpler once an appropriate responsibility moved into the TypeVariable class.

## Complected Abstractions

In <a href="http://patrick.lioi.net/2012/08/18/detect-reflect-decomplect/">Detect, Reflect, Decomplect</a>, we saw that when two separate ideas get complected (braided) together, we have to start writing extra code to work around that complexity.

I was twisting up success and failure results into a single return value, forcing me to place special meaning on *null*s, and to include excessive early return statements throughout many methods.  These extra return statements set the code in intellectual concrete, severely limiting my ability to apply any cleanup refactorings.

By untwisting the concepts of success and failure in this class, and by avoiding null with the Null Object pattern, the troubled TypeChecker class finally got *linear* enough to have a hope of ever getting cleaned up.

## Code is Too Malleable to be Intractable

When you find the complexity of your projects growing out of control, don't lose hope!  Take a step back, identify which abstractions you're doing *extra* work *for*, and nudge them in small and safe steps until they better-represent the Truth of your domain.

You don't even have to put forward progress on hold; these kinds of improvements can be a small part of your daily activities.

Make your life easier and your project healthier by growing and cultivating abstractions that let you write *less and less* code.
