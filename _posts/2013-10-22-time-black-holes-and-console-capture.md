---
title: Time, Black Holes, and Console Capture
layout: post
---
In [Listen Up!](https://patrick.lioi.net/2013/10/21/listen-up/), we covered the implementation of Fixie/TeamCity integration, focusing on the main abstraction that feature introduced. Integrating with TeamCity forced me to improve the project in other ways, affecting more than just TeamCity: tracking the execution time of each test, addressing a console redirection bug, and capturing console output for report generation.

## Test Timing

The first improvement was simple. TeamCity wants to display the time each test takes to run, and the messages we output to TeamCity therefore contain a duration value in milliseconds. This was the first time I needed to know the duration of a test, so I had to implement the timing functionality first. The main test execution loop gained a Stopwatch for each test case:

{% gist 7089303 %}

The collected duration, a TimeSpan, is one of the things passed along to the Listener interface upon each test completion. TeamCityListener, for instance, receives that TimeSpan and outputs the duration in the units expected by TeamCity.

## The Black Hole Bug

Next, TeamCity integration motivated the discovery and fix of a bug which could allow a "rude&#8221; test to trample Fixie's own output.

TeamCity, and potentially any other test reporting tool, wants to be handed the string equivalent of any output that a test wrote to the console. In general, tests should rely on assertions as their main form of "output&#8221;, but sometimes console output can help with diagnosing tricky problems. Maybe stepping through a test in the debugger locally works fine, but then the same test fails on the CI machine. Resorting to "low tech&#8221; debugging with `Console.WriteLine`s may be the quickest path to a diagnosis.

As I started to work my way through the console-capture feature, I realized that a related bug was in play. Consider a test method that redirects the standard output stream:

{% gist 7089325 %}

Fixie starts up a test run, outputting results to the console as it goes, either via ConsoleListener or TeamCityListener. It reaches this evil test, outputs "All is well.&#8221;, and then the developer sees _nothing else_. All subsequent attempts by Fixie to write anything to the console, such as other tests' failures, fall into the blackHole StringWriter instead of the real console.

Imagine you're using Fixie to test your project, and your project happens to redirect console output for legitimate reasons, but fails to gracefully return things to the original output TextWriter? Suddenly, Fixie stops telling you what's going on. Imagine having to diagnose that issue. Bad news.

Thankfully, implementing the console output capture feature fixes this bug at the same time.

## Console Capture

The RedirectedConsole class below allows you to temporarily redirect all console output to a StringWriter for the duration of a `using` block:

{% gist 7089334 %}

We wrap the execution of _each_ test in a RedirectedConsole for two reaons. First, we need to obtain the string equivalent of anything the test wrote to the console. Second, we need to ensure that we redirect console output back to the right place at the end of each test, so that our EvilTest can no longer interfere with other tests. Fixie captures the output of each test, but can still confidently write output of its own without fear of it falling into a black hole:

{% gist 7089345 %}