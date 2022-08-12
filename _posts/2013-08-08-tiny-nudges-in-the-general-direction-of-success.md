---
title: Tiny Nudges in the General Direction of Success
layout: post
---
Yesterday, I described a [leaky abstraction](https://patrick.lioi.net/2013/08/07/the-leakiest-abstraction/) in the [Fixie](https://github.com/fixie/fixie) test framework. I introduced a complex and viral solution to a simple and localized problem. Eventually, it caused a bug: consumers who customized Fixie in _just_ the right way could experience apparently-passing tests even though exceptions were thrown during test setup. [<img src="/images/2013/08/fix-a-flat.png" alt="" title="fix-a-flat" width="133" height="264" style="float:right;margin:5px;" />](/images/2013/08/fix-a-flat.png)That unacceptable behavior had to be fixed. To make matters worse, my test coverage wasn't enough to catch the bug; all of Fixie's self-tests were green. Today, I'll share the approach I took to safely mend Fixie's flat tire.

We use test frameworks so that we can make changes with confidence. I've used Fixie to test Fixie, in order to give me the confidence to make changes. A fundamental bug amidst all-green tests was a slap to the face. **I immediately lost confidence in my ability to make changes, yet I had to make a significant change.**

The solution came in several small steps over the course of a week:

  1. Add legitimate test coverage, revealing current behavior including the bug. The build fails.
  2. Apply a local fix to the immediate bug. The build passes.
  3. Find and clean up the smallest part of the ugly pattern that could be phased out on its own, while keeping all the tests green.
  4. GOTO 3

## Legitimate Test Coverage

Because my current test coverage was no longer giving me confidence in Fixie's most fundamental behaviors, I needed to start by shoring up my tests. I needed test coverage which would confirm the current correct behavior while also exposing the bug.

The features in question all had to do with treatment of a test class's _lifecycle_: end users can customize test class construction; customize frequency of construction; and wrap behaviors around each test method, around each test class instance, and around each test class.

The customization aspect led to my insufficient test coverage. I offer several degrees of freedom, but I need to promise that we'll do something reasonable when end users _exercise_ that freedom. There are a lot of combinations you can produce with those building blocks. &#8220;Create an instance once per test, wrapping each test in a setup/teardown,&#8221; &#8220;Create an instance once per test class, using a custom factory method, wrapping each instance in a fixture setup/teardown and each test in a transaction,&#8221;&#8230;

Testing these lifecycles gets extra interesting when you want to prove what happens when an end user's own test code fails. What happens when a test class constructor fails? When a custom factory fails? When a setup fails? When test teardown fails _and_ class teardown fails _and_ disposal fails?

The number of combinations grows quickly, and I wanted to test them all. Normally, testing _everything_ has diminishing returns, but these combinations are the whole point of this project, so they deserve it. Also, the problem I wanted to solve deals with my exception handling scheme, which directly affects the test class lifecycle.

I mitigated the combinatorial explosion by recognizing that if I covered _some_ features (like construction and custom factories) with a _few_ targeted tests, I wouldn't have to bother re-testing those in combination with _all_ the other scenarios. Still, I was left with many scenarios to test.

My test classes work with a nested hypothetical end user test class, containing a passing test and a failing test of its own:

```cs
class SampleTestClass : IDisposable
{
    bool disposed;

    public SampleTestClass()
    {
        WhereAmI();
    }

    public void Pass()
    {
        WhereAmI();
    }

    public void Fail()
    {
        WhereAmI();
        throw new FailureException();
    }

    public void Dispose()
    {
        if (disposed)
            throw new ShouldBeUnreachableException();
        disposed = true;

        WhereAmI();
    }
}
```

The WhereAmI() helper method simply writes the member to the console. The constructor writes &#8220;.ctor&#8221;, the Pass() method writes &#8220;Pass&#8221;, etc. From the outside, I can force any of these members to deliberately throw some generic exception, in order to prove what happens when end users' test classes throw exceptions at any step.

When Fixie tests itself, it executes the full lifecycle of this sample test class, given an example customization of the lifecycle. All console output is captured so that we can assert on the exact order the members were hit. The overall results of the mini test run are also collected, so we can assert on which tests passed, which tests failed, and which exceptions were the cause of each expected test failure. It's a little mind-bendy. Fixie's own tests fire up test runs of this hypothetical mini test class. This way, I can make positive assertions about what happens when _your_ test classes go negative.

Finally, I had [meaningful coverage of the test class lifecycle](https://github.com/fixie/fixie/tree/d3cc2fd1e2092bbcdc464d172a8ca5344a175ec9/src/Fixie.Tests/Lifecycle). I had confidence that I could dramatically change my exception handling scheme without breaking all of these user-facing promises.

## The Fix: Preserving Stack Traces

Yesterday's mess resulted from the desire to rethrow the InnerException within a TargetInvocationException:

```cs
try
{
    method.Invoke(instance, null);
}
catch (TargetInvocationException ex)
{
    throw ex.InnerException;
}
```

Alas, rethrowing like this destroys the stack trace we want to report to the user, so I had done this:

```cs
ExceptionList exceptions = new ExceptionList();

try
{
    method.Invoke(instance, null);
}
catch (TargetInvocationException ex)
{
    exceptions.Add(ex.InnerException);
}
catch (Exception ex)
{
    exceptions.Add(ex);
}

return exceptions;
```

Returning the ExceptionList to the caller started the complexity virus I described yesterday. The real fix is to wrap-and-rethrow within an exception type devoted to this purpose, [PreservedException](https://github.com/fixie/fixie/blob/d3cc2fd1e2092bbcdc464d172a8ca5344a175ec9/src/Fixie/PreservedException.cs):

```cs
try
{
    method.Invoke(instance, null);
}
catch (TargetInvocationException ex)
{
    throw new PreservedException(ex.InnerException);
}
```

When I later catch exceptions, I do have to take care to unwrap the OriginalException before reporting to the user. Why not just let the TargetInvocationException throw to that same catch block? If my end-of-the-line catch block sees a TargetInvocationException, I don't _know today and forevermore_ that it came from _this_ call to MethodInfo.Invoke(). If I catch a PreservedException, though, I _know_ that it is mine and that I should unwrap it.

This approach is structurally similar to NUnit. xUnit appears to poke a private field on the Exception type instead. Still, we can do even better. In yesterday's post, super-helpful commenter jdogg13 alerted me to the new ExceptionDispatchInfo type, which can rethrow an exception while preserving its stack trace. Fortunately all my work here doesn't go to waste. My solution will simply get shorter and simpler: the innermost catch will use ExceptionDispatchInfo, and the end-of-the-line catch block won't have to look for PreservedException as a special case.

## Tiny Nudges

At this point, I had meaningful coverage and a fix to apply at the deepest part of the mess. Still, I took several small steps to apply it. I identified the smallest place where I could partially apply the fix: exception handling around the attempt to Dispose() an instance of a test class. All the tests passed, so I grabbed the next smallest place, and repeated until I was finished:

  1. [Fix test class disposal](https://github.com/fixie/fixie/commit/c9d8abce8cf662c0e394b4b80b76c33e8971a33c). Tests pass. Commit
  2. [Fix test class construction](https://github.com/fixie/fixie/commit/f0bc304b1a77725be08a0edf889806979dc89ae3). Tests pass. Commit
  3. [Fix test method invocation](https://github.com/fixie/fixie/commit/3c82276eb83a6b1f9bb0d643e544785d1ef21eeb). Tests pass. Commit
  4. [Simplify the original buggy class](https://github.com/fixie/fixie/commit/4535866daa271a84f08df84f3866f81e9c53153d). Tests pass. Commit
  5. [Simplify](https://github.com/fixie/fixie/commit/70383b1361346276e5b639d5b0b32f0a94eec0c9). Tests pass. Commit

Each commit dramatically simplified yesterday's mess. Ugly helper classes in support of the mess melted away. The confusing, paranoid &#8220;Did we fail yet?&#8221; checks are gone. I dramatically &#8220;rewrote&#8221; the innermost secrets of the system, but **at no time did I take a large step**. The bug is fixed, and I have coverage I can actually rely on going forward. Now that I'm back to writing simple idiomatic C#, there's room for even more simplification. Phew!