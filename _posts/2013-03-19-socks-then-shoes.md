---
title: "Socks, *then* Shoes"
layout: post
---


A couple weeks back, I announced the <a href="http://patrick.lioi.net/2013/03/08/insufficiently-round-wheels/">start of development on Fixie</a>, a .NET test framework. Last week, I covered some early <a href="http://patrick.lioi.net/2013/03/13/strongly-typed-whiteboarding/">proof-of-concept work</a> performed with throw-away code. This week, we'll take a close look at the first <a href="http://patrick.lioi.net/2012/02/10/am-i-not-dead-commit/">several small commits</a> to <a href="https://github.com/fixie/fixie">Fixie on GitHub</a>.

The commits we cover today are all about preparing the initial solution structure, installing dependencies with NuGet, writing a build script, and applying a simple version numbers scheme.  I strongly recommend performing similar steps when starting up any new project.  You've gotta put on your socks before you can seriously think about putting on your shoes.  **Today's work is all about the socks.**

The goal here is to prepare a build script that compiles, runs tests, and applies version numbers to the built assemblies. Any developer should be able to clone the repo and run "build" at the command line in order to compile the project and run its tests.  It should be that easy; if it isn't, we've failed at putting on *socks*.

Although I'm framing this advice as "necessary first steps for any project", it's never too late to get a reliable build in place around your existing project.  Take it in small steps: get a build script that can simply compile the solution, then grow it to apply version numbers to assemblies, then grow it to run unit tests, put it on a Continuous Integration server, expand it to run integration tests...

## Initialize .gitignore

Here's a minimal .gitignore file you can start with.  It tells git to ignore some files that have no business being checked into source control, such as compiled assemblies under the bin/ folders.  Grow it as needed:

```
*.suo
bin
obj
src/packages
*.DotSettings.user
TestResult.xml
```

## Choose an Open Source License

I am not a lawyer, so I cannot offer advice as to which open source license is right for you.  I selected the MIT License and included it as a .txt file at the root of my repo.  The fact that I went with this is not a recommendation.  This is your decision, but whatever you choose, include it in your repo and be sure to include the official name of the license at the top of the file so that everyone knows what it is.

<a href="https://github.com/fixie/fixie/commit/af7c43afaf3111c2139e5886abc9b57e983a0962">Commit</a>.

## Initial Solution Structure

As we've already seen, our repo needs to contain things other than source code.  To keep our peas and rice separated on the plate, we'll put our actual code files under a src/ folder.  In Visual Studio, I created a new solution containing a Class Library project named Fixie, all under src/.

It's tempting at this point to start adding several projects to the solution.  Surely, I'll eventually need this test framework to produce a console-runner EXE, and a DLL to integrate with TestDriven.NET.  Also, I used to set up projects in a solution to enforce internal dependencies: Fixie's reflection-helpers code shouldn't depend on anything else, so it's tempting to put that in its own project with no references to other Fixie.sln projects.  Although tempting, **don't start creating projects left and right**.  That would be a symptom of Big Design Up Front, which usually doesn't turn out to be very accurate.

> Instead, we'll create projects at the last responsible moment: when doing so resolves a real *deployment* pain.  The general rule of thumb I follow when separating things into projects within the solution is to **create projects based on how the system will be deployed**. I'll only create a new project when I find myself creating something that must be separately deployable.

The first separately deployable thing we absolutely need right away is Fixie's own test project.  End users will need Fixie.dll to write their tests, but they won't need Fixie.Tests.dll.  Fixie.Tests.dll will be used by a developer working on Fixie, but end users don't need it to be deployed to their machine.  I added a Fixie.Tests project to the solution, also of type Class Library, which references the Fixie framework project.

At this point, we have the following file structure

```
    .gitignore
    LICENSE.txt
    README.md
    /src/Fixie.sln
    /src/Fixie/Fixie.csproj
    /src/Fixie/Properties/AssemblyInfo.cs
    /src/Fixie.Tests/Fixie.Tests.csproj
    /src/Fixie.Tests/Properties/AssemblyInfo.cs
```


