---
title: Cleaning Up Test Failure Noise
layout: post
---
The [Fixie test framework](https://github.com/fixie/fixie) has no built-in assertion library, since assertions are an orthogonal concern to the nuts and bolts of test discovery and test execution. Although I stand by the decision to avoid including a built-in assertion library, there is one neat thing NUnit does with its _own_ assertions in order to simplify output of test failures. Today, we'll see Fixie's answer to that feature.

NUnit simplifies its own output when exceptions are thrown by its own assertion library infrastructure. For instance, when NUnit's `Assert.AreEqual(int, int)` fails by throwing an exception, the output deliberately excludes stack trace lines pointing _within_ the implementation of `AreEqual`, and deliberately excludes the name of the exception type. This filtering allows typical test failure output to remain as simple and direct as possible, pointing the developer to the line where their own test failed.

Since Fixie has no assertion library of its own, you may instruct it which types make up your assertion library's implementation details. Consider some hypothetical assertion library:

{% gist 9338868 %}

Out of the box, Fixie doesn't distinguish AssertionException from any other Exception (How could it?), so all of the exception details appear in the output. Consider a test class with some tests that will surely fail, and the corresponding default verbose output:

{% gist 9338886 %}

{% gist 9338895 %}

The implementation details of assertion libraries are rarely interesting to the developer. A custom convention can be instructed to simplify failure output by listing the types that make up the assertion library:

{% gist 9338909 %}

Rerunning the failing tests, Fixie simplifies the output, directing the developer to the actual failing line of test code:

{% gist 9338916 %}

In addition to identifying the types which make up your assertion library of choice, your custom convention may also list assertion extension classes defined in your own projects, further simplifying your output during failures.