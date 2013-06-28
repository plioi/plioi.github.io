---
title: NuGet Package Restore 
layout: post
---

Text is easy.

Source control tools slice and dice text all day.  It's easy to compare, easy to merge, and easy to compress.  Even in a busy project, large portions of your code don't really change all that often.  Git, for instance, can pretend to store every commit as a point-in-time copy of your whole source tree by <a href="http://progit.org/book/ch1-3.html">sharing large unchanged blobs</a> of text with other commits.

Text is easy.  Binary is tough.<!--more-->

<h2>Do you pass the "New Hire Day 1" test?</h2>

We often store binaries in our source control tools, and with good reason.  We want to make it easy to spin up a new development environment.  When you hire a new developer, they should be able to pull down your project from source control <em>and just go</em>.  Dependencies such as libraries, build tools, and the like should be versioned with the project.

The alternative experience can be painful.  You show up for your first day at a new job, pull down the project, and try to run it… <strong>THUD</strong>.  It won't even build.  You try to install the right third-party libraries and tools after diagnosing each build error, and hopefully you're using the same version as everyone else.  Maybe the Dependency Gods smile upon you, and you'll be up and running before the end of the day.

It's easy to avoid these problems by including your dependencies, both libraries and tools, in source control, but can we do better?

<h2>Binary is Tough</h2>

You can store binaries in source control, but binaries are tough.  Even small changes in functionality can result in dramatically different file contents, so it's impractical to diff/merge them.  Changes in a binary file over time have to be stored without the sneaky compression tricks used for text files.

NuGet makes it easy to install libraries into your projects, but this means dumping a whole lot of DLLs into source control.  <strong>Your repo might start to grow impractically large.</strong>

<h2>Package Restore</h2>

Fortunately, though, we can have the best of both worlds.  We can provide our team with a reliable and easy way to set up new development environments, and benefit from NuGet's version/dependency smarts, without having to actually store NuGet package binaries in source control.

First, <a href="http://docs.nuget.org/docs/start-here/installing-nuget">install NuGet</a> if you haven't already.

Next, go to Solution Explorer, right click on the Solution itself, and select "Enable Nuget Package Restore".  This adds some files to a special .nuget folder, and adjusts the projects of your solution to use that folder at build time.

Lastly, note that NuGet has been placing downloaded packages under a <code>packages/</code> folder near your *.sln.  You can wipe this out.  When using git, I also add this folder to my <code>.gitignore</code> so that its contents no longer get saved to the repository.

Read that last step again.  <strong>We deliberately deleted away all the versioned packages that NuGet has been maintaining for us!</strong>

When enabling Package Restore in your solution, you changed the way that your solution builds.  When you try to build again, NuGet will notice that your projects reference specific packages that are no longer found under <code>packages/</code>.  Fortunately, your solution still contains <code>packages.config</code> files which list dependencies along with their version numbers.  Armed with these config files, NuGet will pull down the missing files into <code>packages/</code> for you right before your project gets compiled.  Upon the next build, it will just use the files already in <code>packages/</code>.
