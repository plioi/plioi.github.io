---
title: "Strongly Typed Whiteboarding"
layout: post
---


Last week, I announced the start of a <a href="http://patrick.lioi.net/2013/03/08/insufficiently-round-wheels/">new open source project, a test framework named Fixie</a>.  This week, we'll see how to mitigate technical risks during the early days of such a project.  In short, you do the hard work first.

When starting a new project, it's easy to spend all of your time thinking about what success looks like: the awesome features you have in mind, all working together in perfect harmony, with code so clean there's just no room for bugs.  However, there are all kinds of reasons a project could fail, and these reasons include the possibility of a technical obstacle turning the project into a nonstarter.  Early on, you've got to put on your Pessimist Hat and think,

> "What is the biggest technical challenge that, if unsolved, would make the rest of the effort worthless?"

## Identify the Deal Breaker

The 10,000 foot view of my plans for Fixie include the following features:

1. Low-ceremony test fixture class discovery by default.
2. Low-ceremony test case discovery by default.
3. Simple test fixture lifecycle by default.
4. A console runner you could invoke from a build script.
5. Integration with TestDriven.NET.
6. Slick customization of the defaults: fixture discovery, case discovery, and fixture lifecycle.


The first two are easy: I basically accomplished that when I <a href="http://patrick.lioi.net/2012/09/13/low-ceremony-xunit/">customized xUnit</a> a while back.  I'm similarly not concerned about making a simple fixture lifecycle either, since I've used enough reflection to be comfortable instantiating a given Type, invoking methods on that Type, and the like.  The console runner will be a relatively small wrapper around that foundation.  Slick customization is a big, fancy feature I'd love to get to, *but utter failure there isn't a deal breaker*.  I'd still want to use this on my own projects even if I could only leverage the simple defaults.

The deal breaker here is actually integration with TestDriven.NET.  Sometimes I use the ReSharper runner because it does a better job of running tests under the debugger, but GUI runners are usually overkill.  I use TD.NET because it's simple, fast, and just fits my daily routine better.  I wouldn't want to use Fixie on my other projects if it meant having to give up on TD.NET.  Fortunately, I found that it can work with third-party test frameworks.

## Create a Proof of Concept

I decided that I needed to see a successful proof of concept for integration with TD.NET, before working on anything else.  It didn't need to actually run any tests.  It just needed to prove that I could reach my own code in a meaningful way via TD.NET's keyboard shortcuts.  Success means hitting TD.NET's shortcut in Visual Studio, reaching my own code as a result, and then witnessing the expected output written by TD.NET.

I made a throw-away Solution for this proof of concept.  I didn't know what I was doing, I was sure I'd make a bit of a mess along the way, and I didn't want to feel any irrational pressure to keep that kind of code around as a starting point for the real thing.  I wrote one to throw away, and the key here is *knowing* this was the one to throw away.

> It was better than a whiteboard sketch, in that I got real feedback from the IDE at every step, but it was weaker than regular development.  All the planning and whiteboard diagrams in the world would have been irrelevant as soon as fingers met keyboard, so I dove into the implementation with the intent to learn from the activity and to discard the code afterwards.

The throw-away Solution contained two projects.  The first project represented the fake implementation of my test framework's TD.NET plugin.  The second project represented the test cases within a hypothetical consumer project.  The goal here was not to make a working plugin, but instead to prove to myself that the TD.NET integration would be technically feasable.  I wanted to discover how TD.NET behaved as I implemented that integration from scratch.

First, I left the fake plugin project *empty* and populated the sample consumer with a single fake test fixture:

```cs
using System;

namespace SampleConsumer
{
    public class MyTestFixture
    {
        public void MyTestCase()
        {
            throw new Exception("The test failed!");
        }
    }
}
```

Before doing anything in the plugin project, I wanted to confirm the behavior of TD.NET in the case that NO test framework is in play at all.  Right-clicking on the MyTestCase method and running it showed the following output:

