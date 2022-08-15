---
title: Fail Loudly
layout: post
---
I ended my last post, [Generating Test Cases at Runtime](https://patrick.lioi.net/2013/10/09/generating-test-cases-at-runtime/), with a pop quiz: "Can you spot the bug? It’s possible to write test methods that never get invoked." Today, let's cover the bug as well as its fix.

## Reproducing the Issue

At the end of the last post, we had a working test class and a convention which could match up a static IEnumerable<T> generator with a test accepting a T. Each value yielded by the IEnumerable<T> would be passed to the test as a single pass-or-fail-result. It worked just fine for that example: TestWithNoParameters() was called once, and TestWithInterestingParameter(Person) was called once for every Person yielded by the static Person-generating methods.

If we add a third test method, though, one whose incoming T type doesn't happen to match any input-generating static methods, something weird happens:

```cs
public class ParameterizedTests
{
    public void TestWithNoParameters()
    {
        Console.WriteLine("TestWithNoParameters");
        Console.WriteLine();
    }
 
    public void TestWithInterestingParameter(Person person)
    {
        Console.WriteLine("TestWithInterestingParameter");
        Console.WriteLine("    Person: " + person);
        Console.WriteLine();
    }

    //Something fishy about this new test!
    public void TestWithDateTimeParameterNeverGetsCalled(DateTime dateTime)
    {
        Console.WriteLine("This test method is never called, and has no effect on the test run's output!");
    }
 
    public static IEnumerable<Person> People()
    {
        yield return new Person { Name = "Arthur Vandelay", BirthDay = new DateTime(1980, 1, 1) };
        yield return new Person { Name = "Kel Varnsen", BirthDay = new DateTime(1978, 1, 1) };
    }
 
    public static IEnumerable<Person> MorePeople()
    {
        yield return new Person { Name = "Martin Van Nostrand", BirthDay = new DateTime(1995, 1, 1) };
        yield return new Person { Name = "H.E. Pennypacker", BirthDay = new DateTime(1920, 1, 1) };
    }
}
```

```
TestWithNoParameters

TestWithInterestingParameters
    Person: Arthur Vandelay, born on 1/1/1980 12:00:00 AM.

TestWithInterestingParameters
    Person: Kel Varnsen, born on 1/1/1978 12:00:00 AM.

TestWithInterestingParameters
    Person: Martin Van Nostrand, born on 1/1/1995 12:00:00 AM.

TestWithInterestingParameters
    Person: H.E. Pennypacker, born on 1/1/1920 12:00:00 AM.

5 passed, 0 failed, 0 skipped, took 0.20 seconds (Fixie 0.0.1.98).
```

That's terrible! Our new test doesn't pass, _and it doesn't even fail_. A test runner's prime directive is to provide reliable pass/fail results. Anything that is a test should have a result, and this new method is most certainly a test according to the convention's discovery rules:

```cs
Classes
    .NameEndsWith("Tests");

Methods
    .Where(method => method.IsVoid());
```

## The Diagnosis

Over my last few posts, we've dealt with Fixie's parameterized tests hook: a Func<MethodInfo, IEnumerable<object[]>>. When you want to define the meaning of a parameterized test method, you must yield an object[] once for every time you want the test method to be called. Until now, I've been assuming that the convention author's Func will always yield at least once. We even saw an attempt to do so in last week's convention:

```cs
Parameters(method =>
{
    //Attempt to find a perfect matching source for N calls.
    var parameters = method.GetParameters();

    if (parameters.Length == 1)
        return FindInputs(method.ReflectedType, parameters.Single().ParameterType);

    //No matching source method, so call it once with with zero parameters.
    return new[] { new object[] {} };
});
```

When a test method had no parameters at all, it would explicitly return a single empty object[], meaning "Call the test method once with no args." FindInputs(...), on the other hand, could still yield zero object[], and thus zero requests to call the method. Here's a bug that causes silent failures, and the worst failure is the one that keeps on happening without anyone knowing. We often hear the advice to "fail fast", but there's an implicit "...and fail as loudly as possible" in there, too.

## The Fix: Fail Loudly

In order to fail loudly, Fixie needed to [treat test methods with _unsatisfied_ parameters as _failures_](https://github.com/fixie/fixie/commit/006f3a74e0f47222e1e44b8f192d44d70172a788), with a failure message explaining that fact. A nice side effect is that it's easier for convention authors to do the right thing: yield when you have a meaningful input, don't when you don't, and let Fixie autofail any test method that still couldn't be called.

Our convention from last time gets simpler, now that we don't have a lurking edge case to care about, and the output now correctly complains about the new test method:

```cs
public class CustomConvention : Convention
{
    public CustomConvention()
    {
        Classes
            .NameEndsWith("Tests");

        Methods
            .Where(method => method.IsVoid());

        Parameters(FindInputs);
    }

    private static IEnumerable<object[]> FindInputs(MethodInfo method)
    {
        var parameters = method.GetParameters();

        if (parameters.Length == 1)
        {
            var testClass = method.ReflectedType;
            var parameterType = parameters.Single().ParameterType;

            var enumerableOfParameterType = typeof(IEnumerable<>).MakeGenericType(parameterType);

            var sources = testClass.GetMethods(BindingFlags.Static | BindingFlags.Public)
                                   .Where(m => !m.GetParameters().Any())
                                   .Where(m => m.ReturnType == enumerableOfParameterType)
                                   .ToArray();

            foreach (var source in sources)
                foreach (var input in (IEnumerable)source.Invoke(null, null))
                    yield return new[] { input };
        }
    }
}
```

```
TestWithNoParameters

TestWithInterestingParameter
    Person: Arthur Vandelay, born on 1/1/1980 12:00:00 AM.

TestWithInterestingParameter
    Person: Kel Varnsen, born on 1/1/1978 12:00:00 AM.

TestWithInterestingParameter
    Person: Martin Van Nostrand, born on 1/1/1995 12:00:00 AM.

TestWithInterestingParameter
    Person: H.E. Pennypacker, born on 1/1/1920 12:00:00 AM.

Test 'ParameterizedTests.TestWithDateTimeParameterNeverGetsCalled' failed: System.ArgumentException
    This parameterized test could not be executed, because no input values were available.
    at Fixie.UncallableParameterizedCase.Execute(Object instance)

5 passed, 1 failed, 0 skipped, took 0.29 seconds (Fixie 0.0.1.101).
```

The fix is available in [Fixie 0.0.1.101](http://www.nuget.org/packages/Fixie/0.0.1.101).