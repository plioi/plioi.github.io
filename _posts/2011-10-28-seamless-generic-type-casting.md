---
title: "Seamless Generic Type Casting"
layout: post
---


In <a href="http://patrick.lioi.net/2011/10/21/array-covariance-in-c/">Array Covariance in C#</a>, we saw that a `List<Apple>` cannot be assigned to a variable declared as `List<Object>`, even though all `Apple`s are `Object`s. If we were allowed to do that, we could get runtime type exceptions as we work with the list contents.

The compiler has to prevent us from these assignments because `List<T>`'s public interface includes some subtle constraints: `List<Apple>` is actually a *more constrained* type than `List<Object>`. You can put anything into a `List<Object>` and get it back out again. You can't put just anything into a `List<Apple>`; what would you expect to happen when you tried to read it back out? Therefore, `List<Apple>` is more constrained than `List<Object>` and we can't treat `List<Apple>` as a subtype of `List<Object>`.

Still, it's tempting to ask, is there some other generic type `Foo<T>` where `Foo<Apple>` *can* be treated as a `Foo<Object>`? **It turns out the answer is yes!**

Let's say your application is sending messages to another process over a message bus, and that you've implemented a helper method that sends a whole collection of messages instead of just one:

```cs
public interface IMessage { }

public class RemoveOrderLineItem : IMessage { }

public class MessageBus
{
    public void Send(IMessage message)
    {
        //Send the message.
    }

    public void SendAll(List<IMessage> messages)
    {
        foreach (var message in messages)
            Send(message);
    }
}

...

List<RemoveOrderLineItem> removals = new List<RemoveOrderLineItem>();
removals.Add(new RemoveOrderLineItem());
removals.Add(new RemoveOrderLineItem());
removals.Add(new RemoveOrderLineItem());

var bus = new MessageBus();

bus.SendAll(removals);
```

Unfortunately, this won't even compile. We're trying to send a list of `RemoveOrderLineItem` messages, but the `SendAll` method accepts a `List<IMessage>`. Fortunately, this starts to compile if we change the definition of `SendAll` to receive an `IEnumerable<IMessage>` instead of a `List<T>`:

```cs
public void SendAll(IEnumerable<IMessage> messages)
{
    foreach (var message in messages)
        Send(message);
}
```

What's so *special* about `IEnumerable<IMessage>` that lets us seamlessly cast from a `List<RemoveOrderLineItem>`? Remember, the reason that the compile originally failed had to do with the fact that the object we were passing to `SendAll` had more constraints than the type declared on the receiving end. With `IEnumerable<IMessage>`, it's a different story. Now, the only thing that `SendAll` can do with our collection is get values out of it, and we know all such values will be individually compatable with `IMessage`.

We're able to do this because, as of .NET 4, `IEnumerable<T>` has an updated definition:

```cs
public interface IEnumerable<out T> : IEnumerable
{
    IEnumerator<T> GetEnumerator()
}
```

The `out` keyword was added to the generic type parameter `T`. Using this keyword is a promise. **"I promise that for all the members in this interface, `T` will only appear in the return types, never appearing in any inputs."** When we use the `out` keyword, we're saying that T is "covariant".

The `in` keyword makes a similar promise: the type parameter `T` could appear in the types of input arguments, but never in the return types. When we use the `in` keyword, we're saying that `T` is "contravariant".

In addition to sanity checking your promise, the compiler is now free to treat any `IEnumerable<RemoveOrderLineItem>` as a `IEnumerable<IMessage>`, without having to cast each individual item first. `IEnumerable<RemoveOrderLineItem>` is truly a subtype of `IEnumerable<IMessage>`.

Introducing these two keywords to the language was a nonbreaking change, with the main effect being that people encountered fewer compiler errors, usually without knowing it. When you design your own libraries, consider whether your interfaces deserve to take part in this style of casting. Beware, though, that *removing* the keywords later on can be a breaking change from the point of view of your library's consumer!

In my next post, we'll see how these keywords can be used to glue together delegates.
