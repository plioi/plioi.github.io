---
title: "Test Discovery"
layout: post
---


Over the last few weeks, I've implemented some customization features in <a href="https://github.com/fixie/fixie">the Fixie test framework</a>. The first of these features is now available. Today, we'll see this feature in action. **We're going to tell Fixie what our tests *look like*, and Fixie will then find them and run them.**

> Today's code samples work against <a href="http://nuget.org/packages/Fixie/0.0.1.49">Fixie 0.0.1.49</a>. The customization API is in its infancy, and is likely to change as I address more involved features in the coming weeks.

## The Default Convention

If you've used NUnit before, you know that you have to mark your test classes with [TestFixture] and your test methods with [Test] in order for NUnit to know that those are your tests.  NUnit uses the presence of those attributes to "discover" your tests before it can run them. NUnit is therefore opinionated about test discovery.

If you've used xUnit before, you know that you have to mark your test methods with [Fact] in order for xUnit to know that those are your tests. xUnit uses the presence of that attribute to "discover" your tests before it can run them. xUnit is therefore opinionated about test discovery.  (We've seen that <a href="http://patrick.lioi.net/2012/09/13/low-ceremony-xunit/">xUnit is a little more flexible in this regard</a>, but it's still pretty opinionated about what a test is.)

**Fixie is not opinionated about test discovery.** It has a simple default, but allows you replace that default with your own conventions. By default, Fixie will look for test classes by a naming convention: if a class in your test project has a name ending with "Tests", then it is a test class. After finding these classes, it will then look for test methods as any public instance void-or-async method with zero parameters. In other words, if it looks like a test, walks like a test, and quacks like a test, Fixie will assume it's a <del>duck</del> test by default.

In my implementation, these rules are defined by <a href="https://github.com/fixie/fixie/blob/075d41822e6bee18624bd8329343d68e31d58c54/src/Fixie/Conventions/DefaultConvention.cs">DefaultConvention</a>:

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

Let's see this convention in action. This demo assumes you have <a href="http://testdriven.net/">TestDriven.NET</a> installed. I have set up CTRL-T to run whatever test method or test class my cursor is sitting on.

Create a new Solution in Visual Studio (I called mine "DiscoveryConventions"), and install <a href="http://nuget.org/packages/Fixie/0.0.1.49">Fixie 0.0.1.49</a> in the Package Manager Console:

```
PM>  Install-Package Fixie -Version 0.0.1.49
```

Fixie deliberately has no assertion statements of its own, so install <a href="http://nuget.org/packages/Should">Should</a> too:

```
PM>  Install-Package Should
```

Add a Calculator class. We're going to write some tests for this in a moment:

```cs
namespace DiscoveryConventions
{
    public class Calculator
    {
        public int Add(int a, int b)
        {
            return a + b;
        }

        public int Subtract(int a, int b)
        {
            return a - b;
        }
    }
}
```

Add a test class using the default convention:

```cs
using Should;

namespace DiscoveryConventions
{
    public class CalculatorTests
    {
        readonly Calculator calculator;

        public CalculatorTests()
        {
            calculator = new Calculator();
        }

        public void ShouldAdd()
        {
            calculator.Add(2, 3).ShouldEqual(5);
        }

        public void ShouldSubtract()
        {
            calculator.Subtract(5, 3).ShouldEqual(2);
        }
    }
}
```

