---
title: "Whittling NUnit"
layout: post
---


Last week, we applied <a href="http://patrick.lioi.net/2013/01/03/code-whittling/">Code Whittling</a> to learn about <a href="http://patrick.lioi.net/2013/01/15/whittling-xunit/">xUnit</a>.  This week, we'll do the same for NUnit, and contrast the two testing frameworks.  NUnit has been around a long time, and has accumulated many features.  <a href="http://www.nunitlite.com/">NUnitLight</a> is a recent endeavor to make a smaller version of the core features.  Since my goal is to get to the core features anyway, I'll start with NUnitLight instead of the full NUnit.

## The Whittle

I started by grabbing a copy of the <a href="https://launchpad.net/nunitlite">NUnitLite source code on Launchpad</a>.

The first thing I noticed is that it has an atypical structure in Solution Explorer.  Project names include .NET version numbers (2.0, 3.5, 4.0), and these projects are placed within solution folders NET-2.0, NET-3.5, and NET-4.0.  Comparing this to the actual file structure on disk, it looks like the same code files are shared by each of these version-based projects.

Why have multiple projects containing the exact same files?  Viewing project properties, I see that each project has version-specific "conditional compilation symbols" like NET_2_0 or CLR_4_0 to distinguish them.  Within the code, there are lots of tests against these symbols:

```cs
#if CLR_2_0 || CLR_4_0
  //Code that should only exist for CLR version 2 and 4.
#endif
```

The developers were therefore able to use one set of files to create builds for each version they will run on, accounting for the differences with these preprocessor conditions.  That doesn't sound like fun, but keep in mind NUnit is ubiquitous, and you don't want to be surprised by incompatibility with your environment.

For my purposes, I picked a single version to focus on, and removed the other versions' projects from the solution (leaving the actual shared code files intact).

When I whittled down xUnit last week, one of the first steps was to remove its own tests, and I did the same here with NUnitLite.  Remember, this is supposed to be a destructive exercise.  The goal is to remove everything that is in the way of my understanding the fundamentals of the project: how tests are discovered, how tests are executed, and how test results are accumulated.  I was left with a single project in the solution: nunitlite-3.5.

Again, as with xUnit, I removed all assertion code, since I use the <a href="http://nuget.org/packages/Should">Should</a> assertion library instead.  This was actually a much larger undertaking than with xUnit.  NUnit has been around for a long time, and the assertion library grew while keeping backward-compatibility as a priority.  It looks like a significant amount of effort went into avoiding duplication of code in the *implementation* of assertions, even while the main end-user-facing assertion APIs gained support for different styles:

```
Assert.AreEqual(x, y); //That's a bit too verbose, eh?

Assert.That(y, Is.EqualTo(x)); //I stand corrected.
```

Behind this API lies a large family of classes called Constraints, which represent concepts like is-equal-to-x.  Both lines in the sample above would ultimately rely on the same constraint class to perform the comparison.

Next, I removed several attributes I never use, so I could instead focus on the main lifecycle attributes: [TestFixture], [TestFixtureSetUp], [TestFixtureTearDown], [SetUp], [TearDown], and [Test].

I made a mostly-random pass through the solution, removing things that were no longer reachable.  When interfaces eventually had a single implementation, I'd phase out the interface.  Once it got down to a manageable size, I tweaked and simplified some of the types and methods as the mood struck me.  Now that I was familiar with the overall solution structure, I took a closer look at the test discovery and execution code.

## Results - Test Discovery

Let's consider NUnitLiteTestAssemblyRunner to be our entry point.  Given an Assembly (a project containing tests), we want to discover and run all the tests.  I whittled it down to this:

