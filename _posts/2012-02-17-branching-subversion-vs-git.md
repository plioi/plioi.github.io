---
title: "Branching: Subversion vs Git"
layout: post
---


At a previous job, I worked for a company that used Subversion for all of its projects.  This was a welcome change coming from a TFS and Visual Source Safe background.

We did relatively little work with branches.  For the most part, a branch was created only to represent the state we wanted to deploy to production.  In other words, everybody worked against trunk, and when we were happy with what we had we would create a branch with the version number as its name, and we would deploy from there.  If a hotfix was needed, at least we didn't have to worry about deploying it along with incomplete features under development; we'd base the hotfix on the version branch and deploy that.

On one occasion, someone needed to make a fairly drastic change to the system.  It was a little risky, and would have interfered with the rest of the team if it was performed against trunk, so they made a feature branch for it.  About a month later, I was tasked with merging that back into trunk.

**This was, shall we say, an Interesting Merge.**

Throughout development of the branch, work being performed in the trunk had *not* been periodically merged into the branch, meaning that there were many, many conflicts without an obvious resolution.  Ultimately things worked out, but it didn't seem like a great way to do things.  With my lack of exposure to alternatives, I didn't really see the great benefit of frequent branching.

## Sane Branching

**The whole concept of branching was intimidating to me because the *way* we did it caused more pain than it alleviated.**

Things have gotten better since then.  I've been able to experience a sane feature branching strategy with Subversion over the past year, and the pain was far outweighed by the benefits.  Throughout development of a feature branch, you periodically merge from the parent branch so that the effort of merging takes place *away* from the main line and in small doses.  When it's time to merge the feature branch back to the parent, there's very little risk involved because you already know that your changes work in concert with your coworkers' changes.

It's still a little frustrating with Subversion, though.  Branching and switching between branches are big and slow operations because they're dealing with point-in-time copies of Big Honking Folders.  It's almost a *coincidence* that Subversion is aware of the history involved.

## Saner Branching

Git has *again* changed my mind about branching.  **With Subversion, a branch is a whole new commit containing a point-in-time copy of your code.  With git, a branch is a named pointer to a commit you've already made.**  You get a little less noise in your history because you don't have to wade through phony commits: some Subversion commits represent real work performed, and some just represent the act of branching, which doesn't really *tell* me something.

Creating a branch in git is extremely fast.  It doesn't take long to write a commit ID to a local file, and that's all there is to it.  Git lets all of your branches use the same workspace, and switching from one branch to another is also fast.  I find myself creating branches way more often than before.  It's so fast and so simple compared to Subversion that speed/complexity no longer have to come into play when deciding whether or not to create one.  **I'm about to start working on a new train of thought, so I branch.**
