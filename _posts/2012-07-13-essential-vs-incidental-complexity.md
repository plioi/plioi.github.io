---
title: "Essential vs Incidental Complexity"
layout: post
---


Confession time: I might not be perfect.  Thankfully, though, that's irrelevant.

I've coded myself into a bit of a corner, and it's all my fault, but as I said above my imperfection is irrelevant.  What's relevant is that when one is faced with a software project that's heading in the wrong direction, the problems can always be identified and addressed.  **Code is just too malleable to be intractable.**  Realizing you have a coding problem is the first step to recovery.

## What Success Feels Like

Last week, I wrote about some lessons learned from a <a href="http://patrick.lioi.net/2012/07/07/meta-yak-shaving/">yak shaving</a> project that I've been working on for the last year.  As that yak is now sufficiently sheared, I've shifted gears back to my *real* pet project, a compiler for the language summarized <a href="http://patrick.lioi.net/2012/05/25/building-a-language/">here</a>.

While working on Parsley, adding each new feature actually became *easier* over time.  I had <del>found</del> <del>planned</del> *stumbled into* the proper abstractions for the task at hand, and it was always clear what I should do next in order to move the project forward while leaving the campground cleaner than I found it.  

## Suspicious Difficulties

With Rook, I'm finding lately that adding each new feature is getting *harder*.  Quite so, actually.  Each new feature is interacting very tightly with those already implemented, and they're starting to interfere with forward progress.  I may be an hour into implementing a feature, even a simple one like adding support for comment syntax, before realizing that my attempt needs to be reverted.  Adding <a href="https://github.com/plioi/rook/commit/3a5ff34c9a8423c21608f3c876d4212eb0f9cddf">basic support for type checking of method calls</a> was the product of about 1 month and N+1 false starts, and includes more unfortunate compromises than I would usually allow myself.

**Difficulty in adding new features is a tell-tale sign that I've got the wrong abstractions in place.**  Odd, though, since the problem I'm working on is the poster child for Solved Problems.  There's a great deal of prior art I'm building on, here, so I'm confident that the overall design is actually still right.  The abstractions that I would fit into an elevator pitch are not the problem.  Probably.

## Is There Hope? (Spoiler Alert: Yes)

The question is, am I having difficulty due to **essential complexity** or **incidental complexity**?  Is the complexity inherent in the problem being solved (essential) or is it a byproduct of poor implementation details (incidental)?

I'm leaning towards incidental.  I'm not exactly breaking RSA encryption here; that's a job for quirky characters on procedural crime shows.  Since my difficulties are likely not just a matter of the problem being tough, it's a sign that these recent hurdles are really telling me something.

To sum up the situtation,

1. I have reason to believe the complexity I'm experiencing is not essential to the problem.  Rather, it's in my control.
2. My *key* abstractions are probably right, as they're the same used by Every Single Compiler Ever Made.
3. The increasing difficulty to add new features suggests that I *do* have poor abstractions somewhere.


This train of thought leads me to suspect the *other* abstractions, the ones I've been inventing myself along the way as I try to fill in the gaps of my own understanding of all that prior art.  This hunch is consistent with the last few hurdles I've run into.

This is great news!  Rather than dispair ("Oh woe is me, I done goofed!"), I have every reason to believe that the answers are now reachable.  I need to take a closer look at the abstractions I've invented along the way, the ones I've made with insufficient forethought regarding future prioritized features.  

I've focused too much time on sharpening my jigsaw's blades, and not enough time on what the puzzle is actually supposed to look like when it's done.  Lo and behond, I've forgotten to cut out the edge pieces.

Now that I know the difficulties are due to incidental complexity rather than essential complexity, I know I should slow down on adding new features while I address the problem abstractions.  Overall, my throughput will probably speed up.  I can start to work in earnest on phasing out this incidental complexity.  No "Grand Rewrite" for me!
