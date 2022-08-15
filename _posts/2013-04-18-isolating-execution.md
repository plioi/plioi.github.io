---
title: "Isolating Execution"
layout: post
---


In last week's post, <a href="http://patrick.lioi.net/2013/04/11/dogfooding/">Dogfooding</a>, I uncovered a bug in the <a href="https://github.com/fixie/fixie">Fixie test framework</a> by trying to use it on two of my other side projects.  At the end of that post, I claimed that the bug had something to do with "AppDomains" and stated that it would be fixed once I met the following requirement:

> A test framework should fool your test DLL into thinking it is an EXE running in the build output folder, when in fact the running EXE is the console runner off in some other folder.

Today, we'll cover the bug's diagnosis and resolution.

## Initial Clues

I originally developed <a href="https://github.com/plioi/rook">Rook</a> with the xUnit test framework.  I installed Fixie beside xUnit to see if they produced the same results.  The results were surprising:

1. xUnit under TestDriven.NET ran all the tests, as it always has.
2. xUnit's console EXE ran all the tests, as it always has.
3. Fixie under TestDriven.NET ran all the tests.
4. Fixie's console EXE failed on all the *integration* tests.


This odd mix gave me some useful information.

First, xUnit and TestDriven.NET must be doing extra work prior to executing the tests, but Fixie's console EXE was neglecting that work.

Second, in the failure scenario, all the unit tests worked while all the integration tests failed.  The unit tests were relatively simple: chop up strings, walk through collections, assert on the collection contents.  The integration tests, on the other hand, needed to touch the file system too.

I concluded that Fixie's console EXE was most likely neglecting some kind of setup step related to the file system, but I needed more information.

## Diagnosing the Bug

Rook's integration tests take plain text files as input and generate new assemblies (DLLs) as output.  When the tests failed, the generated assemblies were trying to locate some types defined in the Rook.Core.dll library, which sits right beside the tests' own DLL.  **When the tests failed, they failed because they could not find Rook.Core.dll, even though it was sitting right there in plain sight.**

I added some debugging output to the tests, right before the point of failure, in order to see where .NET was trying to look for assemblies like Rook.Core.  I output the value of `AppDomain.CurrentDomain.BaseDirectory`, the first place .NET looks for DLLs.  The results revealed the issue:

1. Under TestDriven.NET, the BaseDirectory was src/Rook.Test/bin/Debug, which I expected.
2. Under the console EXE, the BaseDirectory was src/packages/Fixie.0.0.1.24/lib/net45, **which is where the console EXE lives.**


Aha! When you run a .NET EXE, the BaseDirectory is the same as the EXE's directory, so that the EXE can find all the DLLs that live right beside it.  This default is convenient 99.9% of the time, because the EXE is *king* 99.9% of the time.  A test runner EXE, however, should allow your test assembly to be king.  If a test tries to use the "current directory", it should use the test assembly's directory.  If a test tries to load a DLL from the "base directory", it should use the test assembly's directory.  When Fixie.Console.exe ran tests within Rook.Test.dll, and those tests generated assemblies that depended on Rook.Core.dll, *Fixie was looking for that in the wrong folder*.  I was asking .NET to perform magic:

> **Me:** Would you kindly locate a DLL for me?\
> **.NET:** Sure, I'll look where I always look.\
> **Me:** Oh, no, you should look in a folder that I consider to be special.\
> **.NET:** Where's that?\
> **Me:** It's a secret.\
> **.NET:** Get off my lawn.


## The Solution: Multiple AppDomains

We usually don't hear much about AppDomains because most of the time they are 1-1 with our EXE's process. Most of the things we think of as "the process" are really "the single AppDomain living inside the process".  The basic idea is that an AppDomain is a list of assemblies that have been loaded and that can call each other.  AppDomains also have some state such as the BaseDirectory, which answers the question, "What folder should I look in to find DLLs?"

Since the default BaseDirectory was wrong for my purposes, I needed to spin up a second AppDomain within the process, with BaseDirectory set correctly.  Then, I needed to make sure that Fixie did all of its work within *that* AppDomain instead of the default AppDomain.

Communicating between AppDomains is tricky because they are very much like separate processes.  They don't share access to objects in memory, so you have to throw serializable objects across the chasm.  In order to make this "long distance" communication *feel* like a regular method call, you can use a subclass of `MarshalByRefObject` to act as an intermediary.

