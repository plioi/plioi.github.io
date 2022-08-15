---
title: "Leaky Abstractions Rot Code"
layout: post
---


Over the past month, we've seen the inception and early development of my ongoing side project, <a href="https://github.com/fixie/fixie">the Fixie test framework</a>.  As of last week, it was far enough along to run all of its own tests. Although I'm pleased with the progress so far, last week's success introduced an exceptionally leaky abstraction.  Today, we'll see how I patched up the leak.

A software abstraction, such as an interface or abstract class, "leaks" when it fails to hide implementation details. Leaky abstractions are annoying enough when the damage is local to a small area of a project, but **when the leak has a high risk of spreading, it can cause your code to rot.**  An abstraction's leak can spread when the calling code takes on too much knowledge of the leaked information, to the point where the calling code becomes a new source of that leaked information.

> A calls B, which calls C, which calls your leaky abstraction.  C is forced to deal with information you intended to hide, putting it at high risk of leaking that information back to B.  If B starts dealing with that information, A is at risk, too. The ugliness rots all the way through your system.

Once the information you intended to hide has spread, everything it touches is now brittle.  If you make changes within the abstraction's implementation, those details may cause a ripple effect through all the rotting code.  If other people or projects now depend on the leaked information, you may not even have the option of making the desired changes in the first place!

## Fixie's Leaky Abstraction

By the end of last week's post, I had developed <a href="http://patrick.lioi.net/2013/03/26/bootstrapping/">a roughly-usable test framework</a>. This version had an abstraction called Convention which is fundamental to the project: I want the end user to be able to tell me what their test fixtures and test cases look like.  I want you to be able to say things like "My test fixtures are the classes whose names end with 'Tests'", or "My test fixtures are the classes marked with some [Attribute]".  If you don't provide such a description, you'll get the <a href="https://github.com/fixie/fixie/blob/6a01e382f30c3c598cf7d3d3a3bde450ad684297/src/Fixie/DefaultConvention.cs">default convention</a>:

```cs
public class DefaultConvention : Convention
{
    protected override bool ClassIsFixture(Type concreteClass)
    {
        return concreteClass.Name.EndsWith("Tests") &&
               concreteClass.GetConstructor(Type.EmptyTypes) != null;
    }

    protected override MethodInfo[] QueryCaseMethods(Type fixtureClass)
    {
        return fixtureClass.GetMethods(BindingFlags.Public | BindingFlags.Instance)
                           .Where(method =>
                                  method.ReturnType == typeof(void) &&
                                  method.GetParameters().Length == 0).ToArray();
    }
}
```

The first method says, "A class is a test fixture when its name ends with 'Tests' and it has a default constructor."  The second method says, "A method is a test case if it is a public instance void method with zero parameters." To its credit, DefaultConvention successfully describes the behavior you'll get by default.  However, there's something wrong here.

The lack of symmetry is suspicious.  To tell me what your test *fixtures* look like, you implement a method that says whether or not a *single* class is a test fixture.  To tell me what your test *cases* look like, you implement a method that *lists* all the methods that look like test cases.

I originally wanted both methods to take a single candidate and return a bool, meaning the second method would accept one MethodInfo, returning true when that method looks like a test case.  I couldn't just do that, because the .NET reflection API wants me to start things off with a BindingFlags value to describe visibility (public/private/static/instance/etc), and *then* filter those results down.  Contrast this two-step process with the way reflection wants you to search for Types: you can take an Assembly, call GetTypes(), and *then* filter down on things *including* visibility.

> In the .NET reflection API, searching for types feels very different from searching for methods, even though a lot of the same logic and conditions apply to both.  This same asymmetry is present in the Convention abstraction.  Convention is leaking implementation details about reflection!

With this design, anyone who took advantage of Convention to customize their tests would be forced to understand a lot about the reflection API's quirks.  Worse, they'd have to write some suspiciously-similar boilerplate code each time they wanted to provide an implementation.  DefaultConvention says what the default behavior is, but it isn't very clear to the reader.

## The Fix

Let's jump to the improved abstraction, and then see how it works.  Here's the new version of <a href="https://github.com/fixie/fixie/blob/cb649450e5d42a6eb83faeb58dafc3b9511d92d1/src/Fixie/DefaultConvention.cs">DefaultConvention</a>:

```cs
public class DefaultConvention : Convention
{
    public DefaultConvention()
    {
        Fixtures
            .NameEndsWith("Tests")
            .HasDefaultConstructor();

        Cases
            .Visibility(BindingFlags.Public | BindingFlags.Instance)
            .Void()
            .ZeroParameters();
    }
}
```

