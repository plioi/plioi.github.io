---
title: "A C# Feature that Shocks the Conscience"
layout: post
---


The "default arguments" feature in C# seems like a great time-saver at first, but there is a sharp difference between what it *feels* like it does versus what it *actually* does which can lead to surprises.  Recently, fellow Headspring consultant <a href="https://twitter.com/pedroreys">Pedro Reys</a> discovered a particularly bizarre gotcha related to default arguments.

## Default Arugments Provide Brevity

Let's start with the pain that default arguments are meant to solve.  When overloading a method, we often do so because some arguments aren't always required or because some arguments have a reasonable default that you don't want to make consumers explicitly write again and again:

```cs
public void SayHello()
{
    SayHello(null);
}

public void SayHello(string name)
{
    Console.WriteLine("Hello, {0}!", name ?? "you");
}
```

Here, we don't want to make the consumer *always* specify the argument when a reasonable default exists.  When you want to specify a value, you can.  We have one overload simply call the other in order to avoid repeating ourselves.  Despite trying to avoid repeating ourselves, we really still have duplicated a few lines and a handful of characters of code here.  Enter default arguments:

```cs
public void SayHello(string name = null)
{
    Console.WriteLine("Hello, {0}!", name ?? "you");
}
```

## What Really Happens

This seems like a great little addition to the language.  It *feels* like the compiler is going to react to this by effectively writing the simple overload for me.  Unfortunately, including a default value does something else entirely.  Instead of "writing my overload for me", the compiler *alters the call sites* that make use of this method, effectively copying-and-pasting a `null` whenever we call the method.

In simple scenarios, this distinction doesn't matter.  The moment it does matter, though, you are going to be surprised.

## Interfaces Allow Contradictory Defaults

Let's say we've got some classes that perform an operation against an input int:

```cs
public class Negate
{
    public int Apply(int x = 1)
    {
        return -x;
    }
}

public class Square
{
    public int Apply(int x = 1)
    {
        return x*x;
    }
}

...

Negate operation = new Negate();

Console.WriteLine(operation.Apply());
Console.WriteLine(operation.Apply(2));

//Outputs:
//  -1
//  -2
```

Ok, no surprises yet.  When we omitted the argument in the first call, the compiler copied and pasted a 1 into the call site.

As we start adding more and more such operations to the program, a developer finds a need to extract a common interface for all such one-argument operations.  This developer means well, but when they write the interface definition, they don't look at each and every implementing class's defaults.  They figure 0 is a reasonable default for an integer, and define it like so:

```cs
public interface IUnaryOperation
{
    int Apply(int x = 0);
}

public class Negate : IUnaryOperation
{
    public int Apply(int x = 1)
    {
        return -x;
    }
}

public class Square : IUnaryOperation
{
    public int Apply(int x = 1)
    {
        return x*x;
    }
}
```

*Note that the default in the interface is different from the defaults on the concrete classes.*

Running the same sample code produces the same output as before:

```cs
Negate operation = new Negate();

Console.WriteLine(operation.Apply());
Console.WriteLine(operation.Apply(2));

//Outputs:
//  -1
//  -2
```

This makes sense.  We're calling `Negate.Apply`, and `Negate.Apply`'s default comes into play just like before.  We might (incorrectly!) conclude that the concrete runtime type (`Negate`) is used to determine which default value "wins" the competition.

Later, a new developer joins the project.  They are unaware that the interface and concrete classes have conflicting defaults.  They've read up on SOLID principles and naturally want to favor abstract types over concrete types.  They make one seemingly-safe change to the code, changing only the declared type of the `operation` variable:

```cs
IUnaryOperation operation = new Negate();

Console.WriteLine(operation.Apply());
Console.WriteLine(operation.Apply(2));

//Outputs:
//  0
//  -2
```

> The C# compiler finds our efforts to apply "nonbreaking changes" quaint but ultimately futile.

## What in the *what!?*

The runtime type of `operation` is the same as in the previous execution, and that type oh-so-clearly defines its own default as 1.  However, **the selection of which method will actually run is a separate decision from the selection of which default value gets copy/pasted to the call site.**  Default value selection is based on the compile-time type of `operation` (`IUnaryOperation`), because the copy/paste *happens* at compile time.  The selection of which method needs to execute happens at runtime, upon inspecting the actual type of `operation` (`Negate`).

In order to use default values well, you have to constantly think about how they were implemented.  That's enough to make you want to completely avoid this feature.

I'm still willing to use default arguments, with one simple rule of thumb: **pretend that you may only ever use a default value of 0, null, or false; <a href="http://msdn.microsoft.com/en-us/library/83fhsxwc(v=vs.80).aspx">the same defaults</a> we already expect for otherwise-uninitialized variables.**  If we always use these predictable defaults, we can avoid verbose overload definitions while being confident that we won't have our explicit defaults ignored at runtime.
