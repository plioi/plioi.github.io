---
title: "Memory Management"
layout: post
---


Sometimes I get a reputation for having a good memory, but I actually don't trust it at all.  When it comes to my short-term memory, I find that simple things slip my mind far too easily if I don't place an unusual amount of concentration on them, so "automatic" tasks like locking a door fade too quickly and I'll immediately wonder whether I actually did them.

> Wait. Did I leave the stove on?

I trust my long-term memory only slightly more.  Maybe it's all in my head, but the distrust is real.  Short-term and long-term memory matter when working on a large software project, so how do I deal with my memory distrust? **I. Write. Down. *Everything.*** 

## Typical Workflow

If you peek over my shoulder on a typical day, you'll see Visual Studio open on one monitor and Sublime Text open on the other.  I usually spend more keystrokes typing into Sublime than VS.

I typically have one text file per topic branch, naturally with the same name as that branch.  As I work on a feature or diagnose a bug, my train of thought winds up in that file.  I document experiments, research of the relevant parts of the preexisting source code, key steps taken, and the like.

When I'm working on a brand new feature, I may not have much to document, as the code I'm writing *is* the train of thought and there's no point in being redundant about this.  When working on a difficult bug, on the other hand, just about every word that crosses my mind winds up in that file.  Even the puns.  (Especially the puns.)

## Bug Hunting

Today, I was researching a particularly tricky bug.  The code involved callbacks within callbacks within callbacks, each of which was named 'callback'.  Fortunately, the bug was trivially reproducible.  Unfortunately, stepping through the code wasn't very illuminating.  It became way too easy to lose track of the train of thought.

In order to maintain a useful train of thought, I had to start getting very specific in my notes.  What exact code paths were taken for *this* input, what exact code paths were taken for *that* input, where did they diverge and why, etc.

I had some false starts, in which I thought I knew what was going on and later discovered evidence to the contrary.  When that happens, I make a point of keeping the details handy and permanent.  As with scientists experimenting in a lab, *a failed coding "experiment" is information* contributing to the larger effort.

As I got closer and closer to the actual culprit, these notes and experiments got clearer and clearer.  The little bits of information gleaned in the early stages were starting to congeal into a relatively simple and accurate description of the problem.  Once that description became clear enough, it was obvious what I needed to do to solve the problem.  I spiked out a quick proof of concept for the solution, confirmed success, and then realized the end of the day had come and I was better off tackling the remainder of the work with fresh eyes in the morning.

## Cool Story Bro, But Why?

If I had too much trust in my memory, I wouldn't have this document handy first thing tomorrow morning.  Without the document, I would have faced a tough choice around 5:00 today: take a risk and work long mistake-prone hours through the evening while the details were fresh, or take a risk of forgetting it all by morning.  No, thanks.

**First thing tomorrow, I'll be able to relive my entire train of thought in about 5 minutes.**  I'll be entirely ready to fix the issue while in a responsible state of mind.

## Coming Soon to A&E, *Hoarders: Text Edition*

I keep these notes around forever, for the same reason source control tools keep past commits around forever.

Recently, a coworker remembered that I worked in an area of the project that needed to be extended.  He asked if I remembered anything about what I had done, and naturally I had absolutely no idea what he was talking about.  I was certain I had never worked on such a thing.  I did, it turns out, work on it about a month ago.  A quick keyword search through my notes folder revealed my entire train of thought from that day, complete with the topic branch name and date so we could find the original commit history.

A few months back, I was working on a fairly complex overhaul of the way certain events were detected in the UI.  This work involved wrestling with some inadequacies in a third-party UI control.  While fighting with that third-party control, I had performed some experiments with the available events.  One event's name and documentation made it seem like a *perfect* fit, but my attempts revealed a subtle and serious reason to avoid it.  I documented this decision even though I never expected to revisit that event.  A week later, another developer said he'd found this event and suggested we use it; armed with the details, I could say that it was already attempted, failed, and failed *in this specific/subtle/serious way*.  We saved ourselves from wasted effort and from the cost of introducing a bug.

All in all, I think the time I spend keeping these notes saves me time and improves the code I write.  They keep me focused in the moment, prevent rework in later moments, and allow me to hit the ground running each day.
