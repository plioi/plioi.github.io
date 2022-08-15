---
title: "Anonymous Function Arguments"
layout: post
---


A lambda expression is also called an "anonymous function" as it has no official name of its own: `x => x + 1` does some work, but there's no declared name for that function.  Anonymous functions are powerful and expressive, and are now common in .NET projects.

Anonymous function *arguments*, on the other hand, are weak and inexpressive.  Since the advent of the `Func` and `Action` delegate types, anonymous arguments have unfortuanately become as common as anonymous functions.  That is a trend I'd like to quash ASAP.

## Sneaky Ambiguity

What do I mean by "anonymous function arguments"?  Let's say someone else has written a method that takes in a `Func` and then invokes it as part of its own work:

```cs
public class Simulator
{
  public void ReticulateSplines(Func<int, bool, IList<Spline>> generateSplines)
  {
    int i = SomeMysteriousInt();
    bool b = SomeMysteriousBool();

    var splines = generateSplines(i, b);

    foreach (var spline in splines)
       Reticulate(spline);
  }

  ...
}
```

Let's say you really need to reticulate some splines, and you have a general idea of what function you should be passing into the `ReticulateSplines` method, but you don't happen to really know what the int and bool arguments are supposed to signify.  You confidently put your keyboard to work and get as far as the open-parenthesis in but a few keystrokes:

```cs
simulator.ReticulateSplines(
```

Alas, your stalled and blinking cursor finds your youthful confidence quaint but ultimately futile.  Visual Studio isn't going to be of any use here.  The most you'll find out from Intellisense is that you need to supply some function that takes in an int and a bool.  `Func`'s declared argument names are *arg1* and *arg2*.  These names are so useless that they might as well not exist.  Hence, "anonymous" function arguments.

`ReticulateSplines` has a descriptive name for its *own* argument (`generateSplines`), but has provided no clue as to what the arguments to this `Func` are.  If I started writing methods with parameters named *arg1*, *arg2*, *arg3*... I'd be immediately despised by every other developer who had to use my code.  Why, then, should I be able to get away with expecting those same developers to make sense of delegate arguments that have equally-ambiguous names?

## Getting Specific

The `delegate` keyword gets used less and less now that `Func` and `Action` are used for everything.  It's time to dust it off and put it to work again.  When faced with the problem of anonymous function arguments, and you can resolve the problem with a simple delegate declaration.

I recently needed to call a method that accepted an `Action<Action, Action>`.  I had a rough idea of what such an action needed to accomplish, but even after reading the body of the method in question I was unsure of the details.  The two anonymous arguments (the two inner actions) were passed around a bit before they were actually used.  Thankfully, someone had successfully called this method before, and after looking at the concrete method declaration that was being used in that case, I had a real clue what *arg1* and *arg2* were supposed to be called.

The fix was to introduce a more useful yet equivalent delegate type.

```cs
public delegate void SavePromptContinuation(Action actionUponSuccessfulSave, Action actionUponCancellation);
```

Now, I could replace all usages of `Action<Action, Action>` with `SavePromptContinuation`.  Back in my own code, Intellisense started offering the help I needed, and more importantly started offering the help that the next developer would inevitably need.  When trying to call a method that accepts one of these, ReSharper even offers to write the method stub for the specific delegate expected, complete with well-named parameters!

`Func` and `Action` are useful because they are *at hand*, but the other edge of that sword may just cut your fellow developers.  Here's some advice for their use:

1. Use `Func` and `Action` when writing something that is inherently so abstract that you can't be more specific.
2. You are not writing something that abstract.
