---
title: "You could have invented lambda expressions"
layout: post
---


Delegate types and lambda expressions have been a part of the .NET tool belt for a while now, so they show up everywhere. If you've ever used StructureMap, AutoMapper, ASP.NET MVC and the like, then you've surely run into them. Despite this, delegates and lambdas tend to confuse developers when first encountered. Even when you get comfortable with the syntax as it's used in your favorite library, you might not really be familiar with what they *are* or how they allow for such seemingly magical statements.

Let's pretend that delegate types and lambdas don't exist yet, and then reinvent them in terms of other .NET concepts that are more widely understood.

Suppose we have been given an `IEnumerable<Customer>` and we want to filter that down to the list of those customers who live in Texas. Without delegates, we could write the obvious loop like so:

```cs
public static IEnumerable<Customer> Texans(IEnumerable<Customer> allCustomers)
{
    var texans = new List<Customer>();
    foreach (var customer in allCustomers)
        if (customer.State == State.Texas)
            texans.Add(customer);
    return texans;
}

...

var texans = Texans(allCustomers);
```

Yeesh. I have written foreach-if-add loops a thousand times. Whenever we find ourselves writing virtually the same thing again and again, we should take a step back and think, "Is there a deeper concept here I could pull out into a library?" We have some subtle code duplication whenever we write foreach-if-add loops.

To remove duplication, we first identify what parts vary from one instance of the pattern to the next. For foreach-if-add loops, the things that vary are the type of the items being filtered, and the if-statement's condition expression. To bottle up the foreach-if-add loop, we need a new method that takes in the condition as an *argument*. Without delegates, we could deal with this using an interface:

```cs
public interface IFilterCondition<T>
{
    bool RunMe(T item);
}

public class InTexasFilter : IFilterCondition<Customer>
{
    public bool RunMe(Customer customer)
    {
        return customer.State == State.Texas;
    }
}

public static class CollectionExtensions
{
    public static IEnumerable<T> Where<T>(this IEnumerable<T> allItems,
                                          IFilterCondition<T> condition)
    {
        var result = new List<T>();
        foreach (var item in allItems)
            if (condition.RunMe(item))
                result.Add(item);
        return result;
    }
}

...

var texans = allCustomers.Where(new InTexasFilter());
```

Ok, so we've pulled out the foreach-if-add loop into a reusable `Where` extension method. Still, though, that's a lot of boilerplate code: every time we want to call `Where` with a new filter, we have to bottle it up in a whole new class.

Let's start removing boilerplate code until we're left with delegates! With this example, we can **imagine a whole family of interfaces that have one thing in common: interfaces with a single method named `RunMe`, whose implementing classes contain only that method.** If we knew this family of interfaces only have one method, and its name was always `RunMe`, then we could avoid even writing its name: `condition.RunMe(item)` could be shortened to simply `condition(item)`. Since the relevant part of a single `IFilterCondition` implementation is the body of the now-nameless method, we should be able to simply provide the code for that method inline:

```cs
var texans = allCustomers.Where(delegate(Customer customer) { return customer.State == State.Texas; });
```

**We can imagine that this use of the `delegate` keyword is just shorthand for an instance of a class implementing our single-method interface.**

Since most of these small unnamed classes exist just to evaluate and return a single expression, we might as well get rid of the `return` keyword and braces. Since we already know what the item type T is, we might as well leave out the type declaration of the `customer` parameter, allowing the compiler to fill in the blank. We'll need *something* between the argument list and the return expression so the compiler knows which is which, so we introduce the `=>` operator. This makes the `delegate` keyword redundant, leaving us with some valid (and clean!) C#:

```cs
public delegate bool FilterCondition<T>(T item);

public static class CollectionExtensions
{
    public static IEnumerable<T> Where<T>(this IEnumerable<T> allItems, FilterCondition<T> condition)
    {
        var result = new List<T>();
        foreach (var item in allItems)
            if (condition(item))
                result.Add(item);
        return result;
    }
}

...

var texans = allCustomers.Where(customer => customer.State == State.Texas);
```

There's a lot more to delegates and lamda expressions, but in most cases I find it easiest to reason about them as shorthand for the interfaces and classes we would have otherwise used.
