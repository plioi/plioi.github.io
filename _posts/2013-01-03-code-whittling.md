---
title: "Code Whittling"
layout: post
---


Reading other people's code is a valuable learning exercise, but it can be very difficult no matter how experienced or talented the original developers were.  It's tough to read other people's code, especially on projects you are unfamiliar with.  You may be a fan of an open-source project, and want to learn from it, but get lost once you start trying to read through it.  Even if the project was exceptionally well-written, the size of the project and its unfamiliar concepts may be obstacles to an outside observer.

In recent months, I tried to learn how two open-source projects work: <a href="https://github.com/hibernating-rhinos/rhino-licensing">Rhino Licensing</a> and <a href="http://xunit.codeplex.com/">xUnit</a>.  In each case, I immediately ran into the typical problem: I didn't know where to start to really get a feel for how these projects work, and the thought of just opening Solution Explorer and reading through each file in random order just didn't seem like it would be useful.

Our brains don't work like IDEs.  I can't just read 100,000 lines of code and expect to retain enough of that information to be useful.  Instead, I took a different tack.  When I'm actually working on a team project, I don't find myself sitting still, idly reading thousands of lines of code straight through.  That would be ridiculous, so why was I about to do precisely that to learn about these other projects?

Instead of pointlessly trying to read and remember everything, I wanted to pursue two other goals simultaneously: engaging with the code like an active contributor while discovering and honing in on the most relevant parts.

**Act like a contributor:**  When working on a real project for a client, I don't just sit there and read the team's contributions straight through, for the same reason that didn't work while reading other people's open-source projects.  Instead, we all gain an understanding of a project over time, *through active and repeated interaction with the code*.  I need to implement feature X, which is similar to feature Y, so I start searching for feature Y.  Maybe I compare its implementation to the user interface.  Maybe I see something surprising that I would have forgotten to do in my own feature.  Heck, maybe I find a bug, or a way to make both features share most of their implementation.  Maybe it's not obvious how a particular identifier affects the system, so I do an <a href="http://patrick.lioi.net/2012/03/16/escape-analysis/">escape analysis</a> for it...

In all of these activities, our hands are glued to keyboards: writing code, refactoring code, searching through code paths, and the like.  If we simulate these activities while trying to learn about someone else's open-source project, we are more likely to benefit from the activity in the same way we benefit from these activities on our own "real" projects.

**Hone in on what's relevant:**  Large projects are daunting, and small projects aren't.  Usually, there is a particular part of a project you happen to use the most and are interested in learning about.  Everything else in the project is a distracting obstacle.  By deliberately removing the parts that aren't too important to you, it'll be easier to focus on the parts you do care about.

I like to think of this combined activity as "Code Whittling".  You destroy a project from the outside in, learning about it as you go.

## Destroying a Project

I started by creating a throwaway clone of the tool I wanted to learn about.  Then, I thought of its features from the point of view of someone *using* it within their own project, and identified the main features that I *don't* care about.  These feature may very well be useful, but were unrelated to the ones I really wanted to learn about.  I started violently removing those "irrelevant" features left and right.  After gutting the major features I wasn't interested in learning about, I was left with a smaller task and had a better feel for the high-level organization of the project.

Even at this point, though, I didn't stop whittling.  Within the features I *did* care about, there were now some unused classes, unused methods, and unused fields, some identified by ReSharper.  I started removing these, too, which in some cases had a cascading effect.  This step left me with the code that I really wanted to learn about.

Armed with the confidence of working with something that was now smaller and somewhat familiar, I started applying small refactorings.  Within a pefectly good and still-used method, I would try to rewrite parts in my own style.  "If I was the original contributor, how would I have implemented *this* tiny bit here?" or, "If I was going to write an alternative project, what would I have named *that* concept?"

I found that actually *working* with the code like this, even though the "work" itself was not strictly productive, helped me to get a better understanding of the project than I would have gotten simply reading through it all.  This activity kept me focused, and helped me to see the major concepts through the eyes of the real contributors, all while honing in on the information I was originally after.

In the coming weeks, we'll walk through this process on a few open-source projects, starting with my previous experiences whittling Rhino Licensing and xUnit.