```
------ Test started: Assembly: SampleConsumer.dll ------

Test 'M:SampleConsumer.MyTestFixture.MyTestCase' failed: The test failed!
    System.Exception: The test failed!
    MyTestFixture.cs(9,0): at SampleConsumer.MyTestFixture.MyTestCase()

0 passed, 1 failed, 0 skipped, took 0.17 seconds (Ad hoc).
```

Oh, right.  TD.NET has a convenient feature that it can run methods even if they are not officially recognized as test methods in a test fixture.  That can be useful to quickly try out a simple method during development, before you really have tests in place for it.  This behavior poses an interesting problem, though: **how will I *know* when my proof of concept is actually being detected and used by TD.NET as a real test framework plugin?**

There are **two** aspects to ensuring that my proof of concept is actually playing nicely with TD.NET:

* When I have no test framework in play yet and I attempt to right-click the MyTestFixture class to run ALL of it, TD.NET thankfully complains: "The target type doesn't contain tests from a known test framework or a 'Main' method."  Once I'm successful, I shouldn't get this message anymore.
* My initial implementation of the plugin will simply throw an exception of its *own*, surfacing in TD.NET's output, in contrast to the above output.


TD.NET includes a library, TestDriven.Framework.dll, which you can use to wire things up to your own test runner.  It contains an interface that you must implement for your plugin to be detected.  TD.NET plugs into Visual Studio to pick up on keyboard shortcuts and the like, then turns that into calls against this interface, allowing you to pick things up from there to do the actual work of running tests.

In the plugin project, I added a reference to TestDriven.Framework.dll and added the following class as a first pass:

```cs
using System.Reflection;
using TestDriven.Framework;

namespace TestDrivenPoc
{
    public class TdNetRunner : ITestRunner
    {
        public TestRunState RunAssembly(ITestListener testListener, Assembly assembly)
        {
            throw new System.NotImplementedException("RunAssembly");
        }

        public TestRunState RunNamespace(ITestListener testListener, Assembly assembly, string ns)
        {
            throw new System.NotImplementedException("RunNamespace");
        }

        public TestRunState RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
        {
            throw new System.NotImplementedException("RunMember");
        }
    }
}
```

Since each method just throws an exception with a specific message, I'll be able to prove to myself whether or not TD.NET actually found and executed the plugin.  I made the sample consumer project reference the plugin project, and attempted to run the sample test method and test fixture: no dice.  The results were the same as having no test framework in play at all!

There was another piece to the puzzle.

The TD.NET blog explains how to complete the detection of custom test runners: <a href="http://weblogs.asp.net/nunitaddin/archive/2009/11/05/testdriven-net-2-24-xcopy-deployable-test-runners.aspx">XCopy Deployable Test Runners</a>.  Over in the plugin project, I added a text file named TestDrivenPoc.dll.tdnet:

```
<TestRunner>
   <FriendlyName>Proof of Concept {0}.{1}.{2}</FriendlyName>
   <AssemblyPath>TestDrivenPoc.dll</AssemblyPath>
   <TypeName>TestDrivenPoc.TdNetRunner</TypeName>
</TestRunner>
```

