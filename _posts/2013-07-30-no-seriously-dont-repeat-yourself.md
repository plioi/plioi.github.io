---
title: 'No, Seriously. Don''t Repeat Yourself.'
layout: post
---
A few weeks back, while writing about Fixie's [treatment of command line arguments](http://patrick.lioi.net/2013/07/12/by-your-command-line/), I discovered a bug which had been lurking for months. This bug was easy to trigger and produced a useless error message, making for a lame first impression, so I'd like to do a quick postmortem.

## The Symptom

You install [Fixie](https://www.nuget.org/packages/Fixie), write some tests, and run them successfully via TestDriven.NET. For your automated build, you try to invoke the Fixie.Console.exe test runner from your build script. Your script runs the exe from the location NuGet installed it, under the packages/ folder. You provide a single command line argument: a relative path to your compiled test project's DLL.

Instead of locating the DLL and running its tests, though, Fixie.Console.exe would tell you that there is no such DLL, even though it clearly does exist exactly where you said.

## The Diagnosis

A few breakpoints later, it seemed I had failed to convert relative paths into absolute paths, though I distinctly remembered doing just that and was able to find _two_ such calls to `Path.GetFullPath(string)`. Two? That's fishy. Why would I have to convert the relative path to the absolute path twice?

One of the calls took place early on, while preparing the environment in which the tests would run. The call to `GetFullPath` was used so that I could then get the full folder path containing the DLL. It set the current directory to that folder for the duration of the test run.

The second call happens shortly after that environment preparation, when it's time to actually load the DLL and search for the tests contained within it. Unfortunately, _by this point we had already changed the current directory._ Relative paths would be resolved with the wrong prefix path.

I didn't run into this before because my build script happened to pass absolute paths as arguments to Fixie.Console.exe. Both calls would work because the input was already absolute both times.

## The Fix

The fix was simple enough: [convert incoming paths to absolute paths _immediately_](https://github.com/fixie/fixie/commit/104c0c8dda724f6916c364b48b6fb0c17918859a), allowing the rest of the system to work exclusively with absolute paths.

`Path.GetFullPath(string)`, at a quick glance, seems to be a "pure&#8221; function: string goes in, the machine gets a little bit warmer, string comes out. However, this function can only work by reaching out to the environment for an additional input: the current directory. I made the "same call&#8221; twice, but only the first call produced a meaningful result.

Most explanations of the DRY principle focus on the risks of copying chunks of code, due to maintenance issues you can run into later on. Here, though, even a single redundant function call led to problems. Duplicating even simple function calls can lead to issues when those functions are incorrectly treated as pure.