---
title: Generating Test Cases at Runtime
layout: post
---
Last time, we saw how Fixie can [integrate with AutoFixture](https://patrick.lioi.net/2013/10/05/autofixie/). That was a situation in which parameterized tests were meant to be called once. They were parameterized because the _producer_ of the inputs was interesting while the _count_ was uninteresting.

Before that, we saw how a convention could instead cause parameterized test methods to be called _multiple_ times, [using attributes as a source of multiple inputs.](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/) That example was a little stifling: you have to know at compile time how many scenarios you are testing, and you can only specify compile-time constants in attributes.

Today, we'll see an example of parameterized tests which are called multiple times, using values _generated_ at runtime. Consider a test class with two test methods and some static Person-generator methods:

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

Without a custom convention, Fixie fails to invoke the parameterized test:

```
TestWithNoParameters

Test 'ParameterizedTests.TestWithInterestingParameter' failed: System.Reflection.TargetParameterCountException
  Parameter count mismatch.
	at System.Reflection.RuntimeMethodInfo.InvokeArgumentsCheck(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
	at System.Reflection.RuntimeMethodInfo.Invoke(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
	at Fixie.Case.Execute(Object instance)

1 passed, 1 failed, 0 skipped, took 0.19 seconds (Fixie 0.0.1.98).
```

Let's define a custom convention which considers single-argument test methods. For a test method that takes in some type T, it will find all the static methods returning IEnumerable<T>, call them all, and pass in the many T objects into the test method one at a time:

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using Fixie;
using Fixie.Conventions;

public class CustomConvention : Convention
{
    public CustomConvention()
    {
        Classes
            .NameEndsWith("Tests");

        Methods
            .Where(method => method.IsVoid());

        Parameters(method =>
        {
            //Attempt to find a perfect matching source for N calls.
            var parameters = method.GetParameters();

            if (parameters.Length == 1)
                return FindInputs(method.ReflectedType, parameters.Single().ParameterType);

            //No matching source method, so call it once with with zero parameters.
            return new[] { new object[] {} };
        });
    }

    private static IEnumerable<object[]> FindInputs(Type testClass, Type parameterType)
    {
        var enumerableOfParameterType = typeof (IEnumerable<>).MakeGenericType(parameterType);

        var sources = testClass.GetMethods(BindingFlags.Static | BindingFlags.Public)
                               .Where(m => !m.GetParameters().Any())
                               .Where(m => m.ReturnType == enumerableOfParameterType)
                               .ToArray();

        foreach (var source in sources)
            foreach (var input in (IEnumerable) source.Invoke(null, null))
                yield return new[] {input};
    }
}
```

Now, we see that our two test methods produce 5 individual test cases.

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

> Oh dear. **Can you spot the bug?** It's possible to write test methods that never get invoked. In our next episode, we'll cover the bug as well as an improvement in Fixie that will prevent such subtle surprises while simplifying parameterized test conventions.