---
title: Fixie's Life Bicycle
layout: post
---

Last week, we saw how the [Fixie test framework](https://github.com/plioi/fixie) gives you control over [test discovery](http://www.headspring.com/patrick/test-discovery/). This week, we'll see my first (admittedly rough) attempt at similarly giving you control over test *execution*. Let's start with a quick review of last week's test discovery feature, and then extend the example to demonstrate Fixie's treatment of test execution.

## Test Discovery (Again)

By default, Fixie uses a reasonable rule of thumb to determine which of your classes are test classes, and which of your methods are test methods. The default rules are implemented like so:

\[gist id="5675320"\]

Test classes are those whose name ends with "Tests".  Test case methods are those with zero parameters, declared to be either <code>void</code> or <code>async Task</code>.  In other words, if it looks like a test, it's a test.

When you wish to stray from these defaults, though, you can provide your own *convention* class: tell Fixie what your test classes and test methods *look like*, and it will gladly use your rule of thumb instead of the default. Last week, we introduced NUnit-style attributes and provided our own custom convention describing the treatment of those attributes:

\[gist id="5675323"\]

By stating that test fixtures are marked with \[TestFixture\] and test cases are marked with \[Test\], Fixie starts to use NUnit-style test discovery behavior.

## Test Discovery is Only Half the Battle

Implicit in the default convention is the notion that you will get a new instance of the test class *for each test method*. That rule matches xUnit, but differs from NUnit, in which you get one instance of the test class *shared* across all the test methods in that class. Using our custom convention, we're not quite behaving like NUnit.  If you wanted to do NUnit-style \[TestFixtureSetUp\] and \[TestFixtureTearDown\], you'd be surprised! Using the above custom convention, consider the following test fixture and its output under Fixie:

\[gist id="5675325"\]

\[gist id="5675329"\]

That's not at all like NUnit! Thankfully, our custom convention was honored so that only FirstTest() and SecondTest() are considered to be tests. Unlike NUnit, though, Fixie has completely neglected the per-test \[SetUp\]/\[TearDown\] and per-class \[TestFixtureSetUp\]/\[TestFixtureTearDown\].  On top of that, it has constructed a fresh instance of the class twice instead of once.

**Our custom convention is allowing us to stray from the defaults for test *discovery*, but so far we're still using Fixie's default test *execution* rules.**

## Customizing Test Execution

<blockquote>The functionality covered in this section is in its infancy and is likely to change in the short term, but serves to demonstrate the kind of customization I am shooting for.</blockquote>

Fixie's Samples project contains a more useful [NUnit look-alike convention](https://github.com/plioi/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie.Samples/NUnitStyle/CustomConvention.cs):

\[gist id="5675332"\]

Here, we see three new sections. First, we say that for each test fixture, create an instance per fixture class instead of creating an instance per test case. Second, for each test class instance, wrap the built-in behavior with calls to the \[TestFixtureSetUp\] and \[TestFixtureTearDown\] methods. Lastly, for each test case method, wrap the built-in behavior with calls to the \[SetUp\] and \[TearDown\] methods.

Armed with this new convention, running the sample test class confirms that we're now following the NUnit test fixture lifecycle:

\[gist id="5675335"\]

The FixtureExecutionBehavior you select in your convention is the key driving force affecting how your test classes will be executed. There are two built-in behaviors: [CreateInstancePerCase](https://github.com/plioi/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie/Behaviors/CreateInstancePerCase.cs), and [CreateInstancePerFixture](https://github.com/plioi/fixie/blob/cd85b7ddae14dbe7deb82d2070a314fd8d710819/src/Fixie/Behaviors/CreateInstancePerFixture.cs).

These two classes give Fixie a two-mode test lifecycle. A life-*bi*cycle if you will, [finally justifying the name beyond any doubt](http://en.wikipedia.org/wiki/Fixed-gear\_bicycle).