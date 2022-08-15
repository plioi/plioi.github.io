---
title: "Overloading on Return Types"
layout: post
---


When you define two methods with the same name in C#, they have to take in different argument types in order for them to be distinguished from each other.  Sometimes, I find myself wishing that you could also overload a method name when the inputs are the same, but return types differ.

Technically, you can't do that in C#, but you can fake it.

## What would that even *mean*?

Let's imagine a hypothetical C# in which you were allowed to overload on return types.  Consider the following class:

```cs
public class Foo
{
  public int GetValue() { return 0; }
  public bool GetValue() { return true; }
}

//Later, attempt to call one of these methods:
foo.GetValue();
```

In the call to `foo.GetValue()`, we cannot tell which method should be called.  However, what if we tried to call it with some more context?

```cs
int i = foo.GetValue();
bool b = foo.GetValue();
```

When we have some local type information available, we really could make an unambiguous decision about which overload to call.  When initializing `i`, the only overload that could work is the one returning an `int`.  When initializing `b`, the only overload that could work is the one returning a `bool`.

We could conceivably find the right overload to call *based on how the return value is used*.  That seems odd at first, but there's nothing really Earth-shattering about it.  Even in today's real C#, overloads are picked based on what call-site types we happen to be using, and this would simply be yet another bit of call-site type information at our disposal.  Also, today's real C# figures out the type of a lambda expression based on how it's *used* as well, so there's a precedent for being able to use usage-site context.

## Using 'out' Parameters

Let's rewrite the original invalid class, using 'out' parameters instead of return types:

```cs
public class Foo
{
  public void GetValue(out int result) { result = 0; }
  public void GetValue(out bool result) { result = true; }
}

//Later...
int i;
bool b;
foo.GetValue(out i);
foo.GetValue(out b);
```

This is completely valid.  **When picking the right overload to call, the compiler considers 'out' parameters as if they were inputs.**

> There is no substantive difference between a method that returns some type `T`, and a `void` method with an 'out' parameter of type `T`.  The difference is only skin deep.

## A Concrete Example

I've encountered this in the real world on a few projects, dealing with the unfortunate interface `IDataReader`.  This is the interface used to process low-level ADO.NET result sets one row at a time.  Nowadays, most of us access databases with higher-level tools like NHibernate or PetaPoco, but this interface serves as a good example because:

1. It was written before .NET gained generics, so it has some awkwardly-named methods we'd like to work around.
2. Using it leads you to write hard-to-read, monotonous code.


Both of these issues will be resolved with the 'out' keyword.

This interface exposes many methods that take in an `int` and return a value.  The input `int` indicates which column of the result set you are interested in, and the method *name* says what type you hope to find in that column.  `reader.GetInt32(2)` gives you the `int` found in the third column, `reader.GetDataTime(3)` gives you the `DateTime` found in the fourth column, etc.  To further complicate matters, you shouldn't call these methods when the result set contains the special `DBNull` value.  When using this interface, you end up writing ugly things like this:

```cs
Guid id = reader.GetGuid(0);
int? nullableInt = reader.IsDBNull(1) ? null : reader.GetInt32(1);
DateTime time = reader.GetDateTime(2);
```

Note that every time you want call one of these Get* methods, you answer two questions:

1. What type do I expect to find at the given position?
2. Are null values possible at the given position?


Note also that in each line, both of these questions are answered by the declared type of the left-hand-side.  Why should we have to repeat those answers on the right-hand-side?  Armed with some extension methods and the 'out' keyword, things get much less repetative:

```cs
public static class DataReaderExtensions
{
    public static void GetValue(this IDataReader reader, int ordinal, out Guid value)
    {
        value = reader.GetGuid(ordinal);
    }

    public static void GetValue(this IDataReader reader, int ordinal, out int value)
    {
        value = reader.GetInt32(ordinal);
    }

    public static void GetValue(this IDataReader reader, int ordinal, out int? value)
    {
        if (reader.IsDBNull(ordinal))
            value = null;
        else
            value = reader.GetInt32(ordinal);
    }

    public static void GetValue(this IDataReader reader, int ordinal, out DateTime value)
    {
        value = reader.GetDateTime(ordinal);
    }

    //etc...
}

//Later...

Guid id;
int? nullableInt;
DateTime time;

reader.GetValue(0, out id);
reader.GetValue(1, out nullableInt);
reader.GetValue(2, out time);
```

When you're reading in 10 or so columns, the brevity adds up.  When writing a method that needs to read in a record from an `IDataReader`, you are *thinking* "I expect to get back values with these names, at these positions, with these types".  You are *not* thinking "I expect to pull a Guid out of the first position so I'll GetGuid, and I expect to pull an int out of the second position so I'll GetInt32 but watch out for DBNull, and...".  By "overloading on the return type", we get to say what we mean *and nothing else*.

I rarely use 'out' parameters, but it's nice to know they actually have a purpose every now and again, when the need presents itself.
