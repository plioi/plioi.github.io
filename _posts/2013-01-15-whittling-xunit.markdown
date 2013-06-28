---
title: Whittling xUnit
layout: post
---

Last week, we saw how <a href="http://www.headspring.com/patrick/code-whittling/">Code Whittling</a> lets you learn about an unfamiliar open-source project, using <a href="http://www.headspring.com/patrick/whittling-rhino-licensing/">Rhino Licensing</a> as an example.  This week, we'll walk through the process I took when whittling xUnit.  xUnit is a unit testing framework similar to NUnit.  I wanted to get a feel for how a test framework really works: how it traverses your test project to discover test cases, how it executes individual test cases, how it tallies results, and how it integrates with the TestDriven.net plugin for Visual Studio.

<h2>The Whittle</h2>

I started by grabbing a copy of the <a href="http://xunit.codeplex.com/SourceControl/BrowseLatest">xUnit source code on CodePlex</a>.

Not surprisingly, this project has a lot of test coverage.  Since code whittling is decidedly <em>not</em> about safe forward development, and since unit tests pose an obstacle to quickly whittling a project down to its essentials, the first thing I removed were all the unit tests.  It's OK, we're not going to make any permanent changes.  This is an exercise in quick discovery, not slow-and-steady accuracy.

I tend to use a console test runner and a TestDriven.Net test runner, but none of the other runners.  My next victims were the projects for the runners I don't happen to use: xunit.gui, xunit.runner.visualstudio, and xunit.runner.msbuild.

Since I use the <a href="http://nuget.org/packages/Should">Should</a> assertion library, I had no need for any of xUnit's own assertion code, allowing me to remove a huge chunk of code I had no interest in.  I deleted the <a href="http://xunit.codeplex.com/SourceControl/changeset/view/2e806844c3c1#src/xunit/Assert.cs">Assert</a> class, its usages, and the many associated exception classes.

I generally don't use the [Theory] attribute in my xUnit tests, so I removed that and its usages next.  Lots of support classes for [Theory] were no longer in my way.

I'm more interested in how results are tallied and output to the console, and less interested in any XSLT reporting of those results, so I removed all the XSLT-dependent code.

Then I removed a long series of features I rarely or never use: IUseFixture, skipped tests, [AutoRollback], [AssumeIdentity], [FreezeClock], [Trace], ...

<strong>Eventually, I goofed up.</strong>  It turns out there is more multithreading code and more dynamic construction (in which a string containing the name of a class is used to instantiate that class) in xUnit than I expected.  I actually expected little or no threading/dynamic stuff, but I ended up removing too much in one step, losing something fundamental in the code path between Main() and actual invocation of a test.

<blockquote>Lesson learned: Even when creating a throwaway clone of a project, just so I can violently destroy it, commit frequently with useful comments.  If you remove too much, you can see where you went wrong and rewind.</blockquote>

I found that I couldn't whittle this down quite as far as I did with Rhino Licensing.  Even with the less-used features taken out, what I was left with involved a lot of classes deferring to classes which defer to classes which defer to static helper methods.  I could have taken the time to refactor this away and boil it all down to the fundamental logic I was searching for, but I was starting to get comfortable with the class naming conventions and could find my way through it clearly enough.

I got a good, though incomplete, understanding of xUnit, and going back to the original code is no longer the daunting task it was before.

<h2>Results</h2>

Test execution is broken down into two phases.

For <strong>Phase 1</strong>, it builds up a tree of objects representing the test methods in the test assemblies: a TestAssembly has TestClass objects, each of which has TestMethod objects.  These are discovered and constructed via reflection.  The class EnumerateTests uses reflection to discover all the types in a given Assembly, and ultimately defers to TypeUtility.GetTestMethods(...):

[gist id=4438396]

Plain old reflection walks through all the methods in the test assembly, picking out the ones that are marked as tests with the [Fact] attribute.

For <strong>Phase 2</strong>, it traverses the tree of tests to execute them and accumulate results.  <a href="http://xunit.codeplex.com/SourceControl/changeset/view/2e806844c3c1#src/xunit/Sdk/Commands/ClassCommands/TestClassCommandRunner.cs">TestClassCommandRunner</a> has the main execution logic.  For a given class, it processes the list of method in random order.  For each method, xUnit gets one or more ITestCommands to execute.  Each test command is invoked, giving a MethodResult which gets included in the overall ClassResult.

A single test command to execute, for a test identified with the [Fact] attribute, looks like this:

[gist id=4438431]

A single test is executed here via Invoke(...), which is also a part of .NET reflection.  xUnit is just asking the .NET framework to call our test method for us.  Any exceptions bubble up and cause the test to fail, including exceptions thrown by Assert methods.  If no exceptions were thrown, it returns the new PassedResult representing success.  Results are hierarchical, just like the original tree of tests: AssemblyResult, ClassResult, FailedResult, and PassedResult allow us to track results at every level that a test runner may need to report on.

<strong>Lastly, TestDriven.NET integration:</strong> TestDriven.NET requires you to implement the following interface.  Depending on how the user invokes TD.Net, one of these three methods will be called:

[gist id=4438582]

xUnit's TdNetRunner implements this by ultimately deferring to the same test execution code that the console runner uses.

I was a little surprised by the size of the project, especially the use of multithreading.  A large number of features added up to a lot of things for me to remove before I could really see the fundamental code path.  This makes me curious about how NUnit compares, so next week we'll see what we can learn by whittling down NUnit.