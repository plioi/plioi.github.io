---
title: "The Sincerest Form of Flattery"
layout: post
---


Last week, we saw how to define <a href="http://patrick.lioi.net/2013/05/30/fixies-life-bicycle/">an NUnit-imitating convention</a> with the Fixie test framework: when the custom Convention class was present in our test project, the default rules for finding and running tests were replaced, allowing us to write test classes with a familiar NUnit class lifecycle.

This week, we'll see how to customize Fixie to imitate the xUnit lifecycle.

> Today's code samples work against <a href="http://nuget.org/packages/Fixie/0.0.1.56">Fixie 0.0.1.56</a>. The customization API is in its infancy, and is likely to change in the coming weeks.

## Review: The NUnit Lifecycle

With NUnit, one instance of your [TestFixture] class is constructed, and that instance is shared across all of that class's [Test] methods.  Test discovery is based on the presence of these attributes.  You can identify methods as [SetUp] and [TearDown] in order to run common code before and after each individual test.  You can also identify methods as [TestFixtureSetUp] and [TestFixtureTearDown], in order to perform class-wide initialization and cleanup steps at the start and end of the class's lifespan.  You can use fields in the class to hold state that lives across all of the tests.  At the end, if the class is IDisposable, the Dispose() method is called once.

## The xUnit Lifecycle

xUnit is based on NUnit, but they both have different rules about what a test is, and how to run a test once it is found.  xUnit test methods are marked with a [Fact] attribute, and test classes don't need any attribute since it is implied by the presence of [Fact]s.  More importantly, xUnit test classes are constructed again and again, once for each [Fact].

Frequent reconstruction of the test class has a few consequences from the point of view of NUnit users.  

The first consequence affects how to go about implementing basic setup and teardown logic.  Construction, fixture-level setup, and test-level setup suddenly collapse into one concept, so all of your setup is simply placed in the constructor.  Disposal, fixture-level teardown, and test-level teardown likewise collapse into one concept, so all of your teardown logic goes in the Dispose() method.

The second consequence of this frequent reconstruction is that test class fields are forgotten from one test to the next, which raises the obvious question, what if I *just plain want* some state to live across all the tests?  I may have an integration test, for instance, with database setup steps that are costly in time.  I don't want to be forced to redo that setup for each test simply to satisfy the strong opinions of a test framework!

Thankfully, xUnit gives us an escape hatch in the form of `IUseFixture<T>`.  Your test class can implement this interface for some type T, and xUnit will in turn construct one shared instance of that T.  After reconstructing the test class and before running the next [Fact] method, xUnit injects that T into your test class instance.  When all the [Facts] are done, xUnit will likewise dispose of the T, giving you something like NUnit's [TestFixtureTearDown].

That's a mouthful.  Let's see a sample xUnit test fixture exercising the whole test lifecycle:

```cs
using System;
using Should;

namespace Fixie.Samples.xUnitStyle
{
    public class LifecycleTests
            : IUseFixture<FixtureData>, IUseFixture<DisposableFixtureData>, IDisposable
    {
        FixtureData fixtureData;
        DisposableFixtureData disposableFixtureData;

        public LifecycleTests()
        {
            Console.WriteLine("LifecycleTests.Constructor");
        }

        public void SetFixture(FixtureData data)
        {
            Console.WriteLine("LifecycleTests.SetFixture(FixtureData)");

            fixtureData = data;
        }

        public void SetFixture(DisposableFixtureData data)
        {
            Console.WriteLine("LifecycleTests.SetFixture(DisposableFixtureData)");

            disposableFixtureData = data;
        }

        [Fact]
        public void FirstTest()
        {
            Console.WriteLine("First Test passes!");
        }

        [Fact]
        public void SecondTest()
        {
            Console.WriteLine("Second Test fails!");
            1.ShouldEqual(2);
        }

        public void Dispose()
        {
            Console.WriteLine("LifecycleTests.Dispose");
        }
    }

    public class DisposableFixtureData : IDisposable
    {
        public DisposableFixtureData()
        {
            Console.WriteLine("DisposableFixtureData.Constructor");
        }

        public void Dispose()
        {
            Console.WriteLine("DisposableFixtureData.Dipose");
        }
    }

    public class FixtureData
    {
        public FixtureData()
        {
            Console.WriteLine("FixtureData.Constructor");
        }
    }
}
```

