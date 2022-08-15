---
title: AutoFixie
layout: post
---
Last week, we saw how the Fixie test framework deals with [test method parameters](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/). Your custom convention calls a Parameters(...) hook, supplying an explanation of _where_ parameters should come from and _how many times_ the test method should be called. The example convention was a little underwhelming, since parameters had to be explicitly listed within [Input] attributes. Today, we'll get our parameter values from somewhere more interesting, [AutoFixture](https://github.com/AutoFixture/AutoFixture).

Parameterized tests are useful when a single test method needs to run many times against a large set of independent inputs. However, it's perfectly reasonable for parameterized tests to be called a _single_ time, and in this case the purpose of using parameters is to hide away some otherwise-distracting instantiation code. Your test can focus on the actual system-under-test. AutoFixture's a great example for single-call parameterized test methods, since it's purpose is to automate boring class-population code you'd rather not clutter your tests with.

Let's start with a simple Person class:
  
```cs
public class Person
{
    public string Name { get; set; }
    public DateTime BirthDay { get; set; }

    public override string ToString()
    {
        return String.Format("{0}, born on {1}.", Name, BirthDay);
    }
}
```

Next, I have a test class with three tests. Two of the tests have parameters. Since I'm just investigating Fixie's ability to pass meaningful values to these tests, they write the actual incoming values to the console instead of making assertions:
  
```cs
public class ParameterizedTests
{
    public void TestWithNoParameters()
    {
        Console.WriteLine("TestWithNoParameters");
        Console.WriteLine();
    }

    public void TestWithTrivialParameters(int i, bool b)
    {
        Console.WriteLine("TestWithTrivialParameters");
        Console.WriteLine("    i: " + i);
        Console.WriteLine("    b: " + b);
        Console.WriteLine();
    }

    public void TestWithInterestingParameters(Person personA, Person personB)
    {
        Console.WriteLine("TestWithInterestingParameters");
        Console.WriteLine("    Person A: " + personA);
        Console.WriteLine("    Person B: " + personB);
        Console.WriteLine();
    }
}
```

I haven't told Fixie what it means for a test to have parameters, so when we run our test under the default convention we see some failures:
  
```
------ Test started: Assembly: AutoFixtureDemo.dll ------

TestWithNoParameters

Test 'ParameterizedTests.TestWithTrivialParameters' failed: System.Reflection.TargetParameterCountException
  Parameter count mismatch.
  at System.Reflection.RuntimeMethodInfo.InvokeArgumentsCheck(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
  at System.Reflection.RuntimeMethodInfo.Invoke(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
  at Fixie.Case.Execute(Object instance)

Test 'ParameterizedTests.TestWithInterestingParameters' failed: System.Reflection.TargetParameterCountException
  Parameter count mismatch.
  at System.Reflection.RuntimeMethodInfo.InvokeArgumentsCheck(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
  at System.Reflection.RuntimeMethodInfo.Invoke(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture)
  at Fixie.Case.Execute(Object instance)

1 passed, 2 failed, 0 skipped, took 0.25 seconds (Fixie 0.0.1.98).
```

Today, I'd like parameters to come from AutoFixture, similar to [AutoFixture's support for xUnit Theories](http://blog.ploeh.dk/2010/10/08/AutoDataTheorieswithAutoFixture/). I needed to define an extension method for AutoFixture first, though, because normal AutoFixture usage involves knowing ahead of time what type to instantiate. Instead, we want to be able to ask AutoFixture to instantiate any arbitrary Type:
  
```cs
public static class AutoFixtureExtensions
{
    //Normal AutoFixture usage is to write:
    //
    //  var thing = autoFixture.Create<MyClass>();
    //
    //If all you have is an arbitrary Type object, though,
    //this extension method lets you write:
    //
    //  var thing = autoFixture.Create(type);
    public static object Create(this ISpecimenBuilder builder, Type type)
    {
        return new SpecimenContext(builder).Resolve(type);
    }
}
```

Finally, I define a custom convention which takes advantage of the Parameters(...) hook introduced last week:
  
```cs
using System;
using System.Linq;
using Fixie;
using Fixie.Conventions;

public class CustomConvention : Convention
{
    readonly Ploeh.AutoFixture.Fixture autoFixture = new Ploeh.AutoFixture.Fixture();

    public CustomConvention()
    {
        Classes
            .NameEndsWith("Tests");

        Methods
            .Where(method => method.IsVoid());

        Parameters(type => autoFixture.Create(type));
    }

    private void Parameters(Func<Type, object> parameterByType)
    {
        Parameters(method =>
        {
            var parameterTypes = method.GetParameters().Select(x => x.ParameterType);

            var parameterValues = parameterTypes.Select(parameterByType).ToArray();

            return new[] { parameterValues };
        });
    }
}
```

As we saw last week, the inherited Parameters(...) method wants you to provide a method that takes in a MethodInfo and returns an IEnumerable<object[]>, since in general it needs to support any number of calls to a given test method. For today's AutoFixture convention, though, we really just want to call each parameterized test **once**, and we want parameters to be based simply on the parameter types. Therefore, this convention includes a convenience overload for Parameters(...) which accepts a Func<Type, object>. This overload finds a single value for each parameter, based on its type, and returns a single-item array to provoke a single call to each test method. If it proves useful in general, I'll promote this overload to be accessible from any convention.

Running the tests again, we see that each test is successfully passed values generated by AutoFixture:
  
```
------ Test started: Assembly: AutoFixtureDemo.dll ------

TestWithNoParameters

TestWithTrivialParameters
    i: 60
    b: True

TestWithInterestingParameters
    Person A: Named5bf97cd-456e-488e-b75c-67c03b10c8f2, born on 8/22/2013 10:09:20 PM.
    Person B: Namec8c74d02-4211-419f-aa09-0ca469271986, born on 2/11/2014 6:29:26 AM.


3 passed, 0 failed, 0 skipped, took 0.26 seconds (Fixie 0.0.1.98).
```