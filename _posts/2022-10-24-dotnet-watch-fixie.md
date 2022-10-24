---
title: 'dotnet watch fixie'
layout: post
---

[`dotnet fixie`](https://www.nuget.org/packages/Fixie.Console) pairs naturally with [`dotnet watch`](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-watch). By initiating a watch, you can automatically rebuild and run tests in response to file saves in your IDE.

`dotnet watch` will initiate a long running "watch" for changes in your code, and will invoke the command of your choosing each time it detects *impactful* file changes. `dotnet fixie` itself performs the equivalent of `dotnet build`, so if we can combine these commands into one we can simplify the frequent "save, build, test" loop we perform a hundred times per day.

If you try to use `dotnet watch` for the first time, though, you're bound to run into a few obstacles. Most examples involve combining it with `dotnet run`, for instance, which has a few limitations of its own such as its need for the indicated project to only have one `TargetFramework`, or at least to specify a single framework as an additional command line argument. Tests for an open source tool or library, though, tend to indicate multiple `TargetFrameworks` where the runtime behavior of the tests may meaningfully vary. It may seem like `dotnet watch` is just a bad fit for test running in such a solution.

Let's build up a realistic test running watcher from scratch, aiming for the fewest surprises and limitations.

We start with the `dotnet fixie` command we would repeatedly run ourselves if `dotnet watch` didn't exist:

```sh
dotnet fixie *.Tests
```

This runs tests in all the solution's projects whose name match our naming convention wildcard, `*.Tests`. It also builds those projects, so at least we don't have to start with a separate `dotnet build`. Still, it'd be nice if I didn't have to shift focus from my IDE to my terminal and reissue that command every time I make a change.

Next, we try a `dotnet watch` command that seems like it might work, but won't:

```sh
dotnet watch fixie *.Tests

dotnet watch ❌ Could not find a MSBuild project file in 'C:\the\current\path'.
Specify which project to use with the --project option.
```

This is unfortunate, and at least a little misleading. `dotnet watch` must be directed towards a single project in your solution. That might make it sound like it will ony react to code changes *within* that project, ignoring the rest of your solution. Instead, `dotnet watch` will detect any change that *impacts* the indicated project, by inspecting the whole project dependency graph. It won't look at your whole solution, necessarily, but since a test project often does include the whole solution in its dependencies, **an "inclusive" test project makes for a very good selection here.** We improve our command by telling `dotnet watch`, "Here is a pivotal project that will give you a great deal of 'reach' into the solution. *Most* code changes that I care to react to will be detectable if you watch for impact to *this* project."

```sh
dotnet watch --project path/to/Most.Inclusive.Tests fixie *.Tests
```

Let's break down this command to understand it in parts:

* `--project path/to/Most.Inclusive.Tests` is an argument to `dotnet watch`, indicating which project to watch. Any code changes in that project or any of its dependencies will provoke the watcher to run the rest of our command. It's the best we can do, but that is often enough.

* `fixie` is the `dotnet ...` command that `watch` will invoke for us each time it detects changes impacting the `--project`.

* `*.Tests` is an argument to the implied `dotnet fixie ...`, the original inclusive project name wildcard, ensuring that *all* of the solution's tests will run, not just those in the given `--project`.

Having to specify `--project` is a little unfortunate. We just have to remember that it is not a limitation of what test projects will run, but instead it is a helping hand for the watcher to notice *any* impactful code edits across the project graph.

By selecting an inclusive `--project` and by invoking `fixie` with an inclusive project wildcard, any code changes in the solution trigger a new build and a new test run, across all test projects, across all `TargetFrameworks` specified in those test projects. When I know I'll be quickly iterating between code changes and full solution test runs, I'll have `dotnet watch --project path/to/Most.Inclusive.Tests fixie *.Tests` running in a terminal on one screen, with my IDE on the other screen. My "developer inner loop" simplifies from `Code, Save, Change Focus, Up, Enter, Change Focus...` to `Code, Save...`.