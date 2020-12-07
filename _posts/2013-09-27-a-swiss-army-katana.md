---
title: A Swiss Army Katana
layout: post
---
Before now, test methods for the [Fixie test framework](https://github.com/plioi/fixie/) had to have zero parameters. If your test method had a parameter, it would fail without being called. Fixie would have no idea what values to pass in. As of [Fixie 0.0.1.98](http://www.nuget.org/packages/Fixie/0.0.1.98), you can define your own conventions for parameterized tests. As a convention author, you decide what it means for a test to have parameters. For example, let&#8217;s say you want your parameter values to come from attributes on the method, similar to [xUnit theories](http://stackoverflow.com/a/9110623):

{% gist 6721858 %}

Our intention is for these 2 test _methods_ to be treated as 5 test _cases_, producing 5 individual pass/fail results. Out of the box, Fixie has no idea what [Input] means. In order to let Fixie know about our intentions, we can define the attribute and a custom convention:

{% gist 6721865 %}

{% gist 6721871 %}

Your own convention wouldn&#8217;t have to be attribute-based. All that Fixie cares about is that you provide it some `Func<MethodInfo, IEnumerable<object[]>>`. That&#8217;s a mouthful, so let&#8217;s break it down:

  1. Parameters(&#8230;) accepts a function that explains what inputs to use for any given test method.
  2. Your function is given the MethodInfo that describes a single test method.
  3. Your function yields any number of object arrays.
  4. Each object array that you yield represents a single call to the test method. If the method takes in 3 parameters, your arrays better have 3 values with corresponding types.

`Func<MethodInfo, IEnumerable<object[]>>` is a Swiss Army Katana. It&#8217;s versatile, but sharp. It will enable us to do a wide variety of things, but it&#8217;s easy to misuse. It represents exactly what the .NET reflection API needs in order to call the method, so no matter what sugar I layer on top of it, `Func<MethodInfo, IEnumerable<object[]>>` is the truth under the hood. After developing a few more examples, it&#8217;ll be more clear how a few convenient overloads of the `Parameters()` method could make it easier to get things right in the most common situations.

In my next post, we&#8217;ll take a little detour to see why such a small change was so hard to implement.