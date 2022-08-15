---
title: "Loops Obfuscate"
layout: post
---


Most of the world's loops have already been written, and their names are "Where" and "Select".

Loops are so commonplace in software that it barely deserves mentioning.  It's like saying "saws are ubiquitous in woodworking" or "epic guitar solos are ubiquitous in Journey albums".  Although there is nothing good or bad about loop constructs themselves, LINQ has changed the way I think about them over the last few years.

Let's start with some typical, monotonous loops:

```cs
public IEnumerable<Product> ValuableProducts(IEnumerable<Product> products)
{
  var result = new List<Product>();
  foreach (var product in products)
  {
    bool valuable = product.Price > 100;

    if (valuable)
      result.Add(product);
  }
  return result;
}

public IEnumerable<string> FormattedProducts(IEnumerable<Product> products)
{
  var result = new List<string>();
  foreach (var product in products)
  {
    string formatted = String.Format("{0}: {1}", product.Number, product.Name);

    result.Add(formatted);
  }
  return result;
}
```

There's nothing Earth-shattering about these methods, and there's nothing particularly wrong with them.  Let's assume, though, that as requirements change the condition for *valuable-ness* and the means of formatting items become more comlex.  Keeping that logic inside these methods would start to feel a little wrong, as it would <a href="http://en.wiktionary.org/wiki/complect">complect</a> the low-level details of iteration and list-building with the high-level business concepts of product value and formatting.

**I don't want to think about iteration and list-building when I'm also trying to think about business concepts.**

Let's say that once these rules got complicated enough, and reading them as monolithic methods got a little too painful, we simplify by doing an Extract-Method refactoring on them:

```cs
public IEnumerable<Product> ValuableProducts(IEnumerable<Product> products)
{
  var result = new List<Product>();
  foreach (var product in products)
    if (IsValuable(product))
      result.Add(product);
  return result;
}

public bool IsValuable(Product product)
{
  ...
}

public IEnumerable<string> FormattedProducts(IEnumerable<Product> products)
{
  var result = new List<string>();
  foreach (var product in products)
    result.Add(Format(product));
  return result;
}

public string Format(Product product)
{
  ...
}
```

Now it's easier to see that we're building up `List` objects just so that we can return them for `IEnumerable` iteration.  Armed with the `yield` keyword, we clean up further:

```cs
public IEnumerable<Product> ValuableProducts(IEnumerable<Product> products)
{
  foreach (var product in products)
    if (IsValuable(product))
      yield return product;
}

public IEnumerable<string> FormattedProducts(IEnumerable<Product> products)
{
  foreach (var product in products)
    yield return Format(product);
}
```

Imagine we have performed a similar refactoring all over a large project.  Our code would be littered with these annoying loop methods that are all suspiciously similar to each other.  All the methods like `ValuableProducts` would take a collection, loop through it, and apply a filter condition function to each item. All the methods like `FormattedProducts` would take a collection, loop through it, and apply some transforming function on each item.

They're all so similar that you might be tempted to reach for your copy/paste shortcuts to write them, and that should tell us something about the value of writing that code: **the operations of iterating, filtering, and transforming collections are so fundamental that we shouldn't have to write them out.**

Using delegates and generics, we can pluck out the tiny parts that vary from one copy/pasted loop method to another, collapsing all those methods down to two:

```cs
public IEnumerable<T> Filter(this IEnumerable<T> items, Func<T, bool> condition)
{
  foreach (var item in items)
    if (condition(item))
      yield return item;
}

public IEnumerable<TOutput> Transform<TInput, TOutput>(this IEnumerable<TInput> items, Func<TInput, TOutput> transform)
{
  foreach (var item in items)
    yield return transform(item);
}
```

At this point we have accidentally reinvented the `Enumerable` extension methods `Where` and `Select`, so even these two methods can be removed.  We're left with the now-commonplace alternatives:

```cs
var valuableProducts = products.Where(IsValuable);
var formattedProducts = products.Select(Format);
```

Once you start writing in this style rather than the original imperative style, **the surface area of your code that corresponds with your actual goals increases compared to that of your infrastructure**.  The focus is on *what* your goal is, and not on *how* you intend to trick a box of wires into reaching that goal.
