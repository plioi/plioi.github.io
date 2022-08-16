---
title: "\"Just\" is a Four Letter Word"
layout: post
---


I'm often guilty of this myself, but I cringe whenever I hear a software developer say that in order to implement a feature, they "Just" have to do x, y, and z.  The reality is that even on healthy projects, you will face at least a little more complexity than could be anticipated in advance.  This complexity makes time-based estimates risky, especially on seemingly-small features.  This week, I was particularly guilty of declaring to myself that a feature would take "Just a few lines of code".

## The Feature

Working on the <a href="https://github.com/fixie/fixie">Fixie test framework</a> this week, I pulled the next task from my backlog.  It read:

> Honor Dispose() when present.

When a test fixture class happens to implement IDisposable, the test framework should treat Dispose() as special.  After constructing your fixture and calling its test methods, and before it discards the fixture instance, it should be sure to call Dispose().  For example, the xUnit test framework uses Dispose() in the same way that NUnit uses [TearDown] methods.  In both of those frameworks, you have a chance to perform cleanup after tests execute, and I wanted Fixie to support Dispose() too.

## Initial Analysis

To get a better idea of what I would have to do, I took a look at the way C# `using` blocks work.  When you write a block like this:

```cs
using (WebClient client = new WebClient())
{
    // do work with client
}
```

...the compiler will rewrite it before actually compiling anything:

```cs
{
  WebClient client = new WebClient()
  try
  {
    // do work with client
  }
  finally
  {
    if (client != null)
      ((IDisposable)client).Dispose();
  }
}
```

To satisfy the requirement, "Honor Dipose() when present," I ***just*** had to wrap my test-running code in a similar try/finally block.  Easy as pie.  It should take about 4 minutes, mostly just to write its acceptance test.

"***Just*** 4 minutes" quickly turned into 4 hours.

## The Easy Part

The <a href="https://github.com/fixie/fixie/commit/16f079b08131026e75d5ae5075dfbf5ec7e1df1b">primary commit for this feature</a> is exactly what I expected.  My acceptance test for this feature involved a sample fixture that implemented IDisposable along with two tests, one that passes and one that fails.  My real test fixture would run that sample test fixture, inspecting the results.  This pattern of having a real fixture wrap a private sample fixture allows me to have sample fixtures with failing tests. Only failures in the outer real fixture cause my build to fail:

```cs
public class ClassFixtureTests
{
    ...

    public void ShouldDisposeFixtureInstancesWhenDisposable()
    {
        var listener = new StubListener();
        var fixtureClass = typeof(DisposableSampleFixture);
        var fixture = new ClassFixture(fixtureClass, defaultConvention);

        DisposableSampleFixture.ConstructionCount = 0;
        DisposableSampleFixture.DisposalCount = 0;

        fixture.Execute(listener);

        listener.ShouldHaveEntries(
            "FailingCase failed: Failing Case",
            "PassingCase passed.");

        DisposableSampleFixture.ConstructionCount.ShouldEqual(2);
        DisposableSampleFixture.DisposalCount.ShouldEqual(2);
    }

    class DisposableSampleFixture : IDisposable
    {
        public static int ConstructionCount { get; set; }
        public static int DisposalCount { get; set; }
        bool disposed;

        public DisposableSampleFixture()
        {
            ConstructionCount++;
        }

        public void Dispose()
        {
            if (disposed)
                throw new ShouldBeUnreachableException();

            DisposalCount++;
            disposed = true;
        }

        public void FailingCase()
        {
            throw new Exception("Failing Case");
        }

        public void PassingCase()
        {
        }
    }
}
```

The primary commit's fix involved wrapping test execution in a `try/finally`:

```cs
try
{
    @case.Execute(listener);
}
finally
{
    var disposable = Instance as IDisposable;
    if (disposable != null)
        disposable.Dispose();
}
```

## The First Four Monkey Wrenches

That wasn't actually the first commit for this feature.  I tried that all first, but the outer test fixture would fail.  Within the sample fixture, Dispose() was being called at the end of test execution, as expected, but Dispose() was *also* being called as a test method too!  Output suggested that my 2-test fixture had 3 tests, and Dispose() was being called 4 times.  Yeesh.

