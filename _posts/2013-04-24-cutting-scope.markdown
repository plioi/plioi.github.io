---
title: Cutting Scope
layout: post
---

Over the last week, I've implemented support for <code>async</code>/<code>await</code> in the [Fixie test framework](https://github.com/plioi/fixie). Thanks to a suggestion from [Pedro Reys](https://twitter.com/pedroreys), I found that this project was susceptible to a serious bug, one that NUnit and xUnit both encountered and addressed back when the <code>async</code>/<code>await</code> keywords were introduced in C# 5.

While developing the fix, I relearned an important lesson: cutting scope is not a sign of defeat. Sometimes less really is more.

## The Bug

With the bug in place, a test framework can report that a test has passed even when it should fail. Consider the following test fixture:

{% gist 5448833 %}

The developer of these tests should expect TestAwaitThenPass to be the only passing test. The other four tests should all fail with one exception or another. Unfortunately, *Fixie would claim that all 5 of these tests pass*. To make matters even more confusing, despite "passing", TestAsyncVoid's DivideByZeroException would still be output to the user.

When you call most async methods, the method call will not actually do the work. Rather, the method will quickly return a <code>Task</code> that *knows how* to do that work. To provoke the <code>Task</code> to execute, you must call its Wait() method. I was failing to call Wait(), so I would happily report success for a test that was never actually executed in full!

In the case of an <code>async void</code> method, calling the method *does* cause the work to take place, but the exception does not surface in the normal fashion. The test framework's own try/catch blocks won't catch it, and it will bubble all the way up before appearing in the output as an unhandled exception.

## The Initial Requirements

Once I could reproduce the problem, I came up with the first version of my new requirements. Since <code>async</code> methods must be declared to return <code>void</code>, <code>Task</code>, or <code>Task&lt;T&gt;</code>, and since all of these pose the same risk of the test passing when it shouldn't,
<ol>
<li>an <code>async Task</code> test method must be waited upon before deciding whether it passes or fails.</li>
<li>an <code>asyc Task&lt;T&gt;</code> test method must be waited upon before deciding whether it passes or fails.</li>
<li>an <code>async void</code> test method must be waited upon before deciding whether it passes or fails.</li>
</ol>
## The Easy Part

We want to do the extra work for methods declared with the <code>async</code> keyword, and fortunately we can detect that keyword using reflection. When you use this keyword, the compiled method gains an attribute available to us at runtime:

{% gist 5448836 %}

Before the fix, a test method would be executed via reflection like so:

{% gist 5448839 %}

We can fix the execution of <code>async Task</code> and <code>asyc Task&lt;T&gt;</code> by waiting for the returned <code>Task</code> to complete:

{% gist 5448843 %}

When a regular test fails, <code>method.Invoke(...)</code> throws. When an <code>async</code> test fails, <code>task.Wait()</code> throws.

## Unforeseen Complexity

The third requirement is problematic. If a test method is declared <code>async void</code>, <code>method.Invoke(...)</code> returns null, so we'll never see the <code>Task</code> object and will never be able to call <code>task.Wait()</code>.  It turns out there is an extremely complex workaround, implemented in NUnit, which takes advantage of implementation details surrounding <code>async</code>/<code>await</code> execution.  After researching the technique, I lacked confidence that I would use it correctly.

## The Actual Requirement

I started to question the train of thought which led to the original 3 requirements.  All async methods have to be declared as returning <code>void</code>, <code>Task</code>, or <code>Task&lt;T&gt;</code>, otherwise they won't compile, and **I was naively assuming that all three of these variations were good test declarations.**

It turns out that declaring methods <code>async void</code> is frowned upon for exactly the same reason they were giving me trouble: it is crazy weird and difficult to correctly wait on a <code>Task</code> when the <code>Task</code> itself is inaccessible to you! <code>async void</code> declarations say, "I want to fire and forget", but a test author does *not* want the test framework to forget what's going on! The only reason <code>async void</code> even *exists* is for a specific edge case: [async event handlers have no choice but to be declared void](http://stackoverflow.com/questions/8043296/whats-the-difference-between-returning-void-and-returning-a-task).

<blockquote>The *actual* requirement I needed to meet was to **provide accurate pass/fail reporting**: a test passes if and only if the test framework executes it in full without throwing exceptions.</blockquote>

In the case of <code>async void</code>, I satisfy *this* requirement by *slapping the test author's hand*. I fail such a test method immediately, without bothering to execute it. The failure message explains that "void" should be replaced with "Task". Requiring that the test author replace 4 characters with 4 characters, rather than encourage a bad habit of writing <code>async void</code> methods, is actually *better* than supporting all variations of <code>async</code> methods.

## Less is More

Requirements are human decisions based on incomplete information. With enough information, you may better-serve the needs of your system and its users by *not* doing something.

In this case, supporting all 3 kinds of asynchronous methods would have introduced a great deal of complexity and risk, and I have absolutely no interest in introducing complexity or risk into something as fundamental as a test framework. By treating <code>async void</code> methods as "real" test cases that always fail, I satisfy the requirement of providing accurate pass/fail reporting. By cutting scope, I'm providing a better solution.