---
title: "Publishing a Library to SymbolSource.org"
layout: post
---


Last week, we saw how to <a href="http://patrick.lioi.net/2012/03/30/publishing-a-library-with-nuget/">publish a library with NuGet</a>.  By including DLL files in our NuGet package under the special "lib" folder, NuGet knows to install them as assembly references in the target project.  This is a great way to get your open-source library into the hands of your audience, but it also provides them with a pretty lame debugging scenario.

## Black Box Debugging

Using that NuGet package from last week, I installed <a href="https://github.com/plioi/parsley">Parsley</a> into another project that consumes it.  While trying to debug *that* project, I got to a point where I wanted to just step the debugger into one of Parsley's methods.  Even with the PDB file included in the NuGet package, Visual Studio just plain didn't have enough information to let me step through Parsley's code.

Too much information about the original source gets lost when a PDB is produced.  We need to make the source code available to the consumer when they install our packages, and we need to so in a way that Visual Studio can take advantage of at debugging time.

## Publishing a Symbol Package

Fortunately there is a free service, <a href="http://SymbolSource.org">SymbolSource.org</a>, which is easy to integrate with both NuGet (for the publisher) and Visual Studio (for the consumer).  Although getting a SymbolSource account is an option, you won't actually need one.  You'll be able to publish a special variant of your NuGet package to NuGet.org, and then NuGet.org shakes hands with SymbolSource.org on your behalf.  All it takes is a slight tweak to your `.nuspec` file, and an extra command-line argument when converting that `.nuspec` into a `.nupkg`!

The previous version of my `.nuspec`'s `files` tag just copied DLLs and PDBs into the package's special "lib" folder:

```
<files>
    <file src="$package_dir\*.dll" target="lib\net40" />
    <file src="$package_dir\*.pdb" target="lib\net40" />
</files>
```

The new version also copies all of the source code to the package's special "src" folder.  We don't need to copy auxiliary files like `.csproj` or `.sln`.  We just need to copy the code files that the debugger will need to display, namely all the *.cs files:

```
<files>
    <file src="$package_dir\*.dll" target="lib\net40" />
    <file src="$package_dir\*.pdb" target="lib\net40" />
    <file src="$source_dir\**\*.cs" target="src" />
</files>
```

I used this single `.nuspec` to produce *two* `.nupkg` files, by including the `-Symbols` command line argument:

```
$ nuget pack Parsley.nuspec -Symbols
```


This produced Parsley.0.0.2.nupkg like normal, but it also produced Parsley.0.0.2.symbols.nupkg.  NuGet.exe is smart enough to know that the first package only needed the DLLs (used at installation time), and that the symbol package needs to contain the PDBs and C# files (used at debugging time).  When I pushed the main package to NuGet.org, NuGet.exe noticed the symbol package was also present, and submitted that one too:

```
$ nuget push Parsley.0.0.2.nupkg
Pushing Parsley 0.0.2 to the NuGet gallery (https://www.nuget.org)...
Your package was pushed.
Pushing Parsley 0.0.2 to the symbol server (http://nuget.gw.symbolsource/Public/NuGet)...
Your package was pushed.
```


I then followed the instructions for <a href="http://www.symbolsource.org/Public/Home/VisualStudio">configuring Visual Studio to work with SymbolSource.org</a>.  Remember, you don't have to follow the suggestion to create a SymbolSource account.

I updated the package in the consumer project, which installed the DLLs.  The next time I tried to step into a Parsley method, I got a quick dialog that said it was downloading the relevant files from SymbolSource, and then I was able to step through the read-only C# files found within the symbol package!
