---
title: Fixie Turns 1!
layout: post
---
Almost two years ago, I [announced the start of development](http://patrick.lioi.net/2013/03/08/insufficiently-round-wheels/) for a new .NET testing framework. This week, I've published Fixie 1.0. Looking back on that announcement, it's refreshing to see that the original vision held true throughout the whole process. I knew what I wanted from a testing framework, and now I have it. It's a happy coincidence (and a boost to my ego) that others have found it useful, too.

## What Does 1.0 Really Mean?

Fixie is ready for production use. Before now, early adopters have had to face two major issues: a "moving target&#8221; API, and missing features compared to similar frameworks.

While the developer-facing convention API was under development, each new release meant breaking changes. Types, properties and methods would change their names as I fleshed out the underlying model and learned from my missteps. Overloads would disappear and return again in later versions. The command line argument syntax has been through a few revisions. Early adopters tend to be open to a bit of a moving target early in a tool's development, but at some point things have to settle down and become reliable in order for the tool to really take off. 1.0 means that breaking changes will no longer be taken so lightly. Once you're on 1.0, subsequent upgrades should go smoothly.

During that early adopter phase, people naturally faced a lack of familiar and important features. In this blog series, I've documented the project from "File > New Project&#8221;, to a [comedically minimal minimum viable product](http://patrick.lioi.net/2013/03/26/bootstrapping/), all the way to the current mature state. One of the earliest steps was to produce a NuGet package, so people have been able to try it out since before it had most of its current features. The first version was only powerful and expressive enough to run its own suite of tests, no more. You wanted to [mark a test as skipped](https://fixie.github.io/docs/skipping-tests/)? You had to wait. You wanted [parameterized tests](https://fixie.github.io/docs/parameterized-test-methods/)? You had to wait. You wanted to [reuse your conventions](https://fixie.github.io/docs/reusing-conventions/) across the test projects of your solution? You had to wait.

Now, all of the must-have features are in place, and the patterns of the implementation will give guidance around extending the system without having to introduce breaking changes.

The biggest missing feature leading up to this release was that there was no visually-friendly test runner. From very early on, it had a console runner and a TestDriven.NET runner. This happened to fit my own workflow, so the fact that they were text-only runners didn't bother me. However, anyone who uses a visual runner like Visual Studio Test Explorer or ReSharper would feel an immediate regret as soon as they switched to Fixie. Since NUnit and xUnit are Fixie's most direct "competitors&#8221;, people should be able to make the switch from those tools without completely upending their own workflow. At last, [Fixie has support for the Visual Studio Test Explorer](https://fixie.github.io/docs/visual-studio-runner/). Work on this feature in December pushed me to greatly improve the API that all test runners can use, so the third-party ReSharper runner will be brought up to speed with 1.0 as well, in the not-too-distant future.

## GitHub Organization

The official repo has been moved from my personal GitHub account to [a new GitHub Organization](https://github.com/fixie). The organization contains the main repo (the core library, console runner, Visual Studio runner, and TestDriven.NET runner), the [documentation site](https://fixie.github.io/), and a sandbox repo useful when testing out the various runners.

## Thank You!

This was definitely not a one-man show.

First, I have to thank the people who made NUnit, xUnit, NSpec, and Machine.Specifications. Any .NET testing framework is going to run into a lot of the same challenges. Whenever I was stumped on a tough problem, I found myself studying the solutions in these other projects. One example is their treatment of AppDomains, which proved to be the most challenging concept throughout the entire project. I could see [how these other projects dealt with AppDomains](https://github.com/fixie/fixie/issues/8), and in comparing them I was able to learn what aspects of AppDomain interactions were essential versus optional. It was similarly enlightening to see how the other frameworks dealt with integrating with the Visual Studio Test Explorer, which has a number of infuriating quirks to work around.

Next, [people around the world](https://github.com/fixie/fixie/graphs/contributors) have submitted GitHub [issues](https://github.com/fixie/fixie/issues/61), [pull requests](https://github.com/fixie/fixie/pull/42), and [awesome analyses of bugs](https://github.com/fixie/fixie/pull/60#issuecomment-63210776).

> I love this one. It's the weirdest bug report I've ever seen, and it revealed a real problem: [TargetParameterCountException during test execution](https://github.com/fixie/fixie/issues/74).

My own coworkers used early versions of Fixie on their projects, once it had enough features to be useful. It's been great to see what happens when your project bumps up against the real world, and it's been great to see it used in ways I hadn't even anticipated. In addition to being my guinea pigs, everyone at work has been very supportive of the project, all the way up to [the president of the company](https://twitter.com/headspring24_7) who provided positive reinforcement of my self-imposed deadline and helped to guarantee a solid block of personal days during an otherwise busy time of the year.

I would not have gotten this far without all your help. Thanks!