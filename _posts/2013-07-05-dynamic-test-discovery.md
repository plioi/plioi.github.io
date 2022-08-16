---
title: "Dynamic Test Discovery"
layout: post
---


Recently I received feature requests for <a href="http://fixie.github.io/">Fixie</a> that initially proved difficult. I was tempted to engage in a vague refactoring effort until it became more clear how to proceed. Of course, giving in to that temptation would have proven wasteful. I might have achieved some degree of subjective cleanliness while getting no closer to actually solving the problems I faced.

Instead, I moved Fixie's design forward by selecting one of the feature requests as a specific goal. I let the feature tell me where the current design was lacking, which suggested a simple improvement. The more often I focus on concrete features, the faster Fixie's design will approach what it needs to be. You don't stop feature development in order to engage in open-ended refactoring. Rather, refactoring is what we should do *throughout* feature development.

The first of these initially-tricky features was to imitate <a href="http://www.nunit.org/index.php?p=explicit&r=2.6.2">NUnit's [Explicit] attribute</a>. Let's say you want to mark one of your tests so that it will only run when it is explicitly selected to run. Under normal runs of your test suite, these "explicit" tests should be ignored. When you run one specifically, though, *only* that test is run.

Supporting explicit tests is a matter of customizing test *discovery*. All the examples of test discovery I've written about so far have been *static*: a method is determined to be a test or not based on the method definition alone. Explicit tests, on the other hand, demonstrate the need for *dynamic* test discovery rules: sometimes an explicit method is a test, sometimes it isn't, and the decision has to be made based on the runtime context we're executing under.

## A Convention for Explicit Tests

> Today's code sample works against <a href="http://nuget.org/packages/Fixie/0.0.1.65">Fixie 0.0.1.65</a>. The customization API is in its infancy, and is likely to change in the coming weeks.

Consider a test class containing an explicit test:

```cs
using System;
using Should;

namespace Tests
{
    public class CalculatorTests
    {
        private readonly Calculator calculator;

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

        [Explicit]
        public void ExplicitTest()
        {
            Console.WriteLine("ExplicitTest was invoked explicitly!");
        }
    }
}
```

Out of the box, Fixie has no idea what [Explicit] means. To support it, we have to define the attribute as well as a custom convention to use it:

```cs
using System;
using Fixie;
using Fixie.Conventions;

namespace Tests
{
    [AttributeUsage(AttributeTargets.Method, Inherited = false)]
    public class ExplicitAttribute : Attribute
    {
    }

    public class CustomConvention : Convention
    {
        public CustomConvention(RunContext runContext)
        {
            Classes
                .NameEndsWith("Tests");

            Cases
                .Where(method => method.Void())
                .ZeroParameters()
                .Where(method =>
                {
                    var isMarkedExplicit = method.Has<ExplicitAttribute>();

                    return !isMarkedExplicit || runContext.TargetMember == method;
                });
        }
    }
}
```

In this example, we're identifying explicit tests with an attribute. We could have just as easily used some other rule for determining which tests are explicit. Perhaps we could see whether the method in question lives in a special *.Explicit namespace, or follows some test naming convention. Attributes seem fine here, though, since explicit tests are rare and should stand out as having a special purpose.

When Fixie searches for test methods, it will use the conditions we specify in the convention. In this case, we inspect each method for an [Explicit] attribute *and we consider the context in which we're running*. We are saying that a method in a test class is a test method if a) it is not [Explicit] or b) it is the sole target of execution.

## Driving Design with Features

When I first attempted to write an example of explicit tests, I didn't yet have access to any such runtime context. I couldn't phrase the if-statement because that line of code had no information about how the test execution was kicked off. This stumbling block motivated a simple design change: if a convention accepts a RunContext in its constructor, Fixie will pass in that context. Conventions are no longer limited to static decision making.

I'm glad I didn't give in to the temptation to refactor without a specific feature in mind. Only when I picked a feature and seriously thought about what was missing did I realize how simple the solution would be.

The effect is similar to TDD.  TDD doesn't magically create your design for you, but selecting the next test to write focuses your attention on something concrete while giving you a definition of success and an opportunity to improve your design by a small amount.  Similarly, tackling features one at a time doesn't magically create your design for you, but selecting the next feature focuses your attention on something concrete while giving you a larger-scale definition of success and a larger opportunity to improve your design.
