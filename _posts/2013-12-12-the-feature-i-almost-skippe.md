---
title: The Feature I Almost Skipped
layout: post
---
Until recently, the [Fixie test framework](https://github.com/fixie/fixie) had no notion of skipped tests for two reasons.

First, skipped tests are like support beams for technical debt. You know there's a problem, and you've decided to either ignore it or delay solving it, contrary to the whole point of automated test coverage. It's unclear to the next developer whether it should be brought back into the fold, or when. The problem being ignored lurks there, waiting to bite you sometime in the future. Yellow warnings in the console runner's output go unnoticed and become the norm. People start skipping over them mentally as much as the test runner does, and it teaches other developers on the project that it's not a big deal to skip other tests once they start failing.

Second, if you _really_ wanted to skip tests in Fixie, you could already effectively do that with a modified convention, like so:

{% gist 7930603 %}

> In other words, "a method is a test method if it isn't marked as skipped."

I don't like skipped tests, and I basically had poor-man's skipped tests anyway, so why bother to implement true support for them in Fixie? Well, the convention trick above is _even worse_ than normal, because the skipped tests would not even show up in result counts and would not come with warnings in the output. They'd be even easier to forget and let rot. Also, Fixie's convention-based approach to test discovery and execution opens the door to creatively mitigating the risks of skipping tests, as we'll see in our example below.

My conclusion was to let Fixie know what it means for a test to be skipped, so that it can count them and warn the user like other test frameworks, but to deliberately not include any way to mark tests as skipped in the DefaultConvention. Out of the box, tests can't get skipped. **If you want skips, you're going to have to ask for them.** We can change the earlier poor-man's skipping convention so that you can alert the user to their pending disaster:

{% gist 7930617 %}

Since the hook is a Func<Case, bool>, you can include custom logic aside from the mere presence of an attribute or naming convention. One way to mitigate the risk of skipping tests is to place an expiration date on them:

{% gist 7930624 %}

You might define a skip attribute that will skip a test until a specific GitHub issue gets closed, for instance (though that one might be a tad overkill, checking GitHub on every test run). Fixie only cares whether or not you want a given test to be skipped, and sets it aside for counting and reporting.

## The Implementation -or- The Greatest Pull Request Ever

Other test frameworks treat skippedness as a kind of _execution result_. "Run this test." "No. I'll pretend I did _and then tell you that I pretended I did_." This is weird, and I was about to mimic that weirdness when Max Malook (aka [mexx](https://github.com/mexx) on GitHub) found a much more clear way to implement it.

From a user's point of view, 3 things can happen to a given test: it can run and fail; it can run and pass; or it can be set aside, counted and warned as never having been run in the first place. mexx's implementation does exactly that: after determining that a method is a test, a separate decision is made about whether it should be skipped. If it should be skipped, it is counted and reported but we never bother running it and we never bother pretending to run it. Simple, and a perfect match for the end-user's expectations. We didn't have to muddy the waters of what it means for a test to be executed. [mexx's pull request was fantastic](https://github.com/fixie/fixie/pull/24/files): a clear implementation, with test coverage that matched the existing test style I was using for similar features.