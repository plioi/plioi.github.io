---
title: DRY Test Inheritance
layout: post
---

Over the last two weeks, we've seen how [Fixie](https://github.com/plioi/fixie) can be configured to [mimic NUnit](http://www.headspring.com/fixies-life-bicycle/) and to [mimic xUnit](http://www.headspring.com/the-sincerest-form-of-flattery/).  That's a neat little trick, but doesn't provide much value.  This week, we'll see how Fixie's convention API can be used to *improve* upon NUnit.

<blockquote>Today's code samples work against [Fixie 0.0.1.62](http://nuget.org/packages/Fixie/0.0.1.62). The customization API is in its infancy, and is likely to change in the coming weeks.</blockquote>

Today's sample convention addresses two problems in NUnit:
<ol>
<li>Lifecycle attributes are redundant</li>
<li>Test class inheritance is needlessly complex.</li>
</ol>

## NUnit Lifecycle Attributes Are Redundant

If you use NUnit, you probably see a lot of test classes like this:

{% gist 5762364 %}

My \[SetUp\] methods are always named "SetUp", my \[TearDown\] methods are always named "TearDown", etc. It's annoying to sacrifice whole lines to that noise.  When 99% of your test fixtures use naming conventions like mine, the attributes stop telling you something.  These attributes start to fill the same role as excessive comments:

{% gist 5762368 %}

## NUnit Inheritance is Needlessly Complex

The use of attributes for these "lifecycle" hooks poses more serious problems when your test classes take part in inheritance.  Since they don't *have* to be placed on methods with the same name, you could have completely unrelated \[SetUp\]s, for instance, at different levels of the hierarchy.

What order do they run in? Should the child class's \[SetUp\] call the base?  Should the base \[SetUp\] call an abstract method you have to implement instead of providing your own \[SetUp\] in the child? \[SetUp\]s get complicated very quickly in the presence of inheritance.

The order of execution during test setup is important. How bizarre would it be if there were no guarantee about the order of *constructor* execution in a class hierarchy?  With NUnit lifecycle hooks, order becomes a problem.  Sure, NUnit has rules of its own for the order, **but it doesn't matter what they are** because even having to ask the question means it's already too complex. In addition, having more than one \[SetUp\] in the same level of the class hierarchy is allowed but ambiguous: there's no guarantee what order they'll run in. Worse yet, over the years I've seen the behavior differ across different test *runners*.

<blockquote>The preparation of state under test should be remarkably dull.  We're trying to confirm our assumptions about the behavior of our system, and we can't do so with confidence if we aren't confident about what all we've set up in the first place.</blockquote>

## A Low-Ceremony Alternative Convention

DRY stands for "Don't Repeat Yourself", not "\[DontRepeatYourself\] Don't Repeat Yourself"! Allowing redundancy has opened the door to complexity. Let's improve upon the NUnit style by defining a simpler, [low-ceremony test class convention](https://github.com/plioi/fixie/blob/a74078dfe3c8f415fd0663af104b75adfb90d29d/src/Fixie.Samples/LowCeremony/CustomConvention.cs) with Fixie:

{% gist 5762372 %}

Armed with this convention class in our test assembly, our original test class gets simpler:

{% gist 5762378 %}

The most relevant part of the convention says that, instead of using attributes, the lifecycle hook methods will be identified by their names:

{% gist 5762381 %}

## What Does This Convention Buy Us?

There are three benefits to this approach:

First, we don't waste time reminding the reader that "SetUp" is in fact spelled "SetUp".

Second, it's impossible to define more than one SetUp method in the same level of the class hierarchy, avoiding the ambiguity allowed by NUnit.

Third, if you do opt into test class inheritance, we get to take advantage of familiar language features. If the base class has a SetUp and the child class has a SetUp, you take advantage of the <code>virtual/override/base</code> keywords to remove all doubt about execution order.