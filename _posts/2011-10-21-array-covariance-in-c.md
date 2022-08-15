---
title: "Array Covariance in C#"
layout: post
---


When we first learn about inheritance, we're given some plain-English examples with an emphasis on the phrase "is-a". A car *is-a* vehicle, so a `Car` class would inherit from a `Vehicle` class. The idea that class inheritance can so directly map to English is **seriously misleading** and can get you into trouble.

In English, it's fair to say that a bag of apples *is-a* bag of objects. It may have an odd sound to it, but it's true. This makes it very tempting to say that in C#, a `List<Apple>` should be assignable to a `List<Object>`, since all `Apple` instances are also `Object` instances:

```cs
List<Apple> apples = new List<Apple> { new Apple(), new Apple() };
List<Object> objects = apples;
```

The sample above won't compile. The compiler is protecting us from potential runtime type exceptions. If this compiled, we could get into trouble like so:

```cs
List<Apple> apples = new List<Apple> { new Apple(), new Apple() };
List<Object> objects = apples;
objects[0] = "This is not an apple.";
```

If this compiled, we would be able to attempt to put a string into a collection that simply does not support that operation. The English metaphor for inheritence (*is-a*) just doesn't adequately match the actual constraints of the type system. We're trying to violate the <a href="http://en.wikipedia.org/wiki/Liskov_substitution_principle">Liskov Substitution Principle</a>. If we declared that `List<Apple>` is a subtype of `List<Object>`, we'd be saying that *every* operation that acts on a `List<Object>` could be given a `List<Apple>`, and that's simply not true. In our example, the operations performed on the `List<Object>` included putting new arbitrary objects into the list, and you just can't do that to a `List<Apple>`.

**Because `List<Apple>` has more constraints than `List<Object>`, `List<Apple>` is not assignment-compatible with `List<Object>`.**

Unfortunately, the previous example **does** compile when you replace `List<T>` with supposedly-strongly-typed arrays. Instead of knowing that you've done something wrong at compile time, you get an exception at runtime:

```cs
Apple[] apples = new[] { new Apple(), new Apple() };
Object[] objects = apples;
objects[0] = "This is not an apple."; //Throws ArrayTypeMismatchException at runtime!
```

Because `TChild[]` can be assigned to `TParent[]` for any `TChild` inheriting from `TParent`, we can say "arrays in C# are covariant".

To sum up, `List<Apple>` can't be treated as `List<Object>` *because of the operations that `List<T>` provides*. The question, then, is could we ever safely take advantage of this convenient funky-casting for *other* generic types? The answer to this question falls under the category of "Covariance and Contravariance" and is provided by the C# 4 keywords `in` and `out`. In my next post, we'll cover an example where this funky-casting is both convenient and safe.
