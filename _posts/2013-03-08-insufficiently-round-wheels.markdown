---
title: Insufficiently Round Wheels
layout: post
---

When Sharon Cichelli started the [Polyglot Programmers of Austin](http://austin.polyglotprogrammers.org), she created the meeting format around learning-by-teaching: one of the best ways to become an expert at something is to start teaching others about it.  Surely you'll make some glorious mistakes along the way, but you'll be motivated to constantly fill in the gaps of your knowledge in order to keep up with yourself and your audience.

My weekly blog posts over the past year and a half have served the purpose of learning-by-teaching, but the subject matter has always been random.  One thing that has made this effort difficult is that my current open source projects are a little esoteric.  It's too difficult to write an article of the form, "Here's a problem I ran into and here's how I solved it," as these projects require too much background explanation each time.

To motivate better learning-by-teaching, I'd like to take on a new experiment: developing a new open source project from scratch, documenting my approach here each week.  To keep the material focused and more generally useful, I'll be selecting a project that most software developers are at least familiar with.  I want to work on something immediately *recognizable*, so that we all know what success will look like at each step, yet complex enough that the engineering challenges will match those we all find on real world projects.

I'm going to reinvent a familiar wheel, but that's okay.  The more I look at this wheel, the more I think it really could benefit from being reinvented.

## Goals

The goals of this project are not limited to delivering the software.

My public goal for this series of articles is to help any developer feel confident that they, too, can start a new open source project.  We'll cover things like attacking the hard problems first, prototyping, build scripts, bootstrapping, dogfooding, and publishing your work for everyone to use.  Along the way, we'll surely run into some interesting engineering challenges, too.

My personal goals are twofold.  First, I want to become a better software developer, and that takes practice.  Second, I want to become better at helping others to become better software developers.

Documenting my progress as I go along is just an extension of the open source point of view.  Not only will the *source* be open source on GitHub, but the *process* will be open source here in these articles.

## The Wheel

In case I haven't been clear enough, I want to pick a problem that is *extremely* familar to software developers, but one that is nontrivial to implement.  Also, in order to add something new to the mix, it should be a problem whose state of the art has basically stagnated and could benefit from a refresh.

The insufficiently round wheel I'd like to reinvent is the unit test framework.

<blockquote>
**Seriously?** Yes.
**No, really.** Yes.
**Those have been around forever.** You could say the same thing about music or turducken, but we've only really started to get those right in the last few years.
**But Patrick, that is a sufficiently round wheel and you shouldn't be reinventing the wheel.  Do something cool instead!** First, all "reinventing the wheel" remarks shall be met with a friendly "That's kind of the point," followed by a not-so-friendly [EPIC EYE-ROLL](http://www.youtube.com/watch?v=fRiiiKA1xUI\#t=13s).  Second, I think we really can add something cool to the mix.

If you must, think of it as a big code review of the current state of the art, followed by some advice on building *any* old open source project you have in mind.
</blockquote>

## NUnit and xUnit: Constructive Criticism

I have happily used NUnit for several years on several projects.  I'm using it on my current project at Headspring.  It's excellent.  Use it.  I have happily used xUnit for several years on several projects.  I'm using it on my current open source projects.  It's excellent.  Use it.

Still, these are complex systems, so there is always room for constructive criticism.

**NUnit has a complex test fixture lifecycle.**  You can have code run before and after each fixture, and before and after each test case.  You specify these setup and teardown actions via \[Attributes\].  One instance of your fixture class is constructed and shared across the test cases, in order to support the fixture-level setup and teardown.

I've run into a lot of frustrating surprises with this lifecycle.  First, I couldn't tell you exactly how these attributes behave in the presence of inheritance (in fact, I believe the rules changed a few times throughout the history of NUnit).  Second, I have seen this behave differently across different *test runners*.  It's pretty jarring to find your SetUp methods blatantly ignored.  **The lifecycle is complex.  I lack control over it, and I lack insight into it when things go weird.**

xUnit was a reaction to NUnit, and it has a much simpler fixture lifecycle.  You get one instance of the fixture class *per test case*, a regular constructor is used for setup, a regular Dispose() is used for teardown, and if you really really want fixture state, you can still do that using an interface dedicated to that purpose.  I like the simplicity of using already-available language concepts like constructors and interfaces for these lifecycle steps.  I never get confused when my xUnit fixture classes take part in inheritance.

<blockquote>I find that when fixture-wide state is downplayed like it is in xUnit, *I don't miss it at all*.  When I use NUnit, on the other hand, I use \[TestFixtureSetUp\] all the time.  Once you open the door to a little complexity, vagabonds and feral cats are free to walk right in.</blockquote>

xUnit is meant to be more customizable with regard to how fixtures and test cases are discovered, but **the means of customization is opinionated**.  I've had some success customizing xUnit in the past, but I ran into two problems: I still had to put an attribute on each of my fixture classes, and the means of customization is an [ISP violation](http://www.headspring.com/patrick/low-ceremony-xunit/).

Okay, so it was a little hard to customize, buy why is it bad that I still had to put an attribute on my fixture classes?  I actually have a use case in mind for one of my other projects, in which some test fixtures would originate from folders of plain text files rather than C\# classes.  If my fixture isn't also a class, *where do I put xUnit's required class attribute*?  **I want to have total control over the means of discovering test fixtures and test cases.**

## Degrees of Freedom

Regardless of the test framework I use, I use the Should assertion library.  Its creator recognized that assertion libraries have no business being tied to a single test framework.  Once that concept was plucked out of the test framework's responsibilities, developers gained an extra degree of freedom: you can change your assertion library without having to change your test framework, and you can change your test framework without having to change your assertions.

<blockquote>I'm thinking about switching to Shoudly.  [Its error messages are amazing.](http://shouldly.github.com/)  How does that even *work*!?</blockquote>

Ensuring the freedom to make changes is vital to successful software development, which raises an important question: if assertion libraries have no business being a part of a test framework, and plucking them out gives us a new degree of freedom, then what *else* are test frameworks mistakenly doing which should *also* be plucked out?

## Fixie

Aside from providing reasonable defaults, I think that test frameworks have no business owning test fixture discovery, test case discovery, or the test fixture lifecycle.  The reasonable defaults should be enough for the 90% of use cases, but it should get completely the heck out of my way for the other 90%.

If we pluck those responsibilities out, what's left?  Test frameworks should only do the monotonous parts: orchestration of test execution, deferring to you for advice on discovery/lifecycle decisions, accumulating results, and reporting those results to whatever test runner is in play.  It should include a console test runner at a minimum, but test running should also be available as a library so that other runners like a GUI runner, ReSharper plugin, TestDriven.NET runner, or the like could be developed independently.

I'd like to announce [Fixie](https://github.com/plioi/fixie), a test framework with the goals of having low-ceremony defaults, and extra degrees of freedom around the test lifecycle.  We'll see this project grow here over the next few months.

<blockquote>A [fixie](http://en.wikipedia.org/wiki/Fixed-gear\_bicycle) is what you get when you start with a bike and remove everything that isn't a bike.  I want Fixie to be what's left when you start with a test framework and remove everything that isn't a test framework.</blockquote>

**Low-ceremony defaults:** No required attributes, no required inheritance.  A test fixture is a class in your test assembly whose name ends in Tests.  A test case is any public method in one of those classes.  I will likely include an answer to NUnit's [\[TestCaseSource\]](http://nunit.org/index.php?p=testCaseSource&r=2.6.2), but only if I can keep it similarly low-ceremony.  Like xUnit, you'll get one fixture instance per test case.

**Degrees of freedom:**  If the defaults aren't enough, I want you to be able to customize the test fixture and test case discovery steps, as well as customize the test fixture lifecycle itself.  Making this customization *easy* will be challenging, but fun.

Next week, we'll tackle a proof-of-concept for the highest-risk requirement: the must-have integration with TestDriven.NET.