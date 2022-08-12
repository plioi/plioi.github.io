---
title: A Swiss Army Katana
layout: post
---
Before now, test methods for the [Fixie test framework](https://github.com/fixie/fixie/) had to have zero parameters. If your test method had a parameter, it would fail without being called. Fixie would have no idea what values to pass in. As of [Fixie 0.0.1.98](http://www.nuget.org/packages/Fixie/0.0.1.98), you can define your own conventions for parameterized tests. As a convention author, you decide what it means for a test to have parameters. For example, let's say you want your parameter values to come from attributes on the method, similar to [xUnit theories](http://stackoverflow.com/a/9110623):

```cs
public class CalculatorTests
{
    readonly Calculator calculator;

    public CalculatorTests()
    {
        calculator = new Calculator();
    }

    [Input(2, 3, 5)]
    [Input(3, 5, 8)]
    public void ShouldAdd(int a, int b, int expectedSum)
    {
        calculator.Add(a, b).ShouldEqual(expectedSum);
    }

    [Input(5, 3, 2)]
    [Input(8, 5, 3)]
    [Input(10, 5, 5)]
    public void ShouldSubtract(int a, int b, int expectedDifference)
    {
        calculator.Subtract(a, b).ShouldEqual(expectedDifference);
    }
}
```

Our intention is for these 2 test _methods_ to be treated as 5 test _cases_, producing 5 individual pass/fail results. Out of the box, Fixie has no idea what [Input] means. In order to let Fixie know about our intentions, we can define the attribute and a custom convention:

```cs
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class InputAttribute : Attribute
{
    public InputAttribute(params object[] parameters)
    {
        Parameters = parameters;
    }

    public object[] Parameters { get; private set; }
}
```

```cs
public class CustomConvention : Convention
{
    public CustomConvention()
    {
        Classes
            .NameEndsWith("Tests");

        Methods
            .Where(method => method.IsVoid());

        ClassExecution
            .CreateInstancePerTestClass();


        //This new hook lets you explain where
        //parameters should come from:

        Parameters(FromInputAttributes);
    }
    
    IEnumerable<object[]> FromInputAttributes(MethodInfo method)
    {
        var inputAttributes = method.GetCustomAttributes<InputAttribute>(true).ToArray();

        if (!inputAttributes.Any())
        {
            //No [Input] attributes?  Just call the
            //test method once with no arguments.

            yield return new object[] { };
        }
        else
        {
            //Call the test once for each [Input] attribute.

            foreach (var input in inputAttributes)
                yield return input.Parameters;
        }
    }
}
```

Your own convention wouldn't have to be attribute-based. All that Fixie cares about is that you provide it some `Func<MethodInfo, IEnumerable<object[]>>`. That's a mouthful, so let's break it down:

  1. Parameters(...) accepts a function that explains what inputs to use for any given test method.
  2. Your function is given the MethodInfo that describes a single test method.
  3. Your function yields any number of object arrays.
  4. Each object array that you yield represents a single call to the test method. If the method takes in 3 parameters, your arrays better have 3 values with corresponding types.

`Func<MethodInfo, IEnumerable<object[]>>` is a Swiss Army Katana. It's versatile, but sharp. It will enable us to do a wide variety of things, but it's easy to misuse. It represents exactly what the .NET reflection API needs in order to call the method, so no matter what sugar I layer on top of it, `Func<MethodInfo, IEnumerable<object[]>>` is the truth under the hood. After developing a few more examples, it'll be more clear how a few convenient overloads of the `Parameters()` method could make it easier to get things right in the most common situations.

In my next post, we'll take a little detour to see why such a small change was so hard to implement.