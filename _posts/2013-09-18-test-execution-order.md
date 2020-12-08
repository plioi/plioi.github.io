---
title: Test Execution Order
layout: post
---
In a perfect world, the order of execution of your tests shouldn&#8217;t matter. If each test is isolated and makes no unfair assumptions about the state under test, they&#8217;ll behave the same no matter the order. In reality, we&#8217;re all humans working on tough problems. Sometimes we mess up, sometimes we make tradeoffs of a little shared state in exchange for practical test speeds, and sometimes the code _under_ test is brittle due to shared state like static fields.

Test execution order can reveal real problems. If your tests are brittle and run in the same order each time, you might not notice for a while. If your tests are brittle and run in randomized order, you might experience the pain earlier, giving you some pressure to improve the situtation right away. [xUnit once helped me catch a serious bug early](http://patrick.lioi.net/2012/10/25/avoid-mutation-by-default/), because it runs the tests within a test class in random order. If you&#8217;re not aware that your test runner shuffles its tests, though, you may be extra-confused during the debugging experience.

I see three possibilities:

  1. You don&#8217;t care what order your tests run in. Whatever order the test runner finds them in is good enough. This order is actually undefined by the reflection API, but tends to be the order the tests are declared.
  2. You deliberately want to shuffle the tests in order to catch issues earlier, and you accept that it&#8217;s up to you to keep that in mind when tests start failing inconsistently.
  3. You have an extremely unusual situation in which you want to order the tests in a test class by some other rule.

I recently added support for test execution order in the [Fixie test framework](https://github.com/fixie/fixie). By default, no order is enforced, meaning you get the undefined (but not very surprising) order provided by the reflection API. If you want to opt into random shuffling or apply your own rule for ordering tests, you can do so in your testing conventions. To randomize the order of test cases within a test class, declare that they should be shuffled:

{% gist 6602967 %}

To specify your own sorting rule, call SortCases instead of ShuffleCases, providing a rule for ordering any two test cases:

{% gist 6602950 %}

When Fixie tests itself, it needs to say things like &#8220;Run all of the tests in this sample test class and then assert on the results and the output across the whole test class.&#8221; It&#8217;s just easier to write these assertions in a small amount of code if I can assume a reliable order of execution, so I sort them by name. Rather than sorting, most users would likely do nothing or call ShuffleCases, but SortCases is there if you really, really want it.
