---
title: "The Compiler's Treatment of LINQ Syntax"
layout: post
---


When LINQ first became available, it was described as having a "lazy evaluation model", meaning that a query would not actually be executed until you started to look at the result. Iterating through a result, for instance, would finally cause the query to actually run. This gives you the power to piece together a query over multiple statements before letting it execute.

Recently, I found that *this abstraction can leak*. I had a set of queries that were misbehaving, and it seemed that a bug would be reproducible in one set of queries but not in another, even though they were remarkably similar to each other in structure. The culprit was a big surprise: it turns out that **the first 'from' clause in a query is treated differently from subsequent 'from' clauses**. Although my project's 'from' clauses were not implemented with side effects, an example that does use side effects best illustrates what is going on. Consider the following:

```cs
class Program
{
   static void Main()
   {
      var result = from x in FirstClause()
                     from y in SecondClause()
                     from z in ThirdClause()
                     select x + y;

      Console.WriteLine("Before Loop");

      foreach (var item in result)
            Console.WriteLine(item);

      Console.ReadLine();
   }

   private static IEnumerable<int> FirstClause()
   {
      Console.WriteLine("First Clause Side Effect");
      return new[] { 1 };
   }

   private static IEnumerable<int> SecondClause()
   {
      Console.WriteLine("Second Clause Side Effect");
      return new[] { 2 };
   }

   private static IEnumerable<int> ThirdClause()
   {
      Console.WriteLine("Third Clause Side Effect");
      return new[] { 3 };
   }
}
```

If we trusted the explanations of LINQ queries as having a lazy evaluation model, we would expect the output to be:

```
Before Loop
First Clause Side Effect
Second Clause Side Effect
Third Clause Side Effect
3
```


We expect this because we're told that the query doesn't start doing any work until we start iterating throught the results. Surprisingly, the actual output is:

```
First Clause Side Effect
Before Loop
Second Clause Side Effect
Third Clause Side Effect
3
```


Yikes. **The SQL-like query syntax is an abstraction over plain old method calls, and that abstraction is leaking.** To make sense of what is going on, it is helpful to know what the compiler does every time it encounters query syntax. Before the compiler even bothers to determine whether your query is actually valid, it performs a plain text transformation to a series of specially-nested method calls. Only after that plain text transformation will it actually determine whether the methods being called exist, that the arguments passed to them are of the right type, etc. ReSharper can actually perform that transformation for you. When we do that to the query above, we get the following:

```cs
var result =
    FirstClause()
      .SelectMany(x =>
        SecondClause(), (x, y) =>
          new { x, y }).SelectMany(@t => ThirdClause(), (@t, z) =>
            @t.x + @t.y);
```

A query with multiple 'from' clauses is transformed into a series of nested method calls. Lambda expressions are used to introduce a new local scope for the identifiers x, y, and z. SelectMany is an extension method, so the specific implementation is irrelevant during the plain text transformation. In other words, you can define the query syntax yourself, based on the type of the object you are selecting from (in this case, we're using the default `IEnumerable<T>` implementation of SelectMany).

**The right hand side of the first from clause is evaluated immediately. Subsequent from clauses are evaluated lazily, because only they appear on the right hand side of a lambda operator `=>`.**

By better-understanding what the compiler is actually doing with our LINQ queries, we can avoid frustrating surprises when the abstraction starts leaking.