Place your cursor in either test method and hit your TestDriven.NET shortcut (for me, that's CRTL-T). You'll see TestDriven.NET ran that test with output like so:

```
------ Test started: Assembly: DiscoveryConventions.dll ------

1 passed, 0 failed, 0 skipped, took 0.30 seconds (Fixie 0.0.1.49).
```

Place your cursor *between* the ShouldAdd and ShouldSubtract methods and run TestDriven.NET again. You'll see it ran all the tests in the class with output like so:

```
------ Test started: Assembly: DiscoveryConventions.dll ------

2 passed, 0 failed, 0 skipped, took 0.14 seconds (Fixie 0.0.1.49).
```

So far, so boring.  This is a similar experience to using NUnit and xUnit. The only thing I've saved you is a few keystrokes for the attributes.

## Custom Conventions

What if you don't like the default convention?  What if you have a different naming convention for your test classes and test methods?  What if you like the way attributes jump out at you? Thankfully, you can set aside the default convention and substitute your own. If you place your own implementation of Convention in your test assembly, Fixie will discover and use that one *instead* of DefaultConvention.

> Let's try this customization out by first making it work more like NUnit, and then making it work more like xUnit. Lastly, we'll see how Fixie accomplishes this behavior.

## Immitating NUnit

Rename CalculatorTests to CalculatorTestFixture. Since the class no longer ends with "Tests", it no longer matches the default convention. If you try to run the tests again, TestDriven.NET *will* run it, but it will say "(Ad hoc)" instead of "(Fixie 0.0.1.49)", which means that TestDriven.NET has no idea that this class is a test class anymore, and it just called the method as best as it could. That's nice, but it won't be enough when we get into things like test classes that have SetUps and TearDowns in the weeks ahead, so today we need to ensure that even when we stray from the default convention, TestDriven.NET should still be able to know that it's looking at a Fixie test class!

Let's define some NUnit-style attributes:

```cs
using System;

namespace DiscoveryConventions
{
    [AttributeUsage(AttributeTargets.Class)]
    public class TestFixtureAttribute : Attribute { }

    [AttributeUsage(AttributeTargets.Method)]
    public class TestAttribute : Attribute { }
}
```

Apply these to CalculatorTestFixture as you would with NUnit tests:

```cs
using Should;

namespace DiscoveryConventions
{
    [TestFixture]
    public class CalculatorTestFixture
    {
        readonly Calculator calculator;

        public CalculatorTestFixture()
        {
            calculator = new Calculator();
        }

        [Test]
        public void ShouldAdd()
        {
            calculator.Add(2, 3).ShouldEqual(5);
        }

        [Test]
        public void ShouldSubtract()
        {
            calculator.Subtract(5, 3).ShouldEqual(2);
        }
    }
}
```

Trying to run these tests, we see that TestDriven.NET is *still* using the lame "(Ad hoc)" test runner.  TestDriven.NET is still unaware that it is looking at a test class! **Teach it to care about these attributes by adding a new Convention subclass to the project:**

```cs
using Fixie.Conventions;

namespace DiscoveryConventions
{
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
}
```

Here, we are saying that our test fixture classes are those which have [TestFixture] attributes, and our test case methods are those which have [Test] attributes. Running our tests again, we see that TestDriven.NET is finally aware that CalculatorTestFixture is a Fixie test class, so it was able to use Fixie again to actually run the tests:

```
------ Test started: Assembly: DiscoveryConventions.dll ------

2 passed, 0 failed, 0 skipped, took 0.15 seconds (Fixie 0.0.1.49).
```

**We have changed the way that Fixie discovers our tests by telling it what our tests look like.**

## Immitating xUnit

xUnit works a little differently from NUnit. You don't have to put an attribute on the test class, but you do have to put a [Fact] on each test method. Any class that happens to have a [Fact] method is assumed to be a test class.

Delete the NUnit-style TestFixtureAttribute and TestAttribute classes, and replace them with a [Fact] attribute:

```cs
using System;

namespace DiscoveryConventions
{
    [AttributeUsage(AttributeTargets.Method)]
    public class FactAttribute : Attribute { }
}
```

Update CalculatorTestFixture to use xUnit-style test decoration:

```cs
using Should;

namespace DiscoveryConventions
{
    public class CalculatorTestFixture
    {
        readonly Calculator calculator;

        public CalculatorTestFixture()
        {
            calculator = new Calculator();
        }

        [Fact]
        public void ShouldAdd()
        {
            calculator.Add(2, 3).ShouldEqual(5);
        }

        [Fact]
        public void ShouldSubtract()
        {
            calculator.Subtract(5, 3).ShouldEqual(2);
        }
    }
}
```

Update the CustomConvention to use xUnit-style rules:

```cs
using System;
using System.Linq;
using Fixie.Conventions;

namespace DiscoveryConventions
{
    public class CustomConvention : Convention
    {
        readonly MethodFilter factMethods = new MethodFilter().HasOrInherits<FactAttribute>();

        public CustomConvention()
        {
            Fixtures
                .Where(HasAnyFactMethods);

            Cases
                .HasOrInherits<FactAttribute>();
        }

        bool HasAnyFactMethods(Type type)
        {
            return factMethods.Filter(type).Any();
        }
    }
}
```

Here, we are saying that our test fixture classes are those which have any methods that have [Fact] attributes, and our test case methods are those which have [Fact] attributes. Running our tests again, we see that TestDriven.NET is again aware that CalculatorTestFixture is a Fixie test class, so it was able to use Fixie again to actually run the tests:

```
------ Test started: Assembly: DiscoveryConventions.dll ------

2 passed, 0 failed, 0 skipped, took 0.14 seconds (Fixie 0.0.1.49).
```

**We again changed the way that Fixie discovers our tests by telling it what our tests look like.**

## Neat Trick. What's the Point?

NUnit, xUnit, and other test frameworks are very opinionated about two major concepts: how to discover your test classes/methods, and how to go about executing them. Today, we see that Fixie can at least give you an extra degree of freedom around test discovery. You're free to use whatever logic you want to decide whether a class is a test class, and whether a method is a test method. (We'll see how Fixie addresses the second part, test *execution*, in the coming weeks.)

