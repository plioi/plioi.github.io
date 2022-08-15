---
title: "A Game of Throws"
layout: post
---


In <a href="http://patrick.lioi.net/2011/12/29/infinite-laziness/">Infinite Laziness</a>, we saw how the `yield` keyword lets us work with *lazy evaluation*, in which work is deferred and performed on demand once something else actually needs to consume the result.  Lazy evaluation with `yield` isn't free, though; it has some unfortunate consequences.

This week, we'll see how lazy evaluation affects exception handling.  Next week, we'll see how `yield` complects brevity with laziness, resulting in some undesirable patterns.

Let's say you're lazily producing a series of objects, but the creation of each object has a real potential to throw an exception.  The developer calling the lazy method knows that the operation could fail, and attempts to protect the rest of the system:

```cs
public IEnumerable<Grenade> SafelyBuildGrenades(int[] serialNumbers)
{
  try
  {
    return BuildGrenades(serialNumbers);
  }
  catch (ExplosionException ex)
  {
    //Sound the alarms, etc.
  }
}

private IEnumerable<Grenade> BuildGrenades(int[] serialNumbers)
{
  foreach (int serialNumber in serialNumbers)
    yield return BuildGrenade(serialNumber);
}

private Grenade BuildGrenade(int serialNumber)
{
  //Either return a new Grenade, or
  //throw a new ExplosionException.
}
```

Despite wrapping the dangerous method call with a `try/catch` block, callers of SafelyBuildGrenades are in for a surprise.  **The call to BuildGrenades will never throw an ExplosionException, while callers of SafelyBuildGrenades might set off a grenade just by looking at them!**

When you call a `yield`ing method, **no work is performed and the call returns immediately**.  It returns an object that *knows how* to produce the list of items on demand.  Those items won't be produced until someone starts looping through them.  In our example above, this quickly-made object is likewise immediately returned by SafelyBuildGrenades.  The *consumer* of the result of SafelyBuildGrenades is then free to start looping through it, at which point the code in BuildGrenades will start to execute.  Since an ExplosionException could only be thrown way up in that consumer's loop, the consumer's loop is where the exception will start moving up the call stack.

> `yield` is an exception smuggler.  `yield` hides your dangerous code in a harmless-looking package in order to sneak it past unsuspecting `try/catch` blocks.  The harmful contents can then be unpacked by the recipient.

When using lazy evaluation, which includes the now-ubiquitous IEnumerable extension methods (Select, Where, Take, Skip...), take care.  Work won't be performed until you begin to iterate through the results, so your exception handling will need to take place higher up the call stack.
