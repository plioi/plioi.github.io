---
title: The Leakiest Abstraction
layout: post
---
Centuries from now, when archaeologists are digging through the rubble patch located over the former home of the GitHub servers, they will recover a copy of [Fixie's repository](https://github.com/fixie/fixie). Inspecting its history, an unfortunate soul will discover a particularly horrible abstraction. Upon unearthing that abstraction, the [lights in Cairo](http://www.kingtutone.com/tutankhamun/curse/) will mysteriously go out. Soon after, kids will tell ghost stories around campfires about just how terrible Patrick's Horribly Leaky Abstraction (the PHLA) really was. Parents will begin to threaten unruly children with it: &#8220;Be good, or Santa will leave a lump of PHLA in your stocking!&#8221; After a while, college students and young professionals envious of college students will promote the inevitable PHLA meme on reddit.

Early in its life, I introduced a leaky abstraction into the Fixie test framework. It didn't all happen at once, but it got away from me before I realized what I was doing. Deviously, it _worked_ right up until it didn't, so it had plenty of opportunity to leak, spreading incidental complexity through the whole system. A week and 34 commits later, I've finally corrected the problem.

Today, I'll cover the initial mistake and the manner in which it spread. Next time, I'll cover the approach I took to responsibly and safely fix it. Hopefully, the clues I missed along the way will jump out at you when you're at risk of making similar mistakes, and if you find yourself in a similar situation you'll hopefully benefit from the approach I took to fix it.

It's just **so** bad. Before proceeding, please keep in mind that pobody is, in fact, nerfect.

## The Original Problem

Let's say you're writing a .NET test framework, similar to NUnit or xUnit. Your goal is to use the reflection API, first to discover test methods and then to invoke them. When invoked test methods throw exceptions, such as from a failed assertion, your job is to catch that exception and report it to the user as a test failure. Within your innermost loop, you therefore want to call [`MethodInfo.Invoke()`](http://msdn.microsoft.com/en-us/library/system.reflection.methodinfo.invoke.aspx) while catching exceptions.

Even this simple goal comes with a subtle complication. When the test method is _reached_ and _fails_, `MethodInfo.Invoke()` throws a `TargetInvocationException` that _wraps_ the original test failure's own exception. When the test method cannot even be reached, such as when it accepts arguments the test framework doesn't know how to provide, `MethodInfo.Invoke()` throws some other Exception.

In the case of tests that are reached yet fail, we want to report the _wrapped_ exception to the user. Reporting the `TargetInvocationException` would be weird and confusing to the user. You want to show them the assertion that originally failed, pointing at the line of their assertion statement.

In the case of tests that cannot even be reached, we want to report exactly the Exception that was thrown. The test framework has failed and the least we can do is explain the plain truth of that failure.

In other words, we want to &#8220;unwrap&#8221; any `TargetInvocationException` thrown by the test method invocation. We'd be tempted to do the following:

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

This attempt would certainly usher test failures back to rest of the test framework. Test methods that are reached yet fail will throw the actual assertion failure back to the rest of the framework, to be caught and reported to the user. Test framework bugs will bypass this catch block (since they are some _other_ Exception type) and likewise throw back to the rest of the framework, to be caught and reported to the user.

This attempt is flawed. &#8220;throw ex.InnerException&#8221; will propagate the original exception _message_ correctly, but the exception _stack trace_ will be destroyed. The end user will see that their assertion failed, but it will say that it failed within the above catch block instead of within their test. They get the right message, but they don't know where in their own code the failure happened!

## The Conflation

The actual fix for this is simple. I didn't see it at the time, though, because I was conflating this problem with another exception-handling problem. Usually, a test failure requires a single exception to be displayed to the user. Sometimes, multiple exceptions need to be reported: a test throws, and then the test's teardown operation throws, and then the test class's Dispose() method throws. We want to report the test failure as the primary exception, but we might as well describe the subsequent failures as well.

To address the issue of potentially-many exceptions per test, I added an abstraction called [ExceptionList](https://github.com/fixie/fixie/blob/d3cc2fd1e2092bbcdc464d172a8ca5344a175ec9/src/Fixie/ExceptionList.cs). Every test case has one. When it's empty, the test passed. When it's not, it failed for a number of reasons.

I had two problems: the need to unwrap TargetInvocationExceptions, and the need to collect multiple exceptions per test case.

Mistakenly combining these two small problems into one all-encompassing exception-handling problem led me to an overly complex solution. Where ExceptionList belonged in a very small part of my system, it quickly spread everywhere. As a &#8220;fix&#8221; for the insufficient try/catch above, I did something like this:

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

Instead of throwing exceptions, I would return nonempty ExceptionLists. I was correctly &#8220;unwrapping&#8221; TargetInvocationExceptions, and I was allowing the rest of the framework to combine these exceptions with things like failed teardown and Dispose() exceptions. I thought I was doing Good.

## The Leak

Collecting and returning exceptions didn't seem too offensive. The caller, though, _also_ had to adopt the pattern of collecting and returning exceptions. The caller's caller? Collect and return. On and on. When Fixie needed to construct a test class, run several tests, and finally Dispose of the test class, I was doing this:

```cs
object instance;

var constructionExceptions = Construct(testClass, out instance);
if (constructionExceptions.Any())
{
    foreach (var @case in cases)
        @case.Exceptions.Add(constructionExceptions);
}
else
{
    //...run all the test cases here, collecting exceptions along the way...

    var disposalExceptions = Dispose(instance);
    if (disposalExceptions.Any())
    {
        foreach (var @case in cases)
            @case.Exceptions.Add(disposalExceptions);
    }
}
```

Wait a sec. At each step I ask, &#8220;Did anything fail yet?&#8221;. If something failed, I note the failure and then stop doing the rest of the operation. These if/else/return sequences are suspiciously like try/catch/throw, except that they are complex, nonidiomatic C#, and just _weird_.

The ExceptionList nonsense was spreading into the public customization API, which is _supposed_ to be Fixie's main selling point. An example from a previous blog post included a ridiculous method of the form:

```cs
ExceptionList DoSomething()
{
    // Do the actual work here...


    // Return an empty list of exceptions
    // to indicate success.

    return new ExceptionList();
}
```

Insanity.

What if the &#8220;actual work&#8221; here, um, throws an exception? Better put some kind of try/catch at the end of the line to patch things over, right? Eventually I ran into a situation where I just couldn't keep propagating the weirdness, and I was _again_ left with the need to safely re-throw some InnerException. I didn't just reinvent exceptions, I made an _insufficient_ reimplementation of exceptions.

The thing that finally lifted my blinders was that I discoverd this buggy need-to-rethrow-again while _all of my tests were happily green_. Stop all of the presses. Hold all of the phones. Shut Down Everything.

## Oops

Requiring users to write methods like DoSomething() above is absolutely unacceptable. Thankfully, I've fixed the problem, and we'll cover that journey in my next post. For today, let's focus on how it feels to get into this kind of predicament, so that you can recognize similar mistakes sooner. (Your own mistakes will surely be _less_ insane.)

I started with a legitimate problem: I needed to preserve stack traces of exceptions thrown by test methods. My initial solution solved the immediate problem. The solution, however, was not localized. While writing other parts of the system, _I had to keep that solution in mind and propagate it to avoid making new mistakes_. I had to keep on remembering to do all exception handling in this unusual way.

> The innermost operation was an oddity. Everything that used the innermost oddity had to be odd. Everything that used something odd had to be odd as well. Therefore, by mathematical induction, everything had to be odd.

Fortunately, there's a way out of messes exactly like this one. The suspense is killing you!