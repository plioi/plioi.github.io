---
title: Nailing Down Generics
layout: post
---
## Calling Generic Methods

You write a class containing a harmless generic method:

```cs
public class Thing
{
    public void GenericMethod<T>(T input)
    {
        Console.WriteLine(typeof(T).Name);
    }
}
```

Actually, for the purposes of this discussion, this is **not** one method declaration. Rather, it is an infinite number of method declarations:

  * `void GenericMethod<int>(int input)`
  * `void GenericMethod<string>(string input)`
  * `void GenericMethod<Customer>(Customer input)`
  * ...

Each time you make a call, the compiler decides which of the infinite methods you really meant. It makes this decision each time you call something called "GenericMethod". It makes this decision based on the compile-time types at each call site:

```
var instance = new Thing();

instance.GenericMethod(1); //Compile-time type is Int32, so prints "Int32".
instance.GenericMethod(true); //Compile-time type is Boolean, so prints "Boolean".
instance.GenericMethod("Cantaloupe Pantaloons"); //Compile-time type is String, so prints "String".
```

The compiler knows the compile-time type of each argument, and compares this to the generic type definition in order to pick the single winning "specific" method.

Easy, right? C# 101 stuff.

## Calling Generic Methods Via Reflection

Let's try to call the same method, with the same inputs, via reflection:

```cs
var instance = new Thing();

var method = typeof(Thing).GetMethod("GenericMethod");

method.Invoke(instance, new object[] { 1 });
method.Invoke(instance, new object[] { true });
method.Invoke(instance, new object[] { "Cantaloupe Pantaloons" });
```

MethodInfo.Invoke(...) wants you to throw an object[] at the method. It wants to take the first item of that array for the first parameter of the method, the second item for the second parameter, etc. 

Easy, right? Wrong. Each call to Invoke above would throw a System.InvalidOperationException with the message:

> Late bound operations cannot be performed on types or methods for which ContainsGenericParameters is true.

This exception message achieves the high honor of being both 100% accurate and 10% helpful. "Late bound operation" is compiler-speak for "something that is figured out dynamically at runtime instead of statically at compile time." The figuring out happens _later_ than compilation. The late bound operation, the thing we're trying to _do_ dynamically at runtime, is the method invocation itself.

Translation: the exception message is saying, "You cannot invoke the method because it _still_ has generic parameters." In order to invoke it, we're going to have to first nail down the generic parameters to specific concrete types so that they won't be generic anymore. We asked .NET to do the impossible:

> You: "Please call this method."
> 
> .NET: "No. That is not a method. It is an infinite number of methods. Try again, and tell me which one you meant."

## Once More, With Feeling

Lets narrow things down, each time we want to call the method via reflection. _Just like the compiler did for us in the original compile-time example_:

```cs
var instance = new Thing();

var method = typeof(Thing).GetMethod("GenericMethod");

var methodOfInt32 = method.MakeGenericMethod(typeof(int));
methodOfInt32.Invoke(instance, new object[] { 1 });

var methodOfBoolean = method.MakeGenericMethod(typeof(bool));
methodOfBoolean.Invoke(instance, new object[] { true });

var methodOfString = method.MakeGenericMethod(typeof(string));
methodOfString.Invoke(instance, new object[] { "Cantaloupe Pantaloons" });
```

This time, thankfully, we get the same output as the original plain calls to GenericMethod<T>(T). As with the original example, we know _exactly which 3_ of the _infinite_ methods are being called.

## Generic Test Methods

Let's say you're using the Fixie test framework and you have [defined an [Input] attribute with an associated Fixie Convention in order to have parameterized tests](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/). If one of your test methods is generic, Fixie faces the same problem we faced above:

```cs
public class Tests
{
    [Input(1)]
    [Input(true)]
    [Input("Cantaloupe Pantaloons")]
    public void GenericTestMethod<T>(T input)
    {
        Console.WriteLine(typeof(T).Name);
    }
}
```

Fixie calls test methods via reflection, using an object[] of inputs. In this case, the object[] has length 1, and the values come from the [Input] attributes. In order to successfully invoke the MethodInfo, Fixie must also call MethodInfo.MakeGenericMethod(...), passing in the right concrete Type, in order to get a handle on the specific, concrete version of the MethodInfo. Finally, Fixie can invoke _that_ MethodInfo.

Thanks to [Anders Forsgren](https://github.com/andersforsgren), Fixie can handle tests like this one. If the generic type parameter T can be nailed down to something unambiguously specific, it will be. If a generic test is declared with multiple generic type parameters like T1, T2, etc, it'll try to nail them all down. When there's any ambiguity for a T, though, Fixie has to assume `object` as the most specific type possible.

In short, Fixie will do the most reasonable thing possible without you ever needing to know that any of the above is happening.