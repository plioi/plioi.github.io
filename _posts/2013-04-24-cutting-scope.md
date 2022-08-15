---
title: "Cutting Scope"
layout: post
---


Over the last week, I've implemented support for `async`/`await` in the <a href="https://github.com/fixie/fixie">Fixie test framework</a>. Thanks to a suggestion from <a href="https://twitter.com/pedroreys">Pedro Reys</a>, I found that this project was susceptible to a serious bug, one that NUnit and xUnit both encountered and addressed back when the `async`/`await` keywords were introduced in C# 5.

While developing the fix, I relearned an important lesson: cutting scope is not a sign of defeat. Sometimes less really is more.

## The Bug

With the bug in place, a test framework can report that a test has passed even when it should fail. Consider the following test fixture:

```cs
public class AsyncTests
{
    public async Task TestAwaitThenPass()
    {
        var result = await Divide(15, 5);

        result.ShouldEqual(3);
    }

    public async Task TestAwaitThenFail()
    {
        var result = await Divide(15, 5);

        result.ShouldEqual(0);
    }

    public async Task TestAwaitOnTaskThatThrows()
    {
        await Divide(15, 0);
    }

    public async Task TestFailBeforeAwait()
    {
        ThrowException();

        await Divide(15, 5);
    }

    public async void TestAsyncVoid()
    {
        await Divide(15, 0);
    }

    private static Task<int> Divide(int numerator, int denominator)
    {
        return Task.Run(() => numerator / denominator);
    }

    private static void ThrowException()
    {
        throw new Exception();
    }
}
```

The developer of these tests should expect TestAwaitThenPass to be the only passing test. The other four tests should all fail with one exception or another. Unfortunately, *Fixie would claim that all 5 of these tests pass*. To make matters even more confusing, despite "passing", TestAsyncVoid's DivideByZeroException would still be output to the user.

When you call most async methods, the method call will not actually do the work. Rather, the method will quickly return a `Task` that *knows how* to do that work. To provoke the `Task` to execute, you must call its Wait() method. I was failing to call Wait(), so I would happily report success for a test that was never actually executed in full!

In the case of an `async void` method, calling the method *does* cause the work to take place, but the exception does not surface in the normal fashion. The test framework's own try/catch blocks won't catch it, and it will bubble all the way up before appearing in the output as an unhandled exception.

## The Initial Requirements

Once I could reproduce the problem, I came up with the first version of my new requirements. Since `async` methods must be declared to return `void`, `Task`, or `Task<T>`, and since all of these pose the same risk of the test passing when it shouldn't,

1. an `async Task` test method must be waited upon before deciding whether it passes or fails.
2. an `asyc Task<T>` test method must be waited upon before deciding whether it passes or fails.
3. an `async void` test method must be waited upon before deciding whether it passes or fails.


## The Easy Part

We want to do the extra work for methods declared with the `async` keyword, and fortunately we can detect that keyword using reflection. When you use this keyword, the compiled method gains an attribute available to us at runtime:

```cs
public static class ReflectionExtensions
{
    public static bool IsAsync(this MethodInfo method)
    {
        return method.GetCustomAttributes<AsyncStateMachineAttribute>(false).Any();
    }
}
```

Before the fix, a test method would be executed via reflection like so:

```cs
MethodInfo method = /* The Test Method */;
method.Invoke(fixture.Instance, null);
```

We can fix the execution of `async Task` and `asyc Task<T>` by waiting for the returned `Task` to complete:

```cs
MethodInfo method = /* The Test Method */;
object result = method.Invoke(fixture.Instance, null);

if (method.IsAsync())
{
    var task = (Task)result;
    task.Wait();
}
```

When a regular test fails, `method.Invoke(...)` throws. When an `async` test fails, `task.Wait()` throws.

## Unforeseen Complexity

The third requirement is problematic. If a test method is declared `async void`, `method.Invoke(...)` returns null, so we'll never see the `Task` object and will never be able to call `task.Wait()`.  It turns out there is an extremely complex workaround, implemented in NUnit, which takes advantage of implementation details surrounding `async`/`await` execution.  After researching the technique, I lacked confidence that I would use it correctly.

## The Actual Requirement

I started to question the train of thought which led to the original 3 requirements.  All async methods have to be declared as returning `void`, `Task`, or `Task<T>`, otherwise they won't compile, and **I was naively assuming that all three of these variations were good test declarations.**

It turns out that declaring methods `async void` is frowned upon for exactly the same reason they were giving me trouble: it is crazy weird and difficult to correctly wait on a `Task` when the `Task` itself is inaccessible to you! `async void` declarations say, "I want to fire and forget", but a test author does *not* want the test framework to forget what's going on! The only reason `async void` even *exists* is for a specific edge case: <a href="http://stackoverflow.com/questions/8043296/whats-the-difference-between-returning-void-and-returning-a-task">async event handlers have no choice but to be declared void</a>.

> The *actual* requirement I needed to meet was to **provide accurate pass/fail reporting**: a test passes if and only if the test framework executes it in full without throwing exceptions.

In the case of `async void`, I satisfy *this* requirement by *slapping the test author's hand*. I fail such a test method immediately, without bothering to execute it. The failure message explains that "void" should be replaced with "Task". Requiring that the test author replace 4 characters with 4 characters, rather than encourage a bad habit of writing `async void` methods, is actually *better* than supporting all variations of `async` methods.

## Less is More

Requirements are human decisions based on incomplete information. With enough information, you may better-serve the needs of your system and its users by *not* doing something.

In this case, supporting all 3 kinds of asynchronous methods would have introduced a great deal of complexity and risk, and I have absolutely no interest in introducing complexity or risk into something as fundamental as a test framework. By treating `async void` methods as "real" test cases that always fail, I satisfy the requirement of providing accurate pass/fail reporting. By cutting scope, I'm providing a better solution.
