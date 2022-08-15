---
title: "Low-Ceremony xUnit"
layout: post
---


"High Ceremony" software includes excessive detail which distracts from the essential thing the coder is trying to convey.  In C# for instance, you could argue that having to place *all* methods inside classes is needless ceremony.  Your Main() method shouldn't need that extra boilerplate, but we just Have To Do It That Way.  We have to perform that ceremony.  Developers who favor dynamic languages would find type declarations to be High Ceremony as well, and even C#ers admit this whenever they use the `var` or `dynamic` keywords.

High ceremony isn't limited to language details, though.  We can be guilty of introducing it, ourselves.

## Framework Ceremony

Frameworks often impose ceremony of their own.  ASP.NET MVC, NUnit, and xUnit all require you to litter your code with `[Attribte]`s in order to give instructions to the framework.  The framework reflects over your code, discovers these attributes, and takes action.  Usually this is all done so that the framework will know when, during its own lifecycle, your code should be invoked.  Take a look at NUnit test fixtures:

```cs
[TestFixture]
public class ArithmeticTests
{
    [Test]
    public void Addition()
    {
        int sum = 1 + 2;
        sum.ShouldEqual(3);
    }
}
```

We *want* to simply define a named test and make an assertion about our system.  Additionally, we have to pay the Attribute Tax so that NUnit knows which classes contain tests (`[TestFixture]`), and which methods within those classes are tests (`[Test]`).  When you take part in the rest of the NUnit fixture lifecycle, you have to litter even more attributes around like `[SetUp]`, `[TearDown]`, `[TestFixtureSetUp]`, and `[TestFixtureTearDown]`.  Since these lifecycle attributes usually decorate methods with the same name as the attribute, it gets even more pointlessly verbose:

```cs
[TestFixture]
public class RedundancyTests
{
    [TestFixtureSetUp] //Oh, is *that* what this is?
    public void TestFixtureSetUp()
    {
        ...
    }

    [TestFixtureTearDown]
    public void TestFixtureTearDown()
    {
        ...
    }

    [SetUp]
    public void SetUp()
    {
        ...
    }

    [TearDown]
    public void NotTearDown_LOL_JK()
    {
        ...
    }

    ...
}
```

xUnit avoids *some* of this by using different conventions.  Test methods are still marked with an attribute (this time, `[Fact]` instead of `[Test]`).  Instead of marking test classes with an attribute, xUnit just infers the obvious: any class with `[Fact]`s must be a test class.  Instead of setup and teardown methods, xUnit relies on idioms already common in C#: default constructors for setup, and Dispose() for teardown.

I've been using xUnit over NUnit for these reasons (however trivial they are in the context of a large project).  Still, I hate typing ceremony into a project, and it seems like xUnit's default conventions got it backwards: why not place an attribute on the test class itself, and then just *assume* all the public methods in them are tests?

## Minimizing xUnit's Ceremony

One important benefit of xUnit over NUnit is that xUnit is partially customizable.  If you want to define your own alternative to `[Fact]`, you can.  Also, if you want to use your own convention for discovering which methods in a test class are actually test cases, you can define your own class-level attribute.  This custom attribute goes at the top of the test class and basically means, "Hold on xUnit, I'll take it from here."

To implement your own test-method-discovery convention, you must implement ITestClassCommand.  Unfortunately, this interface violates the <a href="http://en.wikipedia.org/wiki/Interface_segregation_principle">Interface Segregation Principle</a>: this interface is large, serving several purposes.  I'm only interested in providing a new method which says whether or not a given method is a test method.  **Instead of only telling xUnit how to do that, I have to also tell it how to do several other things for which I really just want the default behavior.**

> ISP violations force you to take part in yet more ceremony, supplying boilerplate implementations for details you don't wish to customize.

To account for this ISP violation, I first created an abstract implementation of ITestClassCommand which uses the default behavior for all these unrelated details, leaving subclasses the responsibility of only having to describe the "test method discovery" part.

```cs
public abstract class TestDiscoveryCommand : ITestClassCommand
{
   private readonly TestClassCommand defaultBehavior = new TestClassCommand();

   public abstract bool IsTestMethod(IMethodInfo testMethod);

   public abstract IEnumerable<ITestCommand> EnumerateTestCommands(IMethodInfo testMethod);

   public IEnumerable<IMethodInfo> EnumerateTestMethods()
   {
       return TypeUnderTest.GetMethods().Where(IsTestMethod);
   }

   public object ObjectUnderTest
   {
       get { return defaultBehavior.ObjectUnderTest; }
   }

   public ITypeInfo TypeUnderTest
   {
       get { return defaultBehavior.TypeUnderTest; }
       set { defaultBehavior.TypeUnderTest = value; }
   }

   public int ChooseNextTest(ICollection<IMethodInfo> testsLeftToRun)
   {
       return defaultBehavior.ChooseNextTest(testsLeftToRun);
   }

   public Exception ClassStart()
   {
       try
       {
           foreach (var @interface in TypeUnderTest.Type.GetInterfaces())
               if (@interface.IsGenericType && @interface.GetGenericTypeDefinition() == typeof(IUseFixture<>))
                   throw new NotSupportedException(GetType() + "does not support IUseFixture<>.");

           return null;
       }
       catch (Exception ex)
       {
           return ex;
       }
   }

   public Exception ClassFinish()
   {
       return null;
   }
}
```

Unfortunately, the ISP violation was so significant here that xUnit *really, really* wanted me to reimplement a fairly involved feature that I never use, <a href="http://xunit.codeplex.com/wikipage?title=Comparisons#note3">`IUseFixture<T>`</a>.  I decided to give up here, instead implementing the ClassStart method to fail fast: it complains if a developer ever assumes that TestDiscoveryCommands take part in this little-used feature.

> ISP isn't just about being pedantic or fearing all interfaces containing more than one method: in order to write this class properly, I had to read a great deal of xUnit's implementation details and *still* had to gut the `IUseFixture<T>` feature.

Now that TestDiscoveryCommand insulates me from the ISP violation, I can very easily sublcass *that* to introduce my own test-method-discovery logic:

```cs
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public class FactsAttribute : RunWithAttribute
{
    public FactsAttribute()
        :base(typeof(FactsTestClassCommand)) { }
}

public class FactsTestClassCommand : TestDiscoveryCommand
{
    public override bool IsTestMethod(IMethodInfo testMethod)
    {
        return !testMethod.IsAbstract &&
               !testMethod.IsStatic &&
               testMethod.MethodInfo.IsPublic &&
               testMethod.MethodInfo.ReturnType == typeof(void) &&
               testMethod.MethodInfo.GetParameters().Length == 0;
    }

    public override IEnumerable<ITestCommand> EnumerateTestCommands(IMethodInfo testMethod)
    {
        yield return new FactCommand(testMethod);
    }
}
```

(*Aside: `void` is not a type, but `typeof(void)` is a `Type`.  Yikes.*)

With these 3 classes in my test assembly, I can convert the old, high-ceremony xUnit style (zero attributes on test classes, one `[Fact]` per test method) <a href="https://github.com/plioi/rook/compare/d48039983bebd2ee3dbaf09992ad48b438392204...ff8fb1c6896d20db2cd0a2a637a744f2d8827b71">into a slightly-cleaner style</a> (one `[Facts]` attribute per test class, zero attributes on test methods).

Yes, I was bored.  Very, very bored.
