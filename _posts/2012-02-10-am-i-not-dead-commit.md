---
title: "Am I not dead?  Commit."
layout: post
---


As I mentioned last week, I've <a href="http://patrick.lioi.net/2012/02/03/hey-tfs-git-out-of-my-way/">started using git</a> recently, specifically a git-to-tfs bridge so that I can avoid as much contact with TFS as humanly possible.

Most of my recent experience has been with subversion, so I have been *unlearning* that as much as I have been *learning* git.  The most apparent difference right away is what it really means to make a "commit".

With subversion, even when working in a feature branch, my commits would be immediately shared with my coworkers.  I was basically handing a diff to my coworkers every time I checked something in.   Since anything I committed would be immediately shared, I had an extra concern at commit-time: "If I commit right now, and someone else gets latest, will I mess things up for them?".  Despite wanting to commit as often as possible, I could only allow myself to commit when I had completed *the smallest chunk of work that could be shipped if I got hit by a bus*.  This seemed like the most reasonable rule of thumb to follow, but even these commits could end up fairly large.

Subversion is like a video game with sparse save points.  You play for a while, maybe even an hour or two, before reaching a point where you're even *allowed* to save your progress.  You might even get a little lost along the way, and you can't just jump back to the last save point without wasting all that time.

Working with git has felt a lot more like playing the original <a href="http://en.wikipedia.org/wiki/Half-Life_(video_game)">Half-Life</a>.  Maybe it was really difficult, or maybe I was just terrible at it, but I found myself being squashed or blown up every other minute.  Thankfully, instead of having to make your way to a special save point, you could just save your progress at any moment or jump back to any previous save.  **Have I moved 10 feet?  Am I not dead?  Save.**

Now that I can work with a local feature branch, I can think differently about what a commit actually *is*.  If I commit far more often than I did with subversion, I harm nobody; it's all just local anyway.  When I ultimately push up to TFS, it'll batch up all my local commits into a single set of changes representing the before/after comparison of all of my work.  Any missteps along the way won't show up to my coworkers, yet while *doing* the work I could always look back to retrace my own train of thought.  **Have I made any forward progress, shippable or not?  Am I not dead? Commit.**
