---
title: "When Writing C#, Use C#"
layout: post
---


Recently, Jimmy Bogard described several <a href="http://lostechies.com/jimmybogard/2013/06/18/strategies-for-isolating-the-database-in-tests/">strategies for isolating a database in tests</a>.  Today, we'll see how one of these strategies can be implemented.  We'll start with a common implementation under NUnit, then we'll identify some issues with that implementation, and lastly we'll translate it into a Fixie convention to address those issues.

> Today's code samples work against <a href="http://nuget.org/packages/Fixie/0.0.1.63">Fixie 0.0.1.63</a>. The customization API is in its infancy, and is likely to change in the coming weeks.

## Transactions Under NUnit

One of the strategies from Jimmy's post involves starting up a transaction before a test and rolling back that transaction at the end of the test.  If we're using NUnit, a common technique is to stow this concept away in a test fixture base class like so:

```cs
public abstract class TransactionScopedTests
{
    TransactionScope scope;

    [SetUp]
    public void SetUp()
    {
        scope = new TransactionScope();
    }

    [TearDown]
    public void TearDown()
    {
        scope.Dispose();
    }
}
```

Our integration tests can inherit this base class, allowing each test to run in isolation.  Each test gets to work against the same state.

This approach has a few problems.

First, we've got a bit of a temporal coupling issue: this works because we trust that SetUp() and TearDown() will be called in a particular order. Ok, so it's not an example of temporal coupling gone horribly wrong - NUnit *will* call them in the right order. Still, it's a bit of a code smell in that it motivates us to explicitly call Dispose(), despite the fact that C# already has a keyword ("using") devoted to automating that call safely. NUnit's lifecycle attributes are like a mini language built on top of C#, and **we're writing this code in NUnit-the-language instead of writing it in C#**.

Second, relying on [SetUp] and [TearDown] in a base class can get a little ugly when the child test class also needs to have a [SetUp] or [TearDown], as we saw in <a href="http://patrick.lioi.net/2013/06/12/dry-test-inheritance/">DRY Test Inheritance</a>.  Should we mark these methods `virtual` so child classes don't have to come up with new names for their own [SetUp]s and [TearDown]s? If child classes override them, they could easily forget to call base.SetUp() and base.TearDown(). Even if they remember to, that's more boilerplate than feels necessary.

**Yikes. I just wanted to wrap my tests in transactions. Let's do that instead.**

## Transactions Under Fixie

As I've demonstrated in recent weeks, Fixie conventions allow you to describe test *discovery* as well as test*execution*. In this case, we'll stick with the simple style of test discovery that Fixie uses by default, but we'll also augment test execution with a transaction:

```cs
public class IntegrationTestConvention : Convention
{
    public IntegrationTestConvention()
    {
        Fixtures
            .NameEndsWith("Tests");

        Cases
            .Where(method => method.Void())
            .ZeroParameters();

        InstanceExecution
            .Wrap((fixture, innerBehavior) =>
            {
                using (new TransactionScope())
                    innerBehavior.Execute(fixture);
            });
    }
}
```

The Fixtures and Cases code resembles what Fixie offers in its DefaultConvention.  Like the DefaultConvention, we also get one instance of the test class for each test being executed.  The relevant bit is the last section:

```cs
InstanceExecution
    .Wrap((fixture, innerBehavior) =>
    {
        using (new TransactionScope())
            innerBehavior.Execute(fixture);
    });
```

Here, we are saying that the normal behavior for each test class instance ("run the test") should be wrapped in a new TransactionScope. In other words, "within a transaction, proceed with the test execution".

> This part of the API deserves more attention. Better names for each concept could make it more clear what's going on with this 'innerBehavior'. It works, but it's still too wordy.

I've been focusing lately on supporting this notion of *wrapping* the built-in behavior with a short code snippet, instead of NUnit's separate Before/After approach, because it offers more degrees of freedom: a wrapping action can do extra work before, after, around, or even *instead of* the built-in behavior. A nice little bonus is that we get to write our C# in C#: the code that cares about transactions is now as small as can be and resembles the way you would use a TransactionScope in the code-under-test.
