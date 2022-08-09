---
title: 'What&#8217;s in a Name?'
layout: post
---
In my last post, we saw how Fixie lets you define [conventions for parameterized tests](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/). My goal was to provide a way for convention authors to tell Fixie what inputs should be passed into any given test method, allowing for potentially-many calls to a single test method. Each call can pass or fail independently.

I tried and failed to implement this feature a few times, but the actual solution was quite small. My difficulties can be traced back to a simple naming error I made early on. Today we&#8217;ll cover this stumbling block, and in my next few posts we&#8217;ll see the feature in action.

## False Equivalence

Before implementing parameterized tests, a basic convention might look like the following:

```cs
public class ShouldaConvention : Convention
{
    public ShouldaConvention()
    {
        Classes
            .Where(type => type.Name.StartsWith("When_");

        Cases
            .Where(method => method.Name.StartsWith("Should_"))
            .Where(method => method.IsVoid());
    }
}
```

The `Classes` and `Cases` properties held lists of conditions. `Classes` collected conditions describing test classes, and `Cases` collected conditions describing test cases. Armed with these descriptions, Fixie could search through your test assembly for things to execute.

The `Cases` property was defined like so:

```cs
public MethodFilter Cases { get; private set; }
```

Each method was a test case: a single thing that could pass or fail. The naming was weird, though. Why not `public CaseFilter Cases` or `public MethodFilter Methods`? The mismatch in naming should have jumped out at me, and maybe it did, but I downplayed it as &#8220;accurate today&#8221; and let it be. That poor naming, however, muddied the terms &#8220;test method&#8221; and &#8220;test case&#8221; in my head, and the terminology mix-up spread all over.

When I wanted to start working with parameterized tests, in which a single test method is potentially-many individual test cases, I was immediately running into barriers put up by the false equivalence between methods and cases. Shakespeare had it wrong &#8212; names matter. That scene should have gone more like this:

> **Romeo**: What&#8217;s in a name? A rose by any other name would&#8211;  
> **Juliet**: Stop right there. &#8220;What&#8217;s in a name!?&#8221; A MethodInfo by any other name could be mistaken for a Case and hamper your ability to reason about your own software, that&#8217;s what.  
> **Romeo**: I think we should see other people.

## The Fix: Name Things What They Are

Each time that I failed to implement this feature, I started in the wrong place. Before identifying the naming mismatch as the root cause of the problem, I was actually starting clear on the other side of the code at the point where the test method&#8217;s corresponding MethodInfo gets invoked. &#8220;I&#8217;ll just overload this to take in optional parameters, and then see what breaks, and go from there.&#8221; Dead end, because the real issue was far earlier in the process. I was treating the symptom and not the root cause.

The thing that finally got me moving in the right direction was to rename the darn Convention &#8220;Cases&#8221; property to &#8220;Methods&#8221;. Naming it what it _was_ finally made the next culprit stand out on the screen:

```cs
Case[] cases = Methods
                .Filter(testClass)
                .Select(x => new Case(testClass, x))
                .ToArray();
```

The .Select(&#8230;) call here finally stood out as fishy. With the old naming, it was natural that each of the methods found by the _Cases_ property should new up a Case object. With the new naming, I realized that _here_ was a place to decide _how many_ Cases needed to be created for each method found by the _Methods_ property.

In order to answer that question, I needed to introduce a new customization hook. As we saw yesterday, given a MethodInfo, a convention author can yield any number of object arrays, each of which is a set of parameters for a single call to the method. Yield multiple object arrays, get multiple test case pass/fails. The default, if the convention author doesn&#8217;t provide their own rule, is to simply call the method once with zero parameters.

Poor naming motivated a false equivalence between methods and test cases, which muddied the Case creation step and hid the need for this customization hook. With better naming, the need for this hook was clear. The actual commits were quite small:

  1. [Rename the offending property](https://github.com/fixie/fixie/commit/a2260e27efd6471d9fb1214721a12ced2ad2187a)
  2. [Construct many test Case objects with the user-provided Func.](https://github.com/fixie/fixie/commit/70691f241a48aafacdba48b705b72bea7a6e4269#diff-2)