Even if all this accomplished was fewer keystrokes, or an easier path to migrate from another framework *to* Fixie, I'd consider it a net gain. However, I'm already benefiting from the flexibility in more ways. When using Fixie to test Fixie, I use the default convention with a twist: when I need to prove that Fixie will do the right thing in the event of a test *failure*, I want to ask some *other* "phony" test class to run. If the phony test class fails in the way I expect, my real tests pass. Only the real tests need to pass for my build to succeed. The phony tests are identified with the <a href="https://github.com/fixie/fixie/blob/075d41822e6bee18624bd8329343d68e31d58c54/src/Fixie/Conventions/SelfTestConvention.cs">SelfTestConvention</a>:

```cs
public class SelfTestConvention : Convention
{
    public SelfTestConvention()
    {
        Fixtures
            .Where(fixtureClass => fixtureClass.IsNestedPrivate)
            .NameEndsWith("Fixture");

        Cases
            .Where(method => method.Void() || method.Async())
            .ZeroParameters();
    }
}
```

I create phony test classes as nested, private classes with names ending in "Fixture". The wrapper classes follow the DefaultConvention and must pass, while the must-pass tests do their work by asking the SelfTestConvention to run a phony test class. Without these conventions, it would be too hard for me to test that I can properly handle *failing* tests.

## How Does it Work?

We've seen that Fixie somehow knows how to look for Convention classes. After finding them, it must be able to use them in some way, so Fixie must somehow construct instances of your Conventions, too. The answer is <a href="http://msdn.microsoft.com/en-us/library/ms173183(v=vs.110).aspx">reflection</a>: code that searches and uses other assemblies at runtime.

When I ask Fixie to run all the tests in the test assembly, it needs to reach out and find all the Convention classes and then construct them for use. Where it *used* to just construct a `new DefaultConvention()` every time, my Runner class *now* does the following:

```cs
static Convention[] GetConventions(Assembly assembly)
{
    var customConventions = assembly
        .GetTypes()
        .Where(t => t.IsSubclassOf(typeof(Convention)))
        .Select(t => (Convention)Activator.CreateInstance(t))
        .ToArray();

    if (customConventions.Any())
        return customConventions;

    return new[] { (Convention) new DefaultConvention() };
}
```

Here, we search the test assembly for types that are subclasses of Convention, and create an instance of each.  If we didn't find any, we'll assume the DefaultConvention.

By reaching out into your code with reflection, Fixie enables you to tell it what your test classes and test methods look like.