Much better. It says exactly what test fixtures and test cases look like by default, and suggests how you might specify your own similar rules. There's no suspicious asymmetry now: both test fixtures and test cases are discovered using very similar statements.  It's declarative, so the end user doesn't have to be concerned with *how* the discovery will be honored; they can focus on simply stating what success looks like.

The Fixtures property is an instance of <a href="https://github.com/fixie/fixie/blob/cb649450e5d42a6eb83faeb58dafc3b9511d92d1/src/Fixie/ClassFilter.cs">ClassFilter</a>, and the Cases property is an instance of <a href="https://github.com/fixie/fixie/blob/cb649450e5d42a6eb83faeb58dafc3b9511d92d1/src/Fixie/MethodFilter.cs">MethodFilter</a>:

```cs
public class ClassFilter
{
    readonly List<Func<Type, bool>> conditions;

    public ClassFilter()
    {
        conditions = new List<Func<Type, bool>>();

        ConcreteClasses();
    }

    public ClassFilter Where(Func<Type, bool> condition)
    {
        conditions.Add(condition);
        return this;
    }

    public ClassFilter HasDefaultConstructor()
    {
        return Where(type => type.GetConstructor(Type.EmptyTypes) != null);
    }

    public ClassFilter NameEndsWith(string suffix)
    {
        return Where(type => type.Name.EndsWith(suffix));
    }

    public IEnumerable<Type> Filter(IEnumerable<Type> candidates)
    {
        return candidates.Where(IsMatch);
    }

    private bool IsMatch(Type candidate)
    {
        return conditions.All(condition => condition(candidate));
    }

    private void ConcreteClasses()
    {
        Where(type => type.IsClass && !type.IsAbstract);
    }
}
```

```cs
public class MethodFilter
{
    BindingFlags bindingFlags;
    readonly List<Func<MethodInfo, bool>> conditions;

    public MethodFilter()
    {
        bindingFlags = BindingFlags.Default;
        conditions = new List<Func<MethodInfo, bool>>();

        ExcludeMethodsDefinedOnObject();
    }

    public MethodFilter Visibility(BindingFlags flags)
    {
        bindingFlags |= flags;
        return this;
    }

    public MethodFilter Where(Func<MethodInfo, bool> condition)
    {
        conditions.Add(condition);
        return this;
    }

    public MethodFilter Void()
    {
        Where(method => method.ReturnType == typeof(void));
        return this;
    }

    public MethodFilter ZeroParameters()
    {
        Where(method => method.GetParameters().Length == 0);
        return this;
    }

    public IEnumerable<MethodInfo> Filter(Type type)
    {
        return type.GetMethods(bindingFlags).Where(IsMatch).ToArray();
    }

    private bool IsMatch(MethodInfo candidate)
    {
        return conditions.All(condition => condition(candidate));
    }

    private void ExcludeMethodsDefinedOnObject()
    {
        Where(method => method.DeclaringType != typeof(object));
    }
}
```

Both of these objects work in a similar way, achieving the symmetry I'd originally intended.  They each accumulate conditions in a list, and only perform that work when the Filter(...) method is finally called by Fixie's main loop.  Since the real work is deferred to that last moment, MethodFilter is able to similarly accumulate information about the awkward BindingFlags, thus hiding the worst of the information leak found in the original version.

Since each condition-building method returns `this`, the caller can Chain().Together().Calls().Like().This().  This approach gives me a home to place reflection helper methods like HasDefaultConstructor, further hiding odd implementation details about the reflection API, and if I happen to omit such a convenient helper method, the end user can build extension methods upon this foundation using the Where(...) method.

## Criticism

I'm still leaking too much of the reflection API's quirks.  The BindingFlags enum is a complicated mish-mash of unrelated concepts, so only some of its values even make sense for a call to Type.GetMethods(...).  I should hide the need for the end user to know that, instead exposing a few more helper methods with obvious names for the values that *do* make sense here.  It's not perfect, but it's a much better sitation than before, and will be easier to reshape over the coming weeks as I get a better idea of what all Conventions really need to do.

At this point, my goal was to address the immediate risks presented by the older Convention abstraction.  Now that those risks are mitigated, I'll hold off on doing too much design up front for the remainder.  Instead, I'll let real use cases for custom Conventions drive the rest of the abstraction's design.
