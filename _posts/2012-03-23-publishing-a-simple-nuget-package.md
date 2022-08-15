---
title: "Publishing a Simple NuGet Package"
layout: post
---


<a href="http://nuget.org/">NuGet</a> is a Visual Studio extension that makes it easy to pull third-party libraries into your projects.  It can also bring in their dependencies, along the same lines as gems for Ruby development.

I recently <a href="http://patrick.lioi.net/2012/03/02/migrating-from-bitbucket-to-github/">migrated Headspring's Enumeration class to GitHub</a>.  This is a simple class that, for the most part, replaces the need for .NET enum types.  One of the simplest things a NuGet package can do is add a file to your project, so creating and publishing an updated NuGet package for Enumeration was a great way to get my feet wet.

Unlike most things, it was as easy as it should be.

First, I created an account on <a href="http://nuget.org">nuget.org</a> and installed the extension using the link on the front page.  Next, I downloaded the <a href="http://nuget.codeplex.com/releases/view/58939">NuGet.exe command-line tool</a> and made sure it was included in my PATH.  On my account profile page, I'm given the chance to see my API Key.  Keep this secret.  At the command line, we need to issue a one-time command so that this key can be active during subsequent nuget commands:

`$ nuget setApiKey 123-abc-123-abc-123`

You create a *.nuspec file which describes the files you want to publish.  Then you ask nuget.exe to turn that file into a *.nupkg file.  A nupkg file is actually a zip file containing the files to be deployed as well as some metadata.

At this point, the project consisted of 3 files: a README, LICENSE.txt, and Enumeration.cs.  While in this folder, I issued a command to create a skeleton of a nuspec file:

`$ nuget spec`

This creates Package.nuspec, which I renamed to Enumeration.nuspec and populated:

```
<?xml version="1.0"?>
<package>
    <metadata>
        <id>Enumeration</id>
        <version>1.0.4</version>
        <authors>Headspring</authors>
        <owners>Headspring</owners>
        <licenseUrl>https://github.com/HeadspringLabs/Enumeration/blob/master/content/Headspring/LICENSE.txt</licenseUrl>
        <projectUrl>https://github.com/HeadspringLabs/Enumeration</projectUrl>
        <iconUrl>https://bitbucket-assetroot.s3.amazonaws.com/c/photos/2010/May/14/spray_avatar.png</iconUrl>
        <requireLicenseAcceptance>false</requireLicenseAcceptance>
        <description>A utility class. Common uses include Java-like enumerations, dispatch table, flyweight, and drop-down list items/select options.</description>
        <tags>utility enumeration</tags>
    </metadata>
    <files>
        <file src="Enumeration.cs" target="content\Headspring\Enumeration.cs" />
        <file src="LICENSE.txt" target="content\Headspring\LICENSE.txt" />
    </files>
</package>
```

There isn't much going on here.  We say what version number we want to publish, a few descriptive fields, and then list the files we want to be included in the nupkg 'zip' file.  In the file tags, `src` refers to the files relative to the location of the nuspec file, and `target` describes the path we want to copy it to *within* the zip.  Starting these paths with "content" means that when someone installs the package into one of their own projects, the files will be copied and then added to the target project file.

To publish, we just need to issue the following commands:

```
$ nuget pack Enumeration.nuspec
$ nuget push Enumeration.1.0.4.nupkg
$ del Enumeration.1.0.4.nupkg
```


`nuget pack` converts a nuspec into a nupkg, and `nuget push` submits it to <a href="http://nuget.org/">nuget.org</a>.  After that, we don't need to keep a copy of the nupgk, so we delete it.

That's it!  Next week, we'll cover the slightly more involved scenario of publishing a whole open-source library.
