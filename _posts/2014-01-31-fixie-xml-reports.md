---
title: Fixie XML Reports
layout: post
---
With help from [Pete Johanson](https://github.com/petejohanson) and [Jonas Samuelsson](https://github.com/JonasSamuelsson) we've implemented NUnit-style and xUnit-style XML reporting for the Fixie test framework. Today we'll cover what the feature is and how to use it. Next time, we'll see why it was initially challenging to implement the feature cleanly.

## Why Bother?

Before this feature, Fixie could already report to the console, TestDriven.NET, and TeamCity, so why bother replicating the XML reports of other frameworks? Who would willingly elect to work with XML anymore? If you're already running a .NET-friendly build server other than TeamCity, that build server almost certainly knows how to read and display results in these XML formats, but has no idea what Fixie is. By allowing Fixie users to opt into these formats, their build server can treat Fixie as a first-class citizen.

## Usage

A typical command for running Fixie as part of your build script is to specify a test assembly and optional custom name/value pairs of the form "`--name value --othername othervalue...`":

{% gist 8743518 %}

This command will write results to the console in a format that is meant for human consumption. It also makes the given key/value pair available to your conventions. Fixie doesn't know or care what a category is, but it will gladly hand that information to your convention.

To produce an NUnit- or xUnit-style XML report, you need to include one of the Fixie-specific reporting arguments:

{% gist 8743536 %}

> It looks like the xUnit file format can only describe a single test assembly, while the NUnit file format supports any number of assemblies.

I'm using the "fixie:&#8221; qualifier for built-in arguments to avoid ever conflicting with an end user's own custom arguments.

Each of these commands will produce a file named TestResult.xml of the desired format, which your build server can be instructed to read and interpret. Here are some samples produced when running Fixie's own self-tests: [NUnit format](https://gist.github.com/plioi/8743559), [xUnit](https://gist.github.com/plioi/8743573) format.