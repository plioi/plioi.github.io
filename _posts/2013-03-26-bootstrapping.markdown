---
title: Bootstrapping
layout: post
---

Last week, I covered [the first several commits to Fixie](http://www.headspring.com/patrick/socks-then-shoes/), resulting in a reliable build script.  This week, we'll see how I've "bootstrapped" Fixie to the point where it can run all of its own tests.

Bootstrapping comes from an old saying.  To "pull oneself up by one's bootstraps", though literally absurd, is to improve without outside assistance.  For software, bootstrapping involves getting a new system off the ground by first leveraging the less-desirable system you had to start with.

I've seen this approach take a few forms.  Bootstrapping is the solution to two categories of software problems: The Grand Rewrite, and The Curiously Self-Sufficient System

## The Grand Rewrite

Grand rewrite projects are very tempting.  Perhaps you work on a large project for a long time, and you get to a point where adding new features becomes more and more difficult.  In other cases, your well-designed system simply no longer matches the new needs of your growing business.

In a perfect world, you could keep what you have and incrementally improve it to meet new demands, but sometimes that simply isn't practical.  Your hand may be forced by the need to escape vendor lock-in, or you may find it increasingly difficult to hire people experienced with an aging platform.

When deciding to go forward with a grand rewrite, recognize that it is an inherently-risky endeavor.  You're going to face all the same design tradeoffs and bugs you faced before, plus all the new ones you introduce this time around.  If anyone says they'll simply "get it right this time," don't listen.  Rather, your team must actively mitigate the inherent risks.

The worst-case scenario involves building the new system in full, off to the side, before ever going to production.  After eventually reaching "feature parity plus a few", you'll just turn off the old system, turn on the new system, and bask in the glory of your dizzyingly-late and overpriced system.  This approach maximizes the risks involved: you could spend a great deal of money chasing a large and moving target before reaping *any* benefits from it.

**The best-case scenario involves bootstrapping the new system**, leveraging what you have in order to get *some* of the new system up and running in production as soon as possible.  Maybe you build a little of the new system, altering the legacy system to defer to the new one for a single feature, and then repeat the process, phasing out the old system one piece at a time.  Maybe the legacy system never invokes the new one, but the new one begins its life by implementing the most important new features against the legacy database.  Whatever the specifics, you want to use as much of the legacy investment as possible in order to get new value in production as fast as possible.

Suddenly, you're not the expensive team that's been toiling away for years without providing value.  You're the team that's constantly pushing valuable progress to production.

## The Curiously Self-Sufficient System

The second class of bootstrapping projects is a little more interesting from a developer's point of view, in that their implementation can be a little mind-bending: a system appears to be built upon itself.  The best example of this seemingly-impossible state is when a compiler for a language is writting in that language.

The current version of the C# compiler was implemented in another language, but the *next* version of the C# compiler is being written in C#.  I don't think I'd call this example bootstrapping, however, since C# has already been established as a full-size language for some time now.  Bootstrapping is more of a technique to get a new language up and running quickly so that it can spend most of its lifetime "self-hosted".

Many languages' first implementations are written in a preexisting language like C.  After the first version is working, the second version can be written in the new language, compiled with the first version.  Each new version is compiled with the previous version.

<blockquote>Even Pascal's *first* compiler was written in Pascal.  Through the power of imagination and bravado, the code for the compiler was written on the assumption that a compiler would someday exist for it.  Then, it was translated backwards down to a simpler language that *did* have a compiler, in order to produce the first working Pascal compiler.  There were, *ahem*, no unit tests.</blockquote>

To create one of these curiously self-sufficient, bootstrapped systems, **identify some fundamentally important subset of your goal**, the subset that would be just useful enough to use for ongoing development, and implement that using the legacy system.  You may even deliberately limit yourself to use only some of the legacy system's features along the way, partly to make it easier to switch over to the new one, and partly to keep yourself honest about sticking to the fundamentally important subset.

The point here is that some problems lend themselves to being curiously self-sufficient, and when solving such a problem you can keep scope creep in check while simultaneously escaping your project's predecessor.

## Bootstrapping Fixie

[Fixie](https://github.com/plioi/fixie), my test framework project, is Curiously Self-Sufficient.

I always want to write code with support from automated tests, even on this project, so I had to start implementing it with some *other* test framework in place.  Now that I have implemented enough features for it to run its own tests, I can simply use it to test-drive all the remaining features.

There are a few benefits to wearing the Bootstrapping Hat on this project.

First, being able to run my tests using both xUnit and Fixie in early development allows me to compare their output, which has been helpful for discovering requirements that were not already on my radar.

Second, it has given me a very reliable check against scope creep.  I have several big features planned, and I occasionally found myself tempted to include a little too much in this first pass.  When in doubt about including some feature F, I could always ask myself the same question:

<blockquote>Is feature F necessary to run any of the xUnit tests I've had to write so far?  If not, I don't need feature F yet.</blockquote>

I limited myself to only use those xUnit features that were absolutely necessary to drive each new feature, so I didn't have to worry about chasing a moving target.

By leveraging the existing system (xUnit) in this way, I've been able to get real value from the new system (Fixie) very quickly.  Fixie can run all of its own test cases, even though those test cases were written using xUnit.  I produce equivalent output when tests pass and when tests fail.  I'm actively writing new test cases as a Fixie user, not an xUnit user, so I consider it successfully bootstrapped.

## Fixie's Initial Implementation

By approaching this task with a bootstrapping mindset, I've successfully gotten a useful-enough test framework up and running in a very short time.  It's not fancy enough to write home about, but that wasn't the goal in this phase of development.  Let's see what this minimal test framework looks like.

First, [Fixie's build script](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/default.ps1) runs all the tests using Fixie and then runs all the tests using xUnit.  In order to compare their output in the case of failing tests, only xUnit failures actually cause the build to fail.  Fixie test failures are output, but don't prevent the build script from progressing to the xUnit run.

The [default convention](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/DefaultConvention.cs) (and currently the only convention), describes how to tell whether a class is a test fixture, and whether a method is a test case.  A class is a test fixture if its name ends with "Tests" and it has a default constructor; a method is a test case if it's a public instance void method with zero parameters:

{% gist 5242692 %}

This convention helps to reach out and find all the fixture classes and test case methods in your test assembly, so it can construct Fixture and Case objects describing the work to be performed.  A [Fixture](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/Fixture.cs) is a named executable thing, a [Case](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/Case.cs) is a named executable thing, and they both rely on a [Listener](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/Listener.cs) to report test failures:

{% gist 5242698 %}

For this bootstrapping phase, all fixtures correspond with classes, so the only implementation of Fixture is [ClassFixture](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/ClassFixture.cs).  ClassFixture takes one of the Types discovered by the DefaultConvention, and owns the test fixture lifecycle in its Execute method:

{% gist 5242707 %}

Note the elaborate try/catch block.  Activator.CreateInstance(fixtureClass) calls your test fixture's default constructor via reflection.  If your test fixture constructor throws an exception, we perceive that here as a TargetInvocationException that *wraps* the original exception.  We don't want to report that wrapper to the user, otherwise every single test failure will hide the original with the unhelpful message, "Exception has been thrown by the target of an invocation."  We unpack the wrapped original exception and report *that* to the listener.

For this bootstrapping phase, all test cases correspond with methods, so the only implementation of Case is [MethodCase](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/MethodCase.cs).  MethodCase takes one of the MethodInfos discovered by the default Convention, and owns the execution of that method with a similar exception handler:

{% gist 5242711 %}

That's all the real work.  The remaining classes include [Suite](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/Suite.cs), which loops through all the Fixtures and asks them to run themselves, [ConsoleListener](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie.Console/ConsoleListener.cs), which is a Listener that outputs failures to the console, and [Program](https://github.com/plioi/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie.Console/Program.cs), whose Main method builds up and executes a Suite with a ConsoleListener for a given Assembly.

By approaching this effort from a bootstrapping point of view, I now have a test framework powerful enough to drive the rest of its own features.  I've kept scope creep in check while laying a reasonable foundation, and even in the early stages I was able to have meaningful code coverage via xUnit.  Now that I'm about to embark on the more significant features, I already have a meaningful set of test cases to rely on.