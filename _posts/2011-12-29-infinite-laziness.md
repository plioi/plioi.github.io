---
title: "Infinite Laziness"
layout: post
---


In <a href="http://www.haskell.org/haskellwiki/Haskell">Haskell</a>, expressions are evaluated lazily by default.  This is quite different from the way most languages behave.  By taking a dramatically different approach to something we take for granted (expression evaluation), you gain the ability to solve problems in different ways.

When you pass arguments to a method in C#, they are all evaluated right away.  Those calculated values are then passed to the method you are calling.  If the first argument to a function is a costly function call itself, the costly call takes place first.  Most languages do it this way, **and that just makes sense**.  Cause and effect seems to demand that arguments be evaluated before they are passed to another method, but Haskell actually turns cause and effect upside down.  When you pass arguments to a Haskell function, they are not evaluated right away.  Instead, they are only evaluated when the called-function actually needs to *look* at them!  We can simulate this behavior in C# using Func<T>:

```cs
public static int DoSomething(int x, int y)
{
    if (x > 0)
        return x;

    return y;
}

public static int DoSomethingLazily(Func<int> x, Func<int> y)
{
    var xValue = x();
            
    if (xValue > 0)
        return xValue;

    return y();
}
```

If we call `DoSomething(1, 1+1)`, the arguments are evaluated eagerly as usual.  If we instead call `DoSomethingLazily(() => 1, () => 1+1)`, the expression `1+1` will never actually be evaluated; we never needed to *look* at y!  **Haskell is basically wrapping every argument of every function call in a lambda, and every function receives those lambdas as something like a `Func<T>`**.

One benefit of Haskell's laziness-by-default is that it doesn't hurt to pass the result of a costly function to DoSomethingLazily, as the value is not always going to be looked at anyway: `DoSomethingLazily(() => 1, () => SomethingThatTakesTenYearsToCalculate())`.  Calling DoSomethingLazy with these arguments will return immediately, with no need to wait ten years.  You can get some surprising performance boosts without really trying in Haskell.  *On the other hand*, it could be quite difficult to determine where a bottleneck really *is* when you do run into one.

Another way laziness is useful is that it allows us to efficiently deal with apparently-infinite lists, made possible in C# with the `yield` keyword.  One of the most common examples of Haskell's lazy evaluation is the calculation of the Fibonacci numbers as a combination of two infinite lists:

```
fibs = 0 : 1 : zipWith (+) fibs (tail fibs)
```

Here, we define the fibs function to take zero arguments, and return a lazily-evaluated list.  Each value in the list will be calculated the moment we try to look at it.  "tail fibs" means "get the infinite Fibonacci series, skipping the first number".  "zipWith (+) a b" means "line up two lists (a and b) side by side, producing a new list by adding together each pair of values".  Translating all of this to C#, we get:

```cs
private static IEnumerable<int> ZipWith(Func<int, int, int> combinePair,
                                        IEnumerable<int> listA,
                                        IEnumerable<int> listB)
{
    return listA.Zip(listB, combinePair);
}

private static IEnumerable<int> Tail(IEnumerable<int> list)
{
    return list.Skip(1);
}

private static int Plus(int x, int y)
{
    return x + y;
}

//fibs = 0 : 1 : zipWith (+) fibs (tail fibs)
private static IEnumerable<int> Fibs()
{
    yield return 0;
    yield return 1;

    foreach (int sum in ZipWith(Plus, Fibs(), Tail(Fibs())))
        yield return sum;
}
        
public static void Main()
{
    foreach (int f in Fibs().TakeWhile(x => x < 100))
        Console.WriteLine(f);

    Console.ReadLine();
}
```

The `yield` keyword allows us to produce results on demand.  The `Enumerable` extension methods `Zip`, `Skip`, and `TakeWhile` are all likewise implemented with the `yield` keyword.  We have effectively glued together a chain of function calls, each of which draws out the next result on demand.  In the main loop, TakeWhile repeatedly tries to pull the next item of the Fibonacci series.  When it does so, Fib() runs until the next time it yields a value, which in turn relies on summing up two *other* 'instances' of the Fibonacci sequence.  Since everything's lazy, none of this gets stuck in an infinite loop.

Haskell assumes you'll want to be lazy all the time.  The benefit is that when you do want laziness, it comes with brevity.  C# on the other hand offers you two tools for laziness when you really *want/need* it: lambdas and yield.
