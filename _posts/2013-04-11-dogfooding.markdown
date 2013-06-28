---
title: Dogfooding
layout: post
---

As soon as your software project has a useful feature or two, it's time to start [eating your own dogfood](http://en.wikipedia.org/wiki/Eating_your_own_dog_food). The usual advice is to use your own software in order to get early feedback, but there's another major benefit to dogfooding that usually goes unmentioned:

<blockquote>To dogfood your software, you have to treat its *deployment* as a first-class feature.</blockquote>

Deployment is not some secondary activity to be figured out later. Just like we should use our products early and often to make them better, we should use our deployment processes early and often to make *them* better.

## Dogfooding Fixie

Now that my test framework is powerful enough to [run all of its own tests](http://www.headspring.com/patrick/bootstrapping/), has a [command line test runner](https://github.com/plioi/fixie/blob/9a124ba6c460cf93c1507be68622245033f30454/src/Fixie.Console/Program.cs), and [integrates with TestDriven.NET](https://github.com/plioi/fixie/blob/9a124ba6c460cf93c1507be68622245033f30454/src/Fixie.TestDriven/Runner.cs), it's time to start dogfooding it.

Deploying a test framework involves making it available for use in other projects. For this project, that means publishing a [Fixie NuGet package](http://nuget.org/packages/Fixie) to the NuGet Gallery. When another developer installs the package, they should gain three things: an assembly reference added to their test project, the console test runner EXE, and TestDriven.NET support.

Since dogfooding a project demands treating its deployment as a first-class feature, I needed to automate as much of the NuGet work as possible. I have set up a one-click deployment process for Fixie, and have tried it out for real by installing it into two other open source projects.

## Creating and Publishing the NuGet Package

First, I added a nuspec file which describes the package contents.  I named it [Fixie.nuspec](https://github.com/plioi/fixie/blob/a4a358e45e5c1ef2aa6074f12d1075066d4e28ca/src/Fixie/Fixie.nuspec) and placed this beside the Fixie.csproj file. Recall that the [AssemblyInfo values in this project are set by the build script](http://www.headspring.com/patrick/socks-then-shoes/).  By naming the nuspec file after the csproj file, NuGet.exe will know to use those same values as replacements for the $tokens$ in the nuspec:

{% gist 5359835 %}

Fixie.dll will be included in the package automatically, since that is the output of compiling Fixie.csproj.  The <code>&lt;files&gt;</code> section lists additional files I needed to include in the package: the console runner, its config, and 3 files needed to integrate with TestDriven.NET.  Since we're including some files we don't always want to add as project references during installation, I explicitly list Fixie.dll as the only reference.

Dropping this nuspec file into the project isn't enough.  I also added a 'Package' task to the [build script](https://github.com/plioi/fixie/blob/a4a358e45e5c1ef2aa6074f12d1075066d4e28ca/default.ps1), which runs NuGet.exe against the nuspec file and produces the deployable package.  This task is the one executed by TeamCity upon each commit to GitHub:

{% gist 5359834 %}

TeamCity creates the NuGet package files with each build, but I don't really want to publish a package to the world upon each commit. I'd rather just publish when I know I've made enough changes to warrant a new deployment.  I found some good advice on how to [selectively publish NuGet packages with TeamCity](http://blog.jonnyzzz.name/2011/09/selective-publishing-of-nuget-packages.html).  In short, my main TeamCity build configuration compiles, runs tests, and creates NuGet package files automatically upon each commit to GitHub, while  a secondary TeamCity build handles publishing the latest successful package to the world.  The secondary TeamCity build only runs when I decide to run it, giving me one-click deployment.

## Inevitable Explosions

Applying this NuGet package to the first target project, [Parsley](https://github.com/plioi/parsley), was a complete success.  That's unfortunate, because it is boring.  It gave me no new information to drive Fixie's development.  Parsley's tests spend all of their time twiddling and comparing strings. Not much can go wrong here that would depend on the details of the host test framework.

Applying this NuGet package to the second target project, [Rook](https://github.com/plioi/rook), has *fortunately* proven very difficult, yielding far more interesting results.

First, Rook's integration tests need to read text files from a folder found beside the tests' own DLL, meaning the tests depend on a test framework's own notion of where the current directory is.  The fix here was easy: [Fixie's console runner changes the current directory to the test assembly location during execution, and then reverts to the previous directory](https://github.com/plioi/fixie/commit/9a124ba6c460cf93c1507be68622245033f30454).

Second, this project's integration tests actually produce some additional DLLs at runtime and call into them, and those DLLs may depend on *other* DLLs that live beside the tests' DLL.  These dependencies are not being found.  That may sound like the same problem: .NET wants to look in the wrong directory for some DLLs.  Unfortunately we're not talking about the operating system's concept of a current directory.  Instead, we're talking about the .NET concept of an AppDomain and its "base directory", and *that* topic is a can of worms for another day.

Dogfooding Fixie on two real projects has given me valuable feedback.  I ran into two similar issues and have realized one major requirement that has not been on my radar so far:

<blockquote>A test framework should fool your test DLL into thinking *it* is an EXE running in the build output folder, when in fact the running EXE is the console runner off in some other folder.</blockquote>

With this week's current directory fix and the upcoming fix regarding AppDomains, *whatever the heck those are*, I'll be able to satisfy this new requirement.  Stay tuned!