<a href="https://github.com/fixie/fixie/commit/c066c8b89ff606a59d26a60ce77d7c890ed53ef6">Commit</a>.

## Enable NuGet Package Restore

It is extremely important that your repo be aware of all of its dependencies, so that a new developer can simply clone from GitHub and *go*.  If at all possible, they shouldn't have to install anything extra.

> If your new coworker has trouble getting up and running on their machine, chances are you'll have similar difficulties when you deploy to your production machine.  Consider any repo clone to be a mini deployment.  Detect deployment pains early, when they're easiest to address.

Before installing any NuGet packages, though, we want to "Enable NuGet Package Restore".  Doing so adds a few files to our solution.  When we later install packages, source control will only contain the master list of the names/versions of the packages we depend on.  The package DLLs themselves are deliberately *excluded* from source control via .gitignore.  When a new developer clones the repo and builds, NuGet will step in, download any missing dependencies (like NUnit), and *then* compile.  The combination of NuGet Package Restore and .gitignore gives us a clean repo on GitHub and a smooth experience for new developers.

In Solution Explorer, right-click the solution and select Enable NuGet Package Restore.  Git will show you that src/Fixie.sln was updated to include the contents of a new folder /src/.nuget containing a few files.  We can leave them alone, as the default settings are just fine.

Be sure to include the following line in your .gitignore.  This is the folder that will contain downloaded packages.  Since package restore is turned on, this folder can be left out of source control.  It will be rebuilt by NuGet during a build, whenever the folder is missing or incomplete:

```
src/packages
```

<a href="https://github.com/fixie/fixie/commit/2cf6ed22a1008787f33bc7d07d0fc0816393d90f">Commit.</a>

## Install NUnit and NUnit.Runners via NuGet

Fixie is a test framework, so it may seem odd to start using NUnit during its development.  However, there's a bare minimum of functionality that I need to implement before Fixie can be used to test itself, and in the meantime I don't want to fall into the tempting trap of "coding without a net".  In the meantime, I'll use NUnit.

There are two dependencies we want to include via NuGet: NUnit and NUnit.Runners.  In Solution Explorer, right-click the solution and go to "Manage NuGet Packages for Solution."  In this dialog, search for and install the NUnit and NUnit.Runners packages.  When installing NUnit, select the Fixie.Tests project so that it will add a reference to Nunit.Framework.dll for us.

I also added the Shouldly test assertion library, again referenced by Fixie.Tests, but use whatever assertion style you are comfortable with.

<a href="https://github.com/fixie/fixie/commit/92cb56fd553639da0b2127f4fcefc408cfb40078">Commit</a>.

## Add a Build Script to Compile and Run Tests

Any new developer should be able to clone your repo and run "build" at a command line in order to compile the solution and run all the tests.  Here, we'll use a third-party PowerShell tool called "psake" to simplify writing such a script.

> Working with PowerShell can be insanely frustrating.  The language behavior for seemingly-simple tasks is often the exact opposite of anything you would expect when compared to any other language.  I would rather work with COBOL.  In Antarctica.  Without a coat.  And there's bees.  Bees everywhere.
>
> Given that, it's good to start out by copying a script you know already works, and tweaking it from there, so feel free to use today's scripts as a starting point for your project, too.


Just like we did for NUnit.Runners, right-click the solution and install the package named "psake".

In Fixie.Tests, I added a single test **that always fails**:

```cs
using NUnit.Framework;

namespace Fixie.Tests
{
    //TODO: Remove this test fixture once we have any real tests in place.

    [TestFixture]
    public class BuildTests
    {
        [Test]
        public void ConfirmBuildWorks()
        {
            Assert.Fail("Always fail, so we can confirm the build is running our tests.")
        }
    }
}
```