> In AppDomain 1, we ask AppDomain 2 to create an instance of our intermediary class.  This request creates a *real* instance over in AppDomain 2.  Back in AppDomain 1, we get a *proxy*.  If you call a method on the proxy in AppDomain 1, the arguments get serialized and thrown over the chasm to the real object in AppDomain 2.  When the work is performed and the real object returns a result, that result is serialized and thrown back over the chasm to AppDomain 1.  It feels like a regular method call.

I created the `ExecutionEnvironment` to wrap all of the AppDomain interaction:

```cs
public class ExecutionEnvironment : IDisposable
{
    readonly AppDomain appDomain;
    readonly string previousWorkingDirectory;

    public ExecutionEnvironment(string assemblyPath)
    {
        var assemblyFullPath = Path.GetFullPath(assemblyPath);

        appDomain = CreateAppDomain(assemblyFullPath);

        previousWorkingDirectory = Directory.GetCurrentDirectory();
        var assemblyDirectory = Path.GetDirectoryName(assemblyFullPath);
        Directory.SetCurrentDirectory(assemblyDirectory);
    }

    public T Create<T>(params object[] args)
    {
        return (T)appDomain.CreateInstanceAndUnwrap(
            typeof(T).Assembly.FullName, typeof(T).FullName,
            false, 0, null, args, null, null);
    }

    public void Dispose()
    {
        AppDomain.Unload(appDomain);
        Directory.SetCurrentDirectory(previousWorkingDirectory);
    }

    static AppDomain CreateAppDomain(string assemblyFullPath)
    {
        var setup = new AppDomainSetup
        {
            ApplicationBase = Path.GetDirectoryName(assemblyFullPath),
            ApplicationName = Guid.NewGuid().ToString(),
            ConfigurationFile = GetOptionalConfigFullPath(assemblyFullPath)
        };

        return AppDomain.CreateDomain(setup.ApplicationName, null, setup,
            new PermissionSet(PermissionState.Unrestricted));
    }

    static string GetOptionalConfigFullPath(string assemblyFullPath)
    {
        var configFullPath = assemblyFullPath + ".config";

        return File.Exists(configFullPath) ? configFullPath : null;
    }
}
```

Upon construction, this class creates the second AppDomain, treating the given folder path as the BaseDirectory.  You call `Create<T>(...)` in order to create an object in the new AppDomain.  You get back a proxy which knows how to cross the chasm between AppDomains.  `Dispose()` frees up the resources used by the secondary AppDomain and returns the current directory back to its original value.  Fixie's `Main` method uses this class like so:

```cs
using (var environment = new ExecutionEnvironment(assemblyPath))
{
    var runner = environment.Create<ConsoleRunner>();
    
    //We are in AppDomain 1.
    //Send assemblyPath to AppDomain 2.
    //ConsoleRunner executes in AppDomain 2.
    //ConsoleRunner's return value is sent
    //back to us here in AppDomain 1.
    
    return runner.RunAssembly(assemblyPath);
}
```

`ConsoleRunner` is our `MarshalByRefObject`, which effectively lives on both sides of the AppDomain chasm:

```cs
public class ConsoleRunner : MarshalByRefObject
{
    public Result RunAssembly(string assemblyPath)
    {
        var assemblyFullPath = Path.GetFullPath(assemblyPath);
        var assembly = Assembly.Load(AssemblyName.GetAssemblyName(assemblyFullPath));

        var runner = new Runner(new ConsoleListener());
        return runner.RunAssembly(assembly);
    }
}
```

I wanted to minimize the amount of code that cared about AppDomains, so the `MarshalByRefObject` subclass is very small.  It receives the `assemblyPath` (the serializable object that got thrown across the chasm), and defers to `Runner` as quickly as possible.  `Runner` does the real work, and is used by both the console EXE and TestDriven.NET.  `Runner` has no idea that AppDomains are involved at all.  Only `ConsoleRunner` cares about that detail.

## Isolating Test Execution

I came to this solution by studying the similar steps taken by NUnit, xUnit, and Machine.Specifications.  All these test frameworks need to let the developer pretend that their unit test assembly is their main EXE, and they all do it by isolating test execution in a specially-configured AppDomain.  AppDomains are like processes-within-the-process, and `MarshalByRefObject` classes help to make inter-AppDomain communication feel like regular method calls.

It takes a lot of work to set up AppDomains, communicate with them, and clean up afterwards.  If you need to run code in isolation, `ExecutionEnvironment` is a useful starting point.
