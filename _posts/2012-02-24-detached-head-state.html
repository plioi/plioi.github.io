---
title: Detached HEAD State
layout: post
---

Today, I was investigating a pretty intricate bug, and wanted to determine whether it was introduced recently or had always been present.  I wanted to revert my workspace to an old commit to see if the problem could be reproduced there.  If not, I'd be well on my way to finding the Guilty Commit That Broke Things.  When I tried to jump back in time like this, I was presented with an unusually verbose warning message that helped to fill in a gap of my understanding of git.
<!--more-->

With git, you switch between branches by issuing the command <code>git checkout branchname</code>.  Branches are just convenient nicknames for commits, though, so I knew that you could also switch to any commit using the commit's ID.  I tried <code>git checkout 77cdf</code> and was surprised by the following message:

<pre><code>
Note: checking out '77cdf'.

You are in 'detached HEAD' state.  You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again.  Example:

  git checkout -b new_branch_name

HEAD is now at 77cdf...
</code></pre>

<h2>Branches and HEAD</h2>
Before breaking this down, let's review a few basics.  Git commits each have an ID number.  Git branches are convenient nicknames for commits.  A branch points at a single commit by remember its ID number.  Git also has a special pointer called HEAD which represents your current location in the potentially-large database of commits.

Almost all of the time, HEAD points at ("is attached to") some branch.  HEAD points at your branch, and the branch points at a commit ID, so when you ask git "Where am I?", git knows the commit you are at as well as its user-facing nickname.  <strong>When HEAD points at a branch that points at the current commit, and you make a new commit, HEAD and the branch follow you to the new commit.</strong>  Even when you're in a fresh repository and haven't made any branches yet, the default branch named 'master' fills this same role.

I understood that much before trying to <code>git checkout 77cdf</code>, but I expected no warning, and I expected that this would put me into a read-only state since no branches were involved.  I was so used to the idea that branches are <em>always involved</em> during the act of committing, and figured that without a branch, a new commit would be meaningless.

<h2>Breaking Down the Warning</h2>

<strong>You are in 'detached HEAD' state.</strong>  HEAD almost always points at a branch, which points at the current commit.  When you jump back in time to a commit that has no branch pointed at it, HEAD is no longer attached to a branch.  HEAD still lets you and git know where you are, but we no longer have an answer to the question "What user-facing nickname describes where I am?"

<strong>You can look around, make experimental changes and commit them...</strong>  This surprised me.  Without a branch name to follow me to the next commit, what would it even <em>mean</em> to commit?  Later on, I'd need a nice nickname for any new commits, so my repository viewer could know to show them to me at all, right?  If I commit from here right now, wouldn't I just forget the ID later on when I wanted to find it again?

Working with HEAD attached to a branch had influenced my mental image of what it means to make a commit; I was so used to the idea of a branch following you along that I mistook them for two parts of the same action.  In reality, creating a commit and having the branch name follow you are two distinct acts.  You can create a commit without having a branch follow you to the new state.

<strong>...and you can discard any commits you make in this state without impacting any branches by performing another checkout.</strong> AHA!  If I make several commits in this state, I <em>do</em> blaze a new nameless trail, and jumping away from that trail without first giving it a name will effectively discard my work (it's still there, but surely you'll forget the ID).  No harm, no foul.  Just an experiment.

<strong>If you want to create a new branch to retain commits you create, you may do so (now or later) by using -b with the checkout command again.</strong> Double-AHA!  If I start blazing a nameless trail as an experiment, and then I realize that I don't want to discard it after all, I can give it a name.  Interestingly, you can give it a name using the exact same command that you would use to create a feature branch off of master: <code>git checkout -b newbranchname</code>.  This makes sense.  When I'm on master during normal development and issue this command to create a branch, I'm really just saying "Oh by the way, the current commit is now ALSO known as newbranchname, and <em>that</em> name should start following me."

To sum up, detached HEAD state is a sharp tool.  Use it to inspect an old commit, or to perform an experiment using an old commit as a starting point.  Be sure to give that new work a name as soon as it stops being experimental.