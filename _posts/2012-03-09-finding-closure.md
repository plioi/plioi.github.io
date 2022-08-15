---
title: "Finding Closure"
layout: post
---


When writing software, we constantly create abstractions.  The goal each time is to present a simple interface in front of something more complex, hiding the gory details from us so we can focus on other things.  The problem is, <a href="http://en.wikipedia.org/wiki/Leaky_abstraction">abstractions leak</a>.  Even when you go to great effort to create a really exellent abstraction, some unfortunate implementation detail manages to rear its ugly head.  This even happens with things so fundamental that we take them for granted.  Take the 'foreach' keyword in C#:

```cs
var delayedGreetings = new List<Func<string>>();

foreach (var name in new[] { "Larry", "Moe", "Curly" })
    delayedGreetings.Add(() => "Hello, " + name + "!");
            
//Some time later...
foreach (var greeting in delayedGreetings)
    Console.WriteLine(greeting());
```

Where we would expect the output to be:<br />Hello, Larry!<br />Hello, Moe!<br />Hello, Curly!

Instead, we see:<br />Hello, Curly!<br />Hello, Curly!<br />Hello, Curly!

Fortunately, ReSharper points out that something fishy is going on with the reference to `name` that appears in the loop body: **"Access to modified closure."**  If you follow the ReSharper suggested fix, you get this:

```cs
foreach (var name in new[] { "Larry", "Moe", "Curly" })
{
    string name1 = name; //Added a seemingly-pointless local variable!
    delayedGreetings.Add(() => "Hello, " + name1 + "!");
}

//Some time later...
foreach (var greeting in delayedGreetings)
    Console.WriteLine(greeting());
```

Oddly, introducing this local variable *does* fix the output, but why?  To find out, let's first rewrite the original version using a transformation from a `foreach` loop to a `while` loop, since that's the first thing the compiler does for us.  It still erroneously prints "Hello, Curly!" three times:

```cs
var delayedGreetings = new List<Func<string>>();

{
    using (var e = ((IEnumerable<string>)new[] { "Larry", "Moe", "Curly" }).GetEnumerator())
    {
        string name;
        while (e.MoveNext())
        {
            name = e.Current;
            delayedGreetings.Add(() => "Hello, " + name + "!");
        }
    }
}

//Some time later...
foreach (var greeting in delayedGreetings)
    Console.WriteLine(greeting());
```

ReSharper still highlights the use of `name` within the loop body, and still says it is an "Access to modified closure."  Before we can try to fix the problem, we're going to have to make sense of that warning.

## What's a Closure?

The word "closure" comes up a lot when talking about any language that has anonymous inline functions.  In C#, that means anonymous delegates and their shorthand lambda expressions introduced with `=>`.  You have comparable anonymous functions in Javascript, Ruby, Python, etc.  Whenever you have an anonymous function whose body refers to identifiers in the surrounding scope, the function earns the right to call itself a closure.

Lambdas are really shorthand for a class with one method as well as the instantiation of that class.  In our example above, the lambdas are living beyond the execution of the loop, and that means they need to live beyond the normal scope of the loop variable, `name`.  In order to work, they'll need to hold onto something about that variable.  In fact, they hold onto the *environment* that was in effect at the moment the lambda was reached.  By environment, I mean *all* of the local variables they might be referring to.  It is as if we wrote the following, which also prints "Hello, Curly!" three times:

```cs
public class Environment
{
    public string name;
    //As well as one field for each of the
    //original loop's local variables, if
    //there were any...
}

public class Greeter
{
    private readonly Environment _environment;

    public Greeter(Environment environment)
    {
        _environment = environment;
    }

    public string Execute()
    {
        return "Hello, " + _environment.name + "!";
    }
}

static void Main(string[] args)
{
    var delayedGreetings = new List<Greeter>();

    {
        using (var e = ((IEnumerable<string>)new[] { "Larry", "Moe", "Curly" }).GetEnumerator())
        {
            var environment = new Environment();
            while (e.MoveNext())
            {
                environment.name = e.Current;
                delayedGreetings.Add(new Greeter(environment));
            }
        }
    }

    //Some time later...
    foreach (var greeting in delayedGreetings)
        Console.WriteLine(greeting.Execute());

    Console.ReadLine();
}
```

AHA! See how we have only *one* instance of Environment shared by all the `Greeter`s, and each iteration of the loop messes with the `name` field *inside* that shared Environment?  No wonder our final loop prints out the same thing each time: all the items in `delayedGreetings` refer to the same Environment, which contains only one name, which equals whatever we set it to *last*.

They call these lambdas "closures" because the secret object behind the scenes is grasping-at / enveloping / *closing-on* the local environment.  As we see in the last example, even after the environment is closed on, we can still affect that environment.  We say that the lambda is closing on the *variables themselves* rather than simply holding a reference to their *values*.

**Even foreach leaks.**

## Fixing foreach

So we've got this fancy compiler that can take a combination of `foreach` and lambda expressions, rewriting them into some equivalent-yet-verbose code, which in turn is easier to compile.  It's a big abstraction that usually works so well we don't stop to think it's even *happening*.  How could we fix it?

It seems the whole problem comes down to the fact that with one shared environment, each iteration messes with a shared mutable object.  We get one environment per scope, though, and the `while` loop body introduces a new, more-local scope of its own!  We could take advantage of that by declaring the loop-iteration variable `name` *within* the `while` instead of *before* it.  This way, each iteration will get its own Environment with its own `name`.  Nothing shared, nothing mutated, leaving us with the three distinct greetings we wanted.  In other words, we wish that `foreach` gets translated into a `while` loop like so:

```cs
var delayedGreetings = new List<Func<string>>();

{
    using (var e = ((IEnumerable<string>)new[] { "Larry", "Moe", "Curly" }).GetEnumerator())
    {
        while (e.MoveNext())
        {
            string name = e.Current; //Variable declaration inside the while.
            delayedGreetings.Add(() => "Hello, " + name + "!");
        }
    }
}

//Some time later...
foreach (var greeting in delayedGreetings)
    Console.WriteLine(greeting());
```

ReSharper's suggestions basically accomplish the same thing.  By introducing a seemingly-redundant copy within the smaller scope, the environment used by the lambdas is different in each iteration.

Usually when something as fundamental as a looping keyword has a problem like this, the language designers don't really have the option of correcting it.  It's technically a breaking change to make all `foreach` loops translate to the better version of the `while` seen in the last example.  **However, they are going to be <a href="http://blogs.msdn.com/b/ericlippert/archive/2009/11/12/closing-over-the-loop-variable-considered-harmful.aspx">fixing foreach in C#5</a>.** It's a breaking change, but one that will *fix* more mistakes than it *creates*!