To resolve that issue, I ***just*** had to omit Dispose() from being treated as a test method.  I introduced a helper method to test whether a given method is Dispose().

```cs
public static bool IsDispose(this MethodInfo method)
{
    return method.Name == "Dispose";
}
```

Oops. Not every method with that name is the Dispose() method.  I really had to look for the right method *signature*:

```cs
public static bool IsDispose(this MethodInfo method)
{
    return method.Name == "Dispose" && method.Void() && method.GetParameters().Length == 0;
}
```

Oops. Not every method with that signature is really IDisposable.Dispose():

```cs
public static bool IsDispose(this MethodInfo method)
{
    var hasDisposeSignature = method.Name == "Dispose" && method.Void() && method.GetParameters().Length == 0;

    if (!hasDisposeSignature)
        return false;

    return method.DeclaredType.GetInterfaces().Any(type => type == typeof(IDisposable));
}
```

Oops.  DeclaredType isn't always the right type to inspect for IDisposable.  Consider this situation:

```cs
abstract class HasDisposeButNotIDisposable
{
    public void Dispose() { }
}

class DisposableTestFixture : HasDisposeButNotIDisposable, IDisposable
{
    //Tests go here.
}
```

In this case, the DeclaredType for the Dispose() method is HasDisposeButNotIDisposable, which doesn't implement IDisposable.  When Fixie tried to run tests in a class like DisposableTestFixture, it *still* treated Dispose() as a test case.  I had to replace DeclaredType with ReflectedType:

```cs
public static bool IsDispose(this MethodInfo method)
{
    var hasDisposeSignature = method.Name == "Dispose" && method.Void() && method.GetParameters().Length == 0;

    if (!hasDisposeSignature)
        return false;

    return method.ReflectedType.GetInterfaces().Any(type => type == typeof(IDisposable));
}
```

Finally, I could <a href="https://github.com/fixie/fixie/commit/3f9dc52a3e4570c7baa197773ae8a1983abc50f8">use that helper method to exclude IDisposable.Dispose()</a> from being treated as a test case.  Running the sample fixture produced one pass and one expected failure, and Dispose was called the right number of times.

All done.

## The Plot Thickens

Wait.  What if someone's test fixture has a Dispose() that throws exceptions?  Just like an NUnit [TearDown], we want exceptions here to cause the corresponding tests to fail, and we want the disposal exception to be included in the output.  I ***just*** have to wrap the disposal in a try/catch and emit a failure when Dispose() throws, like I already do when a test method throws:

```cs
try
{
    var disposable = fixture.Instance as IDisposable;
    if (disposable != null)
        disposable.Dispose();
}
catch (Exception ex)
{
    listener.CaseFailed(this, ex);
}
```

When a test method passes but Dispose() throws, this code does the right thing by treating the test as a failure and presenting the exception to the user.  When a test method *fails* and Dipose() throws, it would incorrectly report 2 test failures (one reported by the test method execution, and one reported by this catch block).  Instead, I want to treat it as one test failure, while reporting both exceptions to the user as the *reasons* the single test failed.

To address that detail, I had to dramatically restructure the test execution code so that it would accumulate potentially-many exceptions throughout the test lifecycle.  Only at the end of the lifecycle would it decide whether the test passed or failed.  If any exceptions had been accumulated, the test would fail and the reasons would list all the exceptions.

> I'm glad I ran into this problem now, because it will surely come up again when I address other test lifecycle methods, corresponding with NUnit concepts like [TestFixtureSetUp], [TestFixtureTearDown], [SetUp], and [TearDown]. The new code makes it easy to have multiple steps in the test lifecycle, all possibly contributing reasons for the test to fail.

## 4 Hours Later

Finally, the original feature, "Honor Dipose() when present," was implemented, and it ***just*** took 4 hours.  The next time you catch yourself saying "Just", take a moment to think critically about what all you've hidden behind that word.  Any given feature may be easy to describe to a user, and the most likely use case may very well be easy to implement, but the devil's in the details.
