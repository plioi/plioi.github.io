---
title: "By Your Command (Line)"
layout: post
---


Last week, we saw an example of <a href="http://patrick.lioi.net/2013/07/05/dynamic-test-discovery/">Dynamic Test Discovery</a> in Fixie, in which the test runner can be made to run a different set of tests each time, depending on context. A Fixie convention could make decisions based on how the test run was initiated. This week, I'll demonstrate a similar feature which takes advantage of yet more context.  For this example, our convention will get to make decisions based on the command line arguments used to kick off the test run.

> Today's code sample works against <a href="http://nuget.org/packages/Fixie/0.0.1.70">Fixie 0.0.1.70</a>. The customization API is in its infancy, and is likely to change in the coming weeks.

Before this build, Fixie.Console.exe treated *all* of its arguments as paths to test assembly files. As of Fixie 0.0.1.70, you can also specify arbitrary key/value pairs on the command line:

```
$ Fixie.Console "C:\path\to\test\assembly.dll" "C:\path\to\another\test\assembly.dll" --key1 value1 --key2 "value 2"
```

The console runner converts the named arguments into an `ILookup<string, string>`.  ILookup objects are similar to a dictionary, except that each key can hold any number of values instead of just one.

Let's take advantage of this ILookup collection in order to implement NUnit-like categories. Consider a test class with several tests, some of which have been categorized with attributes:

```cs
using System;

namespace CategoriesDemo.Tests
{
    public class CategorizedTests
    {
        public void Uncategorized()
        {
            Console.WriteLine("Uncategorized");
        }

        [CategoryA]
        public void TestA()
        {
            Console.WriteLine("TestA");
        }

        [CategoryB]
        public void TestB1()
        {
            Console.WriteLine("TestB1");
        }

        [CategoryB]
        public void TestB2()
        {
            Console.WriteLine("TestB2");
        }
    }
}
```

Fixie has no idea what these category attributes mean, and it has no idea what any of the command line key/value pairs mean. Fixie's responsibility in the matter is simply to pass the ILookup along so that your custom conventions can make use of them however you see fit. Along with test classes like CategorizedTests, my test assembly contains my own category attributes:

```cs
using System;

namespace CategoriesDemo.Tests
{
    [AttributeUsage(AttributeTargets.Method, Inherited = false)]
    public abstract class CategoryAttribute : Attribute
    {
        public string Name
        {
            get { return GetType().Name.Replace("Attribute", ""); }
        }
    }

    [AttributeUsage(AttributeTargets.Method, Inherited = false)]
    public class CategoryAAttribute : CategoryAttribute { }

    [AttributeUsage(AttributeTargets.Method, Inherited = false)]
    public class CategoryBAttribute : CategoryAttribute { }
}
```

Beside these category types, we define a custom convention to take advantage of them:

```cs
using System.Linq;
using System.Reflection;
using Fixie;
using Fixie.Conventions;

namespace CategoriesDemo.Tests
{
    public class CustomConvention : Convention
    {
        public CustomConvention(RunContext runContext)
        {
            var desiredCategories = runContext.Options["include"].ToArray();
            var shouldRunAll = !desiredCategories.Any();

            Classes
                .NameEndsWith("Tests");

            Cases
                .Where(method => method.Void())
                .ZeroParameters()
                .Where(method => shouldRunAll || MethodHasAnyDesiredCategory(method, desiredCategories));
        }

        static bool MethodHasAnyDesiredCategory(MethodInfo method, string[] desiredCategories)
        {
            return Categories(method).Any(testCategory => desiredCategories.Contains(testCategory.Name));
        }

        static CategoryAttribute[] Categories(MethodInfo method)
        {
            return method.GetCustomAttributes<CategoryAttribute>(true).ToArray();
        }
    }
}
```

As we saw last week, our convention class can optionally accept a RunContext in its constructor. Now, this RunContext includes the ILookup of command line arguments. This convention says that a method in a test class should be treated as a runnable test case when a) no categories have been requested or b) categories have been requested and the method in question has at least one matching category attribute.

Let's run our tests a few times, with different command line arguments:

```
$ Fixie.Console.exe "C:\path\to\CategoriesDemo.Tests.dll"

------ Testing Assembly CategoriesDemo.Tests.dll ------

Uncategorized
TestA
TestB1
TestB2
4 passed, 0 failed (Fixie 0.0.1.70).



$ Fixie.Console.exe "C:\path\to\CategoriesDemo.Tests.dll" --include CategoryA 

------ Testing Assembly CategoriesDemo.Tests.dll ------

TestA
1 passed, 0 failed (Fixie 0.0.1.70).



$ Fixie.Console.exe "C:\path\to\CategoriesDemo.Tests.dll" --include CategoryB 

------ Testing Assembly CategoriesDemo.Tests.dll ------

TestB1
TestB2
2 passed, 0 failed (Fixie 0.0.1.70).



$ Fixie.Console.exe "C:\path\to\CategoriesDemo.Tests.dll" --include CategoryA --include CategoryB 

------ Testing Assembly CategoriesDemo.Tests.dll ------

TestA
TestB1
TestB2
3 passed, 0 failed (Fixie 0.0.1.70).
```

> Oh, dear. When creating this demo project, I discovered a bug with the way the assembly path arguments are treated. If you use relative file paths, Fixie will fail to find them, and will produce a completely useless error message. It incorrectly alters the current directory *before* resolving the paths. That's easy to fix, but if you try this demo out yourself against this build, be sure to use an absolute path like the example above. Yay for dogfooding!

Ushering arbitrary key/value pairs from the command line to your custom conventions is a very *small* feature, but one that opens up an important door for open-ended customization.