## Customizing Fixie to Mimic xUnit

In order to mimic xUnit, we first have to tell Fixie how to find [Fact] methods.  Then, we'll need to tell it to find all of the `IUseFixture<T>` declarations to construct the shared instances of whatever type was provided as the "T".  After that prep work, we can start the actual test lifecycle: for each [Fact] method, we want to construct an instance of the test class, inject the T objects into that instance, call the [Fact], and call Dispose().  After performing that cycle for each [Fact], we need to clean up the shared instances of the Ts.

Here's the Fixie Convention class which accomplishes this lifecycle.  The details have been omitted to focus on the Convention API, but the <a href="https://github.com/fixie/fixie/blob/7fa012d1c63016b7b2e6061fa91cca90fbbc3326/src/Fixie.Samples/xUnitStyle/CustomConvention.cs">xUnit-style CustomConvention class</a> can be found on GitHub under the Samples namespace:

```cs
namespace Fixie.Samples.xUnitStyle
{
    public class CustomConvention : Convention
    {
        readonly Dictionary<MethodInfo, object> fixtures = new Dictionary<MethodInfo, object>();

        public CustomConvention()
        {
            Fixtures
                .Where(HasAnyFactMethods);

            Cases
                .HasOrInherits<FactAttribute>();

            FixtureExecution
                .CreateInstancePerCase()
                .SetUpTearDown(PrepareFixtureData, DisposeFixtureData);

            InstanceExecution
                .SetUpTearDown(InjectFixtureData, (fixtureClass, instance) => new ExceptionList());
        }

        ...
    }
}
```

The FixtureExecution section says what should be done with each test fixture class as a whole: we want one instance per test case, we want the whole process to be preceded by a call to PrepareFixtureData, and we want the whole process to be concluded by a call to DisposeFixtureData.

The InstanceExecution section says what should be done immediately after construction and immediately before disposal of the test class.  Test runs should be preceded by a call to InjectFixtureData so that the shared "T" objects can be available to the test.

> Note how awkward it is to say that InstanceExecution has a SetUp action but no relevant TearDown action.  On TearDown, we "do nothing" by returning an empty list of errors.  That's clearly a wart on this API; one I intend to improve upon soon.

The convention class itself has some state, a dictionary which holds onto the shared T objects.  PrepareFixtureData populates the dictionary by finding `IUseFixture<T>` declarations.  InjectFixtureData reads from that dictionary in order to call the test class's SetFixture(...) methods.  DisposeFixtureData disposes and removes items from the dictionary.

When we run our sample test class in the presence of this custom convention class, we get the desired output:

```
FixtureData.Constructor
DisposableFixtureData.Constructor
LifecycleTests.Constructor
LifecycleTests.SetFixture(FixtureData)
LifecycleTests.SetFixture(DisposableFixtureData)
First Test passes!
LifecycleTests.Dispose
LifecycleTests.Constructor
LifecycleTests.SetFixture(FixtureData)
LifecycleTests.SetFixture(DisposableFixtureData)
Second Test fails!
LifecycleTests.Dispose
DisposableFixtureData.Dipose
```

## Mimicry as Motivation

Fixie's customization features are intended to set it apart from other test frameworks, so why spend all this time using it only to mimic those other frameworks?  By using two familiar yet dramatically different test lifecycles as a target, I've been able to discover and expose the "hooks" they both have in common.  I've discovered that I needed to be able to switch between two modes of construction: one instance per test class vs. one instance per test case method.  I've also discovered that I needed *three* levels of setup/teardown hooks, where I was originally guessing that two would be enough: 1) the start and end of each test *method*, 2) the start and end of each test class *instance*, and 3) the start and end of each test *class*.

I selected NUnit and xUnit mimicry deliberately as a first goal along the development of Fixie's customization API.  If I couldn't do what these frameworks do, there'd be no point.  Now that I've been able to mimic them, I can start to use the customization API to do new, more interesting things.  Next week, we'll try to come up with a convention that is similar to NUnit, but addresses some complexity issues I dislike facing in my NUnit tests.
