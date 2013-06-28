---
title: Enabling Change
layout: post
---

Unit testing is meant to enable change by giving you confidence about the current state of your project. However, one of the criticisms of unit testing is that fine-grained tests (such as having one or more tests per method), locks you into implementation details.  With fine-grained tests in place, you're not free to move responsibilities between methods and between classes.

How are we to resolve this apparent contradiction?

I do lean towards fine-grained tests, especially in the early days of a project. At that point, small implementation details are *all you've got*. As a project grows and evolves, that early "scaffolding" of fine-grained tests may start to become an obstacle rather a change-enabler. **Test frameworks are tools meant to give us the freedom to change, but we must deliberately wield them to enable that change.** When your fine-grained tests start to discourage change, introduce new tests at a higher level, focusing on the behavior of your system rather than focusing on individual method details. Once the higher-level tests provide meaningful coverage on their own, the early scaffolding tests can be removed.

<blockquote>Be willing to use your test framework to enable change, even when that change is within your test code. As your project evolves, so does your testing strategy.</blockquote>

## Fixie's Early Test Strategy

While <a href="http://www.headspring.com/patrick/bootstrapping/">bootstrapping</a> the basic functionality of the <a href="https://github.com/plioi/fixie">Fixie test framework</a>, I deliberately tested everything at a fine-grained level. One of the first things I implemented was the logic around executing a single test case. For a given test method, I needed to prove that I could invoke the method via reflection and properly handle some subtle exception catching details. The tests for this were fine-grained: I had several tests for a single pivotal method. I needed confidence over this important block of code because everything that followed would build upon it.

Fast-forward 2 months, and I have built up a lot more infrastructure.  Fixie's starting to resemble something useful, and I'm beginning to take serious steps towards the customization features that motivated the whole project. These features will have a big impact on what exactly happens when a test case runs. I've done some design work on how test case execution needs to work going forward, but *that early test-method-runner and exception-handler code was no longer in a good place*. I needed to start shuffling implementation details between a few classes, in order for the details to find their proper home and enable further work, but the important tests of that behavior were too fine-grained.

I needed to move code, but that code was set in concrete!

## Fixie's Revised Test Strategy

Rather than declare that unit testing is bad, I instead needed to admit that my tests needed to change just as much as the code *under* test needed to change. I needed to revise my testing approach in light of new information, to enable further development.

Fortunately, I was already close to the solution. As I started implementing more involved features like support for async/await test cases and IDisposable test fixtures, I developed a pattern of wrapping fake test fixtures within a real test fixture. The outer real fixture's tests must pass for my build to pass, but the inner fake fixtures are allowed to have failing tests. The outer real tests ask the test runner to run the inner fake test fixtures, capturing their results. The benefit to this approach is that I can confirm how Fixie will handle real test failures in the wild.

Consider the tests for Fixie's treatment of IDisposable test fixture classes (details omitted to emphasize the pattern):

[gist id="5581509"]

This pattern appeared a few times:
<ol>
<li><a href="https://github.com/plioi/fixie/blob/754af5e9c14bcb9ad55ce70d7f69ebdb84c26c35/src/Fixie.Tests/ClassFixtures/DisposalTests.cs">DisposalTests.cs</a> as described above.</li>
<li><a href="https://github.com/plioi/fixie/blob/754af5e9c14bcb9ad55ce70d7f69ebdb84c26c35/src/Fixie.Tests/ClassFixtures/ConstructionTests.cs">ConstructionTests.cs</a> demonstrates the behavior of test classes that have constructors.</li>
<li><a href="https://github.com/plioi/fixie/blob/754af5e9c14bcb9ad55ce70d7f69ebdb84c26c35/src/Fixie.Tests/ClassFixtures/AsyncCaseTests.cs">AsyncCaseTests.cs</a> demonstrates the behavior of test classes when the individual test case methods use async/await.</li>
</ol>

Even though the specific code paths under tests are not *super close* to the code that tests them, all the relevant paths are being exercised. I'm getting meaningful code coverage but at a not-so-fine-grained level.

I translated the original fine-grained tests to this new approach, giving me <a href="https://github.com/plioi/fixie/blob/754af5e9c14bcb9ad55ce70d7f69ebdb84c26c35/src/Fixie.Tests/ClassFixtures/CaseTests.cs">CaseTests.cs</a>.  Now, *all* test execution is exercised at the same high level. Rather than asserting on the behavior of running a single test method, I assert on the behavior of running a whole test class. I needed to admit that there's more to running a test case than just calling the test method itself.

<blockquote>I don't think it's a coincidence that the level at which I'm testing resembles the level at which end users would reason about a test framework. Fixie's test suite is not quite executable documentation, but it certainly suggests what ought to appear in the documentation.</blockquote>

I dropped the original fine-grained tests now that they are redundant. With my obstacle removed, I am free to make some important changes to the organization of Fixie's test-executing code, the results of which we'll see here in the coming weeks.

## Be An Enabler

The new approach exercises all the same code as before, but because it is not directly calling into low-level implementation details, I am now free to shuffle those details around without breaking anything. I've found the level of test granularity that is appropriate for this system. When your tests start to discourage change, consider moving up a level to test the larger behaviors of your system, and then drop the fine-grained tests once they are no longer telling you anything useful.