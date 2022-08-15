---
title: "Delegate Frustration"
layout: post
---


One of the things I like about C#'s delegate types, compared to similar concepts in other languages, is that C#'s take on the concept lets you give a useful *name* to a family of functions:

```cs
public delegate TResult Converter<T, TResult>(T input);
```

With this definition, all functions that turn one thing into another thing are Converters.  Nice.

We also have some generically-named `Action<...>` and `Func<...>` delegate types defined for us, and these have become ubiquitous ever since LINQ was introduced.  They're not as communicative, but still extremely useful, and for the most part the naming stays out of the way.

Delegate types allow you to give a name to a family of functions.  However, you can run into trouble when you use two different names to refer to the *same* family of functions:

```cs
public string Stringify(int x)
{
    return x.ToString();
}


//Later...

Func<int, string> func = Stringify;
Converter<int, string> convert = Stringify;
```

Since all methods are compatable with any delegate type that has matching input/output types, why aren't the following queries equivalent?

```cs
int[] integers = new[] {1, 2, 3};

integers.Select(func); // returns "1", "2", "3"
integers.Select(convert); // fails to compile!
```

This was a little frustrating when C# 3 came out.  Before then, all the delegate types we were used to working with had explicit names like `Converter` and `Predicate`.  When C# 3 introduced lambdas and LINQ, we gained `Func<...>` and `Action<...>`, and I found myself stumbling while converting older code to use the new LINQ extension methods like `Select`, which takes a `Func<T, TResult>`.

It's extra-frustrating because *every single `Converter<T, TResult>` method that ever lived is also a `Func<T, TResult>`!* I know it, you know it, Grandma tweeted it.  We can drive that point home by making the `convert` delegate work with `Select` by wrapping it in a trivial do-nothing lambda: `integers.Select(x => convert(x));`  Yeesh!

To help make sense of this behavior, we can make an analogy between delegates and interfaces.

Think of a delegate *type* as a special kind of interface type.  The interface has only one method, called "Call".  Since it is *always* called "Call", you can shorten the reader-unfriendly `convert.Call(5)` to simply `convert(5)`.

Think of a method as an *instance* of these interfaces.  All methods implement the infinite number of possible delegate 'interfaces' that match their input and output types.  `Stringify` is an object that implements the imaginary `IConverter<int, string>` interface, and also implements the imaginary `IFunc<int, string>` interface.  Both of these interfaces just happen to contain the imaginary method with the signature: `string Call(int x)`.

Now, the example above makes more sense.  We have two variables declared as 'interface' types, and assign the same object to both variables.  `Select` is written to accept instances of one 'interface' type, and although `convert` points at a method compatable with `Select`, `convert` itself can only be seen as having some other, incompatible type.
