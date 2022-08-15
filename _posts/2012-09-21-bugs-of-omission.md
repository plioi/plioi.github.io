---
title: "Bugs of Omission"
layout: post
---


When your software includes faulty logic or conflicting requirements, you're very likely to run into unexpected behavior.  These bugs are relatively easy to trigger and diagnose.

A bug of omission, on the other hand, in which you *should* have included code to cover an edge case, might sit around lurking for quite a while.  When such a bug finally does surface, the behavior may be so counter to your expectations  that you'll wonder if maybe, just maybe, a rogue pixie or trickster demigod has cursed your machine.

"This has been working *forever* and nobody has changed it!"  It can seem like things as fundamental as your language's own operators can no longer be taken for granted.  Of course, the bug later turns out to be the fault of mere mortals; a particularly rare set of circumstances reveals the gap that was always there.

Case in point: in <a href="http://patrick.lioi.net/2011/11/18/building-rich-enums/">Building Rich Enums</a> we saw the Headspring <a href="https://github.com/Headspringlabs/Enumeration">Enumeration</a> class as a rich alternative to the `enum` keyword.  This base class makes it easy to create a finite set of named instances of your class.  This week, *all of a sudden and without overt provocation*, equality comparisons between Enumerations stopped working for us on one project.

## This has been working *forever*...

Until this week, equality comparisons have worked perfectly fine for our Enumerations.  Consider a typical `Enumeration`:

```cs
public class Color : Enumeration<Color, int>
{
    public static Color Red = new Color(1, "Red");
    public static Color Blue = new Color(2, "Blue");
    public static Color Green = new Color(3, "Green");

    private Color(int value, string displayName) : base(value, displayName) { }
}
```

The constructor is private, so you will only be working with the three trusted instances.  If you store these to a database as their underlying integers, you can read them back out with the `FromInt32(int)` method to get a hold of the corresponding instance.  We even overrode the `Equals(object)` method just in case you ever passed these around as `object` instead of the concrete type and still needed to compare them.

The default behavior of the `==` operator has been enough to date, because reference equality has been a safe assumption: any instance you ever get your hands on will come from one of the three explicit `Color` constructor calls in the class definition... **but that's a lie** or, if we're being generous, an untruth of omission.

## ...*except* for this new set of circumstances.

This week, we started comparing instances of an `Enumeration` like `Color` and got unexpected results.  Instances that appeared to be the same were coming back as 'not equal'.  Stepping through with the debugger, we saw that the instances on the left and right of the `==` operator contained the same int Value, and the implementation of `Equals` was written to compare exactly those ints.  Where did we go wrong?

For the first time, we were making comparisons between instances after some instances were serialized-to-JSON and subsequently deserialized-from-JSON.  Deserializing an Enumeration from a string of JSON effectively bypasses all of C#'s "protective" measures like `private` constructors.  We were producing multiple distinct instances which happened to contain the same int Value, and our reliance on reference equality was no longer enough.

Fortunately, <a href="https://github.com/HeadspringLabs/Enumeration/commit/66afd7471f6588ce52653fa361c445815dc035fa">the fix was simple.</a>  We had omitted overloading the `==` operator to favor value equality over reference equality.  Now, whether your equal instances are literally the same object or not, you get the equality semantics you'd expect.
