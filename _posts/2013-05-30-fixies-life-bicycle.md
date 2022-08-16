---
title: "Fixie's Life Bicycle"
layout: post
---


Last week, we saw how the <a href="https://github.com/fixie/fixie">Fixie test framework</a> gives you control over <a href="http://patrick.lioi.net/2013/05/22/test-discovery/">test discovery</a>. This week, we'll see my first (admittedly rough) attempt at similarly giving you control over test *execution*. Let's start with a quick review of last week's test discovery feature, and then extend the example to demonstrate Fixie's treatment of test execution.

## Test Discovery (Again)

By default, Fixie uses a reasonable rule of thumb to determine which of your classes are test classes, and which of your methods are test methods. The default rules are implemented like so:

```cs
public class DefaultConvention : Convention
{
    public DefaultConvention()
    {
        Fixtures
            .NameEndsWith("Tests");

        Cases
            .Where(method => method.Void() || method.Async())
            .ZeroParameters();
    }
}
```

Test classes are those whose name ends with "Tests".  Test case methods are those with zero parameters, declared to be either `void` or `async Task`.  In other words, if it looks like a test, it's a test.

When you wish to stray from these defaults, though, you can provide your own *convention* class: tell Fixie what your test classes and test methods *look like*, and it will gladly use your rule of thumb instead of the default. Last week, we introduced NUnit-style attributes and provided our own custom convention describing the treatment of those attributes:

```cs
public class CustomConvention : Convention
{
    public CustomConvention()
    {
        Fixtures
            .HasOrInherits<TestFixtureAttribute>();

        Cases
            .HasOrInherits<TestAttribute>();
    }
}
```

By stating that test fixtures are marked with [TestFixture] and test cases are marked with [Test], Fixie starts to use NUnit-style test discovery behavior.

## Test Discovery is Only Half the Battle

Implicit in the default convention is the notion that you will get a new instance of the test class *for each test method*. That rule matches xUnit, but differs from NUnit, in which you get one instance of the test class *shared* across all the test methods in that class. Using our custom convention, we're not quite behaving like NUnit.  If you wanted to do NUnit-style [TestFixtureSetUp] and [TestFixtureTearDown], you'd be surprised! Using the above custom convention, consider the following test fixture and its output under Fixie:

```cs
using System;
using Should;

namespace Fixie.Samples.NUnitStyle
{
    [TestFixture]
    public class LifecycleTests : IDisposable
    {
        public LifecycleTests()
        {
            Console.WriteLine("Constructor");
        }
 
        [TestFixtureSetUp]
        public void TestClassSetUp()
        {
            Console.WriteLine("TestFixtureSetup");
        }
 
        [TestFixtureTearDown]
        public void TestClassTearDown()
        {
            Console.WriteLine("TestFixtureTearDown");
        }
 
        [SetUp]
        public void TestSetUp()
        {
            Console.WriteLine("SetUp");
        }
 
        [TearDown]
        public void TestTearDown()
        {
            Console.WriteLine("TearDown");
        }
 
        [Test]
        public void FirstTest()
        {
            Console.WriteLine("First Test passes!");
        }
 
        [Test]
        public void SecondTest()
        {
            Console.WriteLine("Second Test fails!");
            1.ShouldEqual(2);
        }
 
        public void Dispose()
        {
            Console.WriteLine("Dispose");
        }
    }
}
```

```
Constructor
First Test passes!
Dispose
Constructor
Second Test fails!
Dispose
```

That's not at all like NUnit! Thankfully, our custom convention was honored so that only FirstTest() and SecondTest() are considered to be tests. Unlike NUnit, though, Fixie has completely neglected the per-test [SetUp]/[TearDown] and per-class [TestFixtureSetUp]/[TestFixtureTearDown].  On top of that, it has constructed a fresh instance of the class twice instead of once.

**Our custom convention is allowing us to stray from the defaults for test *discovery*, but so far we're still using Fixie's default test *execution* rules.**

## Customizing Test Execution

> The functionality covered in this section is in its infancy and is likely to change in the short term, but serves to demonstrate the kind of customization I am shooting for.

Fixie's Samples project contains a more useful <a href="https://github.com/fixie/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie.Samples/NUnitStyle/CustomConvention.cs">NUnit look-alike convention</a>:

```cs
public class CustomConvention : Convention
{
    public CustomConvention()
    {
        Fixtures
            .HasOrInherits<TestFixtureAttribute>();

        Cases
            .HasOrInherits<TestAttribute>();

        FixtureExecutionBehavior = new CreateInstancePerFixture();

        InstanceExecutionBehavior =
            InstanceExecutionBehavior
                .Wrap<TestFixtureSetUpAttribute, TestFixtureTearDownAttribute>();

        CaseExecutionBehavior =
            CaseExecutionBehavior
                .Wrap<SetUpAttribute, TearDownAttribute>();
    }
}
```

Here, we see three new sections. First, we say that for each test fixture, create an instance per fixture class instead of creating an instance per test case. Second, for each test class instance, wrap the built-in behavior with calls to the [TestFixtureSetUp] and [TestFixtureTearDown] methods. Lastly, for each test case method, wrap the built-in behavior with calls to the [SetUp] and [TearDown] methods.

Armed with this new convention, running the sample test class confirms that we're now following the NUnit test fixture lifecycle:

```
Constructor
TestFixtureSetup
SetUp
First Test passes!
TearDown
SetUp
Second Test fails!
TearDown
TestFixtureTearDown
Dispose
```

The FixtureExecutionBehavior you select in your convention is the key driving force affecting how your test classes will be executed. There are two built-in behaviors: <a href="https://github.com/fixie/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie/Behaviors/CreateInstancePerCase.cs">CreateInstancePerCase</a>, and <a href="https://github.com/fixie/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie/Behaviors/CreateInstancePerFixture.cs">CreateInstancePerFixture</a>.

These two classes give Fixie a two-mode test lifecycle. A life-*bi*cycle if you will, <a href="http://en.wikipedia.org/wiki/Fixed-gear_bicycle">finally justifying the name beyond any doubt</a>.
