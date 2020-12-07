---
title: Listening to Leaves
layout: post
---
In [Fixie XML Reports](https://patrick.lioi.net/2014/01/31/fixie-xml-reports/), I described a new [Fixie](https://github.com/plioi/fixie) feature that enables integration with build servers. When you need to output test results in the XML formats made popular by NUnit and xUnit, you can enable that extra output at the command line. This feature initially proved more difficult than expected, and all the trouble originated from trying to work within the confines of the wrong abstraction. As soon as we switched to the right abstraction, the implementation became simple.

## The Wrong Abstraction

Test frameworks like NUnit, xUnit, and Fixie can each be split into two main functional areas: the framework part, which knows what tests are and how to execute them, and then various _runners_. Runners include things like the console exe you call from a build script, but also runners for third-party systems like TestDriven.NET, ReSharper, and TeamCity.

From early on, Fixie has used the following abstraction for runners to implement (NUnit and xUnit have similar abstractions):

{% gist 8986300 %}

Fixie calls each method as the corresponding action takes place. The ConsoleListener reacts by echoing things to standard out, the TestDrivenListener reacts by echoing things to TestDriven.NET&#8217;s own listener abstraction, etc.

When it became clear that being able to output an NUnit-style XML file would help with build server integration, and in turn adoption of Fixie, I thought, &#8220;Gee, that&#8217;s just another Listener. It&#8217;ll be a breeze.&#8221;

I even got [a pull request](https://github.com/plioi/fixie/commit/08c430fa38bbf811963932553b1f598dd29ec8ef) that did exactly what I&#8217;d been picturing. It included a Listener which reacted to each event (AssemblyStarted, CasePassed&#8230;) by producing XML nodes of the NUnit style. The original developer realized there were two main problems with using the Listener abstraction:

  1. **Listener has the wrong lifetime for reporting on the run.** Each test assembly must be executed within its own AppDomain, and the Listener itself lives within that AppDomain. A single Listener can only ever know about the results of a single test assembly, but the XML report needs to include results from all assemblies in the run. The initial implementation of the feature could only work if your solution had one test project.
  2. **Your goal is to build a tree, but all you get is leaves.** A run of your tests, conceptually, produces a tree of results: tests within classes within assemblies. The Listener abstraction, though, gives you that information by only reporting the _leaves_ of that tree: a passed test case, a failed test case, or a skipped test case. The NUnit XML Listener had to infer the class associated with each case and build up an internal dictionary mapping classes to cases, so that it could finally output the tree structure as an XML document (otherwise, you might accidentally report the same class several times over, once for each method in that class).

The initial implementation gave nearly the results I wanted, but the implementation details seemed needlessly complex.

## The Right Abstraction

With help from [Sharon Cichelli](https://lostechies.com/sharoncichelli/) at a meeting of the [Polyglot Programmers of Austin](http://austin.polyglotprogrammers.org/), we realized the problems stemmed from using the wrong abstraction. We needed to acknowledge that we wanted to do tree-processing instead of leaf-event-listening. The fix was to have each assembly run return a tree of results to the console runner, so the console runner could build the One Complete Tree representing the entire run of N assemblies, and _then_ do a trivial traversal of that tree to spit out the corresponding XML. It turns out this is exactly what NUnit does to solve the same problem.

The resulting tree processing class (no longer a Listener), is [NUnitXmlReport](https://github.com/plioi/fixie/blob/d7c712a5286772dc3829a74080fbb1e969b45546/src/Fixie/Reports/NUnitXmlReport.cs). It is clean and straightforward: the shape of the code mimics the shape of the resulting XML document. It works no matter how many test projects your solution has. Supporting the [similar xUnit XML format](https://github.com/plioi/fixie/blob/d7c712a5286772dc3829a74080fbb1e969b45546/src/Fixie/Reports/XUnitXmlReport.cs) was likewise simple. No need to _infer_ the tree structure given only leaves; instead we simply turn one tree into another.

When you find yourself having to jump through hoops to make something _simple_ work, take a step back and evaluate whether you&#8217;re using an abstraction that helps you or hinders you.