We'll know we are done writing our build script if we can run the build, witness the test failing, fix the test, rerun the build, and witness the test passing.  This process will give us confidence that actual errors will be properly reported to us later.

At the root of the repo, outside the /src folder, add a build script written in PowerShell.  This example just builds the solution and runs the tests:

```
Framework '4.0'

properties {
  $project = 'Fixie'
  $configuration = 'Release'
  $src = resolve-path '.\src'
}

task default -depends Test

task Test -depends Compile {
    $nunitRunner = join-path $src "packages\NUnit.Runners.2.6.2\tools\nunit-console.exe"
    & $nunitRunner $src\$project.Tests\bin\$configuration\$project.Tests.dll /nologo /nodots /framework:net-4.0

    if ($lastexitcode -gt 0)
    {
        throw "{0} unit tests failed." -f $lastexitcode
    }
    if ($lastexitcode -lt 0)
    {
        throw "Unit test run was terminated by a fatal error."
    }
}

task Compile {
  exec { msbuild /t:clean /v:q /nologo /p:Configuration=$configuration $src\$project.sln }
  exec { msbuild /t:build /v:q /nologo /p:Configuration=$configuration $src\$project.sln }
}
```

This script is meant to run with psake, but recall that psake's files all live under the /src/packages NuGet folder, *which is not in source control*.  We've got a bit of a chicken-and-egg problem here: we want NuGet to step in during the compile step in order to download any missing dependencies *like psake*, but we need psake to exist in the first place to run our build script and kick off that download!

The solution to that problem is to include a build.cmd batch file, which starts by asking NuGet.exe to install solution-wide dependencies (including psake), and *then* invoke psake to run our default.ps1 file:

```
@echo off

.\src\.nuget\nuget.exe install src\.nuget\packages.config -source "https://nuget.org/api/v2/" -RequireConsent -o "src\packages"

powershell -NoProfile -ExecutionPolicy Bypass -Command "& '%~dp0\src\packages\psake.4.2.0.1\tools\psake.ps1' %*; if ($psake.build_success -eq $false) { write-host "Build Failed!" -fore RED; exit 1 } else { exit 0 }"
```

When someone clones the repo and runs "build" at the command line, the results in the console should show the dependencies successfully downloaded (if they were missing), the solution compiled successfully, and the single unit test executed but failed.  After changing the test to always pass, rerunning the script should show the whole build-and-test process completes successfully.

I encourage the habit of running "build" at the command line manually prior to any commit.  Speaking of commits,

<a href="https://github.com/fixie/fixie/commit/c45ac7a6c50c1be995303b70a85396c89b8fb768">Commit</a>.

## Version Your Assemblies with Common AssemblyInfo

By default, every Project in a Visual Studio Solution contains a file called AssemblyInfo.cs under a Properties folder.  It's easy to forget that this exists and actually needs to be updated periodically.  The values in these files live on in the compiled assemblies.  When releasing our libraries/tools to the public, we should be good citizens by keeping this information up to date, especially the version numbers.  That sounds tiresome and error-prone, so let's automate as much as possible.

Versioning assemblies is actually a complex topic, and lots of high-profile projects do it a little differently from each other, so let's err on the safe side while also keeping it simple. If our versioning plans need to grow into something more involved later, we can always change it then.

Note that .NET version numbers have four parts.  Let's reserve the first three as special, to be changed only when a human decides that it's time to bump the version to reflect the amount of change that has gone on in the project.  Let's reserve the fourth number to be set by our CI server.

> A Continuous Integration (CI) server is any tool that monitors our source control for new commits, automatically checking out the latest, running our build script, and reporting on success or failure.  We're all human, and we're bound to occasionally make changes that don't pass the build.  The responsible thing is to let a tool alert us whenever our humanity gets the best of us.
>
> I'll be using TeamCity, which additionally tracks an auto-incrementing integer indicating the build number.  We'll use the build number as the fourth part of our assemblies' version numbers.


