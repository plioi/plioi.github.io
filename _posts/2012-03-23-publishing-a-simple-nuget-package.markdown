---
title: Publishing a Simple NuGet Package
layout: post
---

<a href="http://nuget.org/">NuGet</a> is a Visual Studio extension that makes it easy to pull third-party libraries into your projects.  It can also bring in their dependencies, along the same lines as gems for Ruby development.

I recently <a href="http://www.headspring.com/2012/03/migrating-from-bitbucket-to-github">migrated Headspring's Enumeration class to GitHub</a>.  This is a simple class that, for the most part, replaces the need for .NET enum types.  One of the simplest things a NuGet package can do is add a file to your project, so creating and publishing an updated NuGet package for Enumeration was a great way to get my feet wet.

Unlike most things, it was as easy as it should be.<!--more-->

First, I created an account on <a href="http://nuget.org">nuget.org</a> and installed the extension using the link on the front page.  Next, I downloaded the <a href="http://nuget.codeplex.com/releases/view/58939">NuGet.exe command-line tool</a> and made sure it was included in my PATH.  On my account profile page, I'm given the chance to see my API Key.  Keep this secret.  At the command line, we need to issue a one-time command so that this key can be active during subsequent nuget commands:

<code>$ nuget setApiKey 123-abc-123-abc-123</code>

You create a *.nuspec file which describes the files you want to publish.  Then you ask nuget.exe to turn that file into a *.nupkg file.  A nupkg file is actually a zip file containing the files to be deployed as well as some metadata.

At this point, the project consisted of 3 files: a README, LICENSE.txt, and Enumeration.cs.  While in this folder, I issued a command to create a skeleton of a nuspec file:

<code>$ nuget spec</code>

This creates Package.nuspec, which I renamed to Enumeration.nuspec and populated:

[gist id="2155348"]

There isn't much going on here.  We say what version number we want to publish, a few descriptive fields, and then list the files we want to be included in the nupkg 'zip' file.  In the file tags, <code>src</code> refers to the files relative to the location of the nuspec file, and <code>target</code> describes the path we want to copy it to <em>within</em> the zip.  Starting these paths with "content" means that when someone installs the package into one of their own projects, the files will be copied and then added to the target project file.

To publish, we just need to issue the following commands:

<p><code>$ nuget pack Enumeration.nuspec</code>
<code>$ nuget push Enumeration.1.0.4.nupkg</code>
<code>$ del Enumeration.1.0.4.nupkg</code></p>

<code>nuget pack</code> converts a nuspec into a nupkg, and <code>nuget push</code> submits it to <a href="http://nuget.org/">nuget.org</a>.  After that, we don't need to keep a copy of the nupgk, so we delete it.

That's it!  Next week, we'll cover the slightly more involved scenario of publishing a whole open-source library.
