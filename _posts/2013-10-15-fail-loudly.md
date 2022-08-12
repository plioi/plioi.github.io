---
title: Fail Loudly
layout: post
---
I ended my last post, [Generating Test Cases at Runtime](https://patrick.lioi.net/2013/10/09/generating-test-cases-at-runtime/), with a pop quiz: &#8220;Can you spot the bug? It’s possible to write test methods that never get invoked.&#8221; Today, let's cover the bug as well as its fix.

## Reproducing the Issue

At the end of the last post, we had a working test class and a convention which could match up a static IEnumerable<T> generator with a test accepting a T. Each value yielded by the IEnumerable<T> would be passed to the test as a single pass-or-fail-result. It worked just fine for that example: TestWithNoParameters() was called once, and TestWithInterestingParameter(Person) was called once for every Person yielded by the static Person-generating methods.

If we add a third test method, though, one whose incoming T type doesn't happen to match any input-generating static methods, something weird happens:

{% gist 6994379 %}

{% gist 6994403 %}

That's terrible! Our new test doesn't pass, _and it doesn't even fail_. A test runner's prime directive is to provide reliable pass/fail results. Anything that is a test should have a result, and this new method is most certainly a test according to the convention's discovery rules:

{% gist 6994427 %}

## The Diagnosis

Over my last few posts, we've dealt with Fixie's parameterized tests hook: a Func<MethodInfo, IEnumerable<object[]>>. When you want to define the meaning of a parameterized test method, you must yield an object[] once for every time you want the test method to be called. Until now, I've been assuming that the convention author's Func will always yield at least once. We even saw an attempt to do so in last week's convention:

{% gist 6994443 %}

When a test method had no parameters at all, it would explicitly return a single empty object[], meaning &#8220;Call the test method once with no args.&#8221; FindInputs(&#8230;), on the other hand, could still yield zero object[], and thus zero requests to call the method. Here's a bug that causes silent failures, and the worst failure is the one that keeps on happening without anyone knowing. We often hear the advice to &#8220;fail fast&#8221;, but there's an implicit &#8220;&#8230;and fail as loudly as possible&#8221; in there, too.

## The Fix: Fail Loudly

In order to fail loudly, Fixie needed to [treat test methods with _unsatisfied_ parameters as _failures_](https://github.com/fixie/fixie/commit/006f3a74e0f47222e1e44b8f192d44d70172a788), with a failure message explaining that fact. A nice side effect is that it's easier for convention authors to do the right thing: yield when you have a meaningful input, don't when you don't, and let Fixie autofail any test method that still couldn't be called.

Our convention from last time gets simpler, now that we don't have a lurking edge case to care about, and the output now correctly complains about the new test method:

{% gist 6994516 %}

{% gist 6994541 %}

The fix is available in [Fixie 0.0.1.101](http://www.nuget.org/packages/Fixie/0.0.1.101).