---
title: Generating Test Cases at Runtime
layout: post
---
Last time, we saw how Fixie can [integrate with AutoFixture](https://patrick.lioi.net/2013/10/05/autofixie/). That was a situation in which parameterized tests were meant to be called once. They were parameterized because the _producer_ of the inputs was interesting while the _count_ was uninteresting.

Before that, we saw how a convention could instead cause parameterized test methods to be called _multiple_ times, [using attributes as a source of multiple inputs.](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/) That example was a little stifling: you have to know at compile time how many scenarios you are testing, and you can only specify compile-time constants in attributes.

Today, we'll see an example of parameterized tests which are called multiple times, using values _generated_ at runtime. Consider a test class with two test methods and some static Person-generator methods:

{% gist 6894225 %}

Without a custom convention, Fixie fails to invoke the parameterized test:

{% gist 6894235 %}

Let's define a custom convention which considers single-argument test methods. For a test method that takes in some type T, it will find all the static methods returning IEnumerable<T>, call them all, and pass in the many T objects into the test method one at a time:

{% gist 6894324 %}

Now, we see that our two test methods produce 5 individual test cases.

{% gist 6894333 %}

> Oh dear. **Can you spot the bug?** It's possible to write test methods that never get invoked. In our next episode, we'll cover the bug as well as an improvement in Fixie that will prevent such subtle surprises while simplifying parameterized test conventions.