```cs
public class NUnitLiteTestAssemblyRunner
{
  public ITestResult Run(ITestFilter filter, Assembly assembly)
  {
    var builder = new NUnitLiteTestAssemblyBuilder();

    var loadedTest = builder.Build(assembly);

    var workItem = WorkItem.CreateWorkItem(loadedTest, filter);
    workItem.Execute();

    while (workItem.State != WorkItemState.Complete)
        System.Threading.Thread.Sleep(5);

    return workItem.Result;
  }
}
```

It defers to NUnitLiteTestAssemblyBuilder to create *something* representing the tests to run, creates a work item to process the tests, kicks off the work item, and waits for the work item to finish.

NUnitLiteTestAssemblyBuilder's job is to traverse the assembly, discover the test fixtures and their tests, and then return a TestSuite summarizing all the work that is to be done.  It does so using reflection, not surprisingly with a very similar implementation as xUnit.  I whittled NUnitLiteTestAssemblyBuilder's pivotal method down to:

```cs
var testTypes = assembly
                  .GetTypes()
                  .Where(TestFixtureBuilder.CanBuildFrom)
                  .Select(TestFixtureBuilder.BuildFrom);
```

The CanBuildFrom(Type) method returns true if the Type has a [TestFixture] attribute or if any of its methods have a [Test] attribute.  The BuildFrom method then builds a test fixture description by reflecting on the Type's methods, converting each test method into a Test:

```cs
var tests = methods.Select(method => BuildTestCase(method, fixture))
                   .Where(test => test != null);
```

To build a single test case, it performs a similar CanBuildFrom/BuildFrom pair as we saw for the fixture as a whole.  In this case, CanBuildFrom tests whether the given method has a [Test] attribute, and BuildFrom constructs a new TestMethod instance, which simply describes the test to be executed.

All of the above lets us go from a given Assembly to a tree of all the tests that need to be executed.

## Results - Test Execution

To actually run the tests, recall that we hand that work off to something called a WorkItem.  The work item was initialized with the TestSuite (aka the test fixture) to run, and this in turn creates yet more work items, one per test.  Running one of these per-test work items takes us to TestMethodCommand.Execute(...), which I whittled down to:

```cs
public TestResult Execute(TestExecutionContext context)
{
  Reflect.InvokeMethod(testMethod.Method, context.TestObject);
  context.CurrentResult.SetResult(ResultState.Success);
  return context.CurrentResult;
}
```

Again, much like xUnit, we use reflection to call the test method.  If an exception is thrown, that will cause the test to fail.  If no exception is thrown, we happily reach the next line and set the result to Success.

## Contrasting xUnit with NUnitLight

NUnitLight and xUnit accomplish virtually the same thing, and from a very high level they do it in the same way:

1. Provide an assertion API so that failed assertions present themselves as exceptions thrown from test methods.
2. Use reflection to traverse the test Assembly, building up a tree that describes what fixtures are to be executed, and which tests appear on each fixture.
3. Walk through those trees, using reflection to invoke each test method.
4. If that invocation didn't throw an exception, consider the test passed.


Despite this similarity, their implementations differ a great deal.  I thought xUnit was big, but even NUnit*Light* seems quite a bit larger.  It was really easy to remove xUnit's assertion code, and very tough to remove NUnitLight's.  I'm not saying "ease of destruction" should be anywhere on a developer's radar when writing an API, but it does speak to the history of the two projects.  As I wrote earlier, NUnit's history involves more than one user-facing API for writing assertions, and they needed to avoid duplicating implementation details; naturally the design of assertions here is more complex than with xUnit.  I started pulling the assertion thread in NUnitLight, and quickly wound up with a long series of dependencies to remove before it would compile again.

Before looking at either of these projects, I figured that after removing the assertion code, there would be very little left, but I was wrong.  As with anything that gets used by such a large audience, the devil is in the details.  Projects this popular have to handle many environments and situations that no single developer ever encounters on their own.

Although I doubt I could make a meaningfully-large contribution to these projects right away, I'm now much closer to that level of knowledge.  Passively reading through the code would not have been as effective as destroying them from the outside in.
