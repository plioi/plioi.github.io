---
title: "Hey TFS, git out of my way!"
layout: post
---


For about half a year, I've been using git for personal projects.  My use case, though, has been atypical and simplistic: a single user committing locally, treating GitHub as little more than a backup.  It suited my needs to date, and at least it exposed me to a few of the advantages of git over subversion, but I knew I was missing out on a lot of the power available.

Now I'm working on a project that uses TFS for source control.  We are <del>mitigating</del> supplementing TFS with <a href="https://github.com/git-tfs/git-tfs">git-tfs</a>, a tool that allows you to work with git locally and then push your finished feature branches up to TFS.  Again, that's a little atypical and a little limited compared to a git-only project, but it's exposed me to a lot more of the concepts and commands that I have been missing out on.

TFS is completely ignorant of the fact that I'm using git.  My local git repository starts out as a copy of the entire history of the TFS trunk, and I'm free to create local feature branches off of that.  I commit as much as I want to a local feature branch, and then use a git-tfs command that batches all of my local commits up into one TFS commit, complete with the TFS commit dialog.  I can avoid TFS-style merging, instead using git commands and my favorite merge tool to incorporate upstream changes into my local copy.  Jimmy Bogard has already described <a href="http://lostechies.com/jimmybogard/2011/09/20/git-workflows-with-git-tfs/">the git-tfs workflow</a> in detail.

Before this recent new project, my progress up the git learning curve was pointlessly slow.  Although there are a lot of good tutorials out there, it just wasn't sinking in.  **What made all the difference for me was to have an experienced git user sit down with me for about an hour, physically going through the motions of the workflow.**  We started with the TFS clone, created and switched between branches, made some commits, submitted to TFS, and then pulled down upstream changes into our local repositories.  We set up two users to simultaneously modify the same files, and walked through a typical rebase.  All of a sudden, things started to sink in.  It's just one of those things you have to *do* rather than *study*.

Along the way, we had the <a href="http://code.google.com/p/gitextensions/">Git Extensions</a> browser open at all times.  We'd refresh this after executing any interesting command.  It helped a lot to do so, especially when you get to the point where you can predict what the updated screen will look like in advance.  This is an exaggeration, but **git is first and foremost a <a href="http://en.wikipedia.org/wiki/Directed_acyclic_graph">directed acyclic graph</a>-building tool, and only incidentally a source control tool**, so seeing the graph build up with each command really helps you to unlearn the metaphors that your previous source control tool etched into your brain.

With that basic foundation, it's now becoming easier to introduce myself to other commands.  If you want to put an extremely sharp tool in your tool belt, find a buddy, have a repository browser handy, and just dive in.