**VERSION.txt** - Store the human-determined first three numbers in a plain text file right beside our build script.

```
0.0.1
```

**CommonAssemblyInfo.cs** - Rather than maintain redundant information in separate AssemblyInfo.cs files, we'll put all of the common parts in a single file, and then link each Project in the Solution to that common file.  Our build script will read the contents of VERSION.txt and generate src/CommonAssemblyInfo.cs upon each build.  When we run the build ourselves, we'll just assume ".0" as the fourth part of the version, since locally-run builds are just for local development purposes.  When our CI server runs the build, we'll pull the actual build number out of the air and use that for the fourth part.  Therefore, we'll always see ".0" in source control's CommonAssemblyInfo.cs, but the DLLs produced on the CI server will have a complete version number for sharing with the world.

I updated the build script to produce a CommonAssemblyInfo.cs file with the build number filled in:

```
Framework '4.0'

properties {
    $project = 'Fixie'
    $birthYear = 2013
    $maintainers = "Patrick Lioi"

    $configuration = 'Release'
    $src = resolve-path '.\src'
    $build = if ($env:build_number -ne $NULL) { $env:build_number } else { '0' }
    $version = [IO.File]::ReadAllText('.\VERSION.txt') + '.' + $build
}

task default -depends Test

task Test -depends Compile {
    $nunitRunner = join-path $src "packages\NUnit.Runners.2.6.2\tools\nunit-console.exe"
    & $nunitRunner $src\$project.Tests\bin\$configuration\$project.Tests.dll /nologo /nodots /framework:net-4.0

    if ($lastexitcode -gt 0)
    {
        throw "{0} unit tests failed." -f $lastexitcode
    }
    if ($lastexitcode -lt 0)
    {
        throw "Unit test run was terminated by a fatal error."
    }
}

task Compile -depends CommonAssemblyInfo {
  exec { msbuild /t:clean /v:q /nologo /p:Configuration=$configuration $src\$project.sln }
  exec { msbuild /t:build /v:q /nologo /p:Configuration=$configuration $src\$project.sln }
}

task CommonAssemblyInfo {
    $date = Get-Date
    $year = $date.Year
    $copyrightSpan = if ($year -eq $birthYear) { $year } else { "$birthYear-$year" }
    $copyright = "Copyright (c) $copyrightSpan $maintainers"

"using System.Reflection;
using System.Runtime.InteropServices;

[assembly: ComVisible(false)]
[assembly: AssemblyProduct(""$project"")]
[assembly: AssemblyVersion(""$version"")]
[assembly: AssemblyFileVersion(""$version"")]
[assembly: AssemblyCopyright(""$copyright"")]
[assembly: AssemblyConfiguration(""$configuration"")]" | out-file "$src\CommonAssemblyInfo.cs" -encoding "ASCII"
}
```

I ran the script and confirmed the contents of CommonAssemblyInfo.cs.  Next, I needed to make both Projects in the Solution actually use this file instead of the defaults.  In both Projects, I removed most of the contents of Properties/AssemblyInfo.cs (leaving only the Project-specific [AssemblyName] attribute), right-clicked the Project to Add \ Existing Item, browsed to CommonAssemblyInfo.cs *and clicked the down-arrow within the Add button to select Add As Link* so that a single copy of the file would be used by all projects.  Once these links were created, I dragged them into the Properties folders to get them out of the way.  Whenever I add a new project to the solution, I'll need to repeat this step.

<a href="https://github.com/fixie/fixie/commit/d861ff8fb6bc5621c7066a855fa96733cbe7eebf">Commit</a>.

## Footwear Accomplished

Phew!  These first steps can be frustrating, and a little boring, but they pay for themselves very quickly.  Now that we have a reliable build script, a simple NuGet-friendly file structure, and a simple versioning scheme, we can finally dive into development.

Socks fully donned, next week, we'll stretch the footwear metaphor a bit further, as we "bootstrap" Fixie to the point where it can test itself!