In Solution Explorer, I set this text file to "Copy Always".  Now, when I rebuild the solution, the consumer gets a copy of this file right beside the similarly named DLL.  (When it comes time to deploy this to users, I'll need to be sure to include this file along with the DLL.)  When running MyTestCase or MyTestFixture, we now see the exception thrown by our fake test framework's TD.NET integration:

```
------ Test started: Assembly: SampleConsumer.dll ------

Test 'M:SampleConsumer.MyTestFixture.MyTestCase' failed: RunMember
    System.NotImplementedException: RunMember
    TdNetRunner.cs(20,0): at TestDrivenPoc.TdNetRunner.RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
    at TestDriven.AdHoc.TestRunner.CustomTestRunner.RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
    at TestDriven.TestRunner.AdaptorTestRunner.Run(ITestListener testListener, ITraceListener traceListener, String assemblyPath, String testPath)
    at TestDriven.TestRunner.ThreadTestRunner.Runner.Run()

0 passed, 1 failed, 0 skipped, took 0.22 seconds (Custom).
```

```
------ Test started: Assembly: SampleConsumer.dll ------

Test 'T:SampleConsumer.MyTestFixture' failed: RunMember
    System.NotImplementedException: RunMember
    TdNetRunner.cs(20,0): at TestDrivenPoc.TdNetRunner.RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
    at TestDriven.AdHoc.TestRunner.CustomTestRunner.RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
    at TestDriven.TestRunner.AdaptorTestRunner.Run(ITestListener testListener, ITraceListener traceListener, String assemblyPath, String testPath)
    at TestDriven.TestRunner.ThreadTestRunner.Runner.Run()

0 passed, 1 failed, 0 skipped, took 0.18 seconds (Custom).
```

Fantastic!  These exceptions prove that TD.NET was able to find our plugin and call methods on it, instead of the default behavior we saw when there was no test framework in place.

To sum up, when creating a test framework that needs to integrate with TD.NET:

* Reference TestDriven.Framework.dll.
* Implement TestDriven.Framework.ITestRunner.
* Include a *.tdnet text file whose name matches the test framework's DLL, and whose contents identify the plugin's DLL and ITestRunner class. (In my case, these two DLLs were one and the same.)
* Consumers must reference the test framework and must have the *.tdnet file present.


## Eliminate Any Remaining Risks

Part of the interface strikes me as odd.  Note how in the sample successful output above, we hit "RunMember" after telling TD.NET to run a single test method **and** after telling it to run a whole test fixture class.  I would use the word "member" to describe a method-belonging-to-a-class, but using "member" to describe a whole class surprised me.

I was unsure of how to really implement RunMember(...), so I wasn't finished with the proof of concept.  With a little trial and error, I fleshed out just enough of this method to see how you can tell the difference between single-test-method execution versus whole-test-fixture execution:

```cs
public TestRunState RunMember(ITestListener testListener, Assembly assembly, MemberInfo member)
{
    if (member.MemberType == MemberTypes.Method)
        testListener.WriteLine("We are running a single test method.", Category.Info);
    else
        testListener.WriteLine("We are running a whole test fixture.", Category.Info);

    var type = member as Type ?? member.ReflectedType;

    var testResult = new TestResult
    {
        FixtureType = type,
        Name = type.FullName,
        State = TestState.Passed,
        TotalTests = 1
    };

    testListener.TestFinished(testResult);

    return TestRunState.Success;
}
```

This implementation determines whether we are running a test method or a test fixure, announces that fact in the output, and then reports success since we're still just faking test execution.  In writing this method, I learned that MemberInfo, part of the .NET reflection API, can be used to describe class members as well as classes.

> The awkward wording in the TD.NET interface is in fact due to awkward naming in the .NET Framework.  I'll want to be sure to avoid letting that complexity leak out into my own API.  Bad naming can spread like a disease.  If it posed an obstacle to my understanding ITestRunner, it would pose a similar obstacle to anyone reading Fixie's implementation.

Now, when I attempt to run my sample tests at the method or class level, I get the expected output:

```
------ Test started: Assembly: SampleConsumer.dll ------

We are running a single test method.

1 passed, 0 failed, 0 skipped, took 0.25 seconds (Proof of Concept 1.0.0).


------ Test started: Assembly: SampleConsumer.dll ------

We are running a whole test fixture.

1 passed, 0 failed, 0 skipped, took 0.24 seconds (Proof of Concept 1.0.0).
```

## Set the Proof of Concept Aside

This is a proof of concept.  I don't actually want this code in my real Solution.  Now that I have these notes handy, it can *and should* be discarded.  I'll grow the real implementation with more care when I'm ready to implement it.  With a little strongly-typed whiteboarding, I've removed the largest risk to the project and can feel free to proceed.

Next week, we'll cover the nuts and bolts of setting up a new project from scratch, covering Fixie's initial set of commits.
