---
title: "Migrating from Bitbucket to GitHub"
layout: post
---


Git and Mercurial certainly have their differences, but in the grand scheme of things they are both equally-viable source control tools.  Bitbucket and GitHub are also both equally-viable hosts for Mercurial and Git projects.

All other things being equal, when it comes to open-source software the Capricious and Arbitrary Popularity Contest kicks in with the deciding vote.  Git, and in particular GitHub, has won the popularity contest in recent years and will own the title of Open Source Thingamabob for the near future.  Something cooler will come along and take its place, but for now GitHub is the place to be.

We still trust Bitbucket with our client work, but we're in the process of migrating our open-source projects to the new <a href="https://github.com/HeadspringLabs">GitHub "HeadspringLabs" organization</a>.

After a few failed attempts and much gnashing of teeth, I successfully migrated the <a href="http://patrick.lioi.net/2011/11/18/building-rich-enums/">Enumeration</a> project from Bitbucket to GitHub, complete with history.


## Installing Hg-Git on Windows

If you're migrating a Subversion project to GitHub, they already provide an simple import service.  They don't yet offer a tool for importing Mercurial repositories, though, so I found a Mercurial-to-Git bridge called <a href="http://hg-git.github.com/">Hg-Git</a>.

Hg-Git is intended to let you work continuously against a Git repository while using Mercurial tools on the client side.  I didn't need all that, but it can also be used to do the initial push to GitHub from within a Mercurial repository.

The documentation on the Hg-Git's main site is a little brief, and after Googling around I found some good advice as well as some good advice that no longer applied.  To get Hg-Git installed on Windows, here's all you really need to do:

1. Install a recent TortoiseHg if you haven't already.  I used TortoiseHg 2.3 (Mercurial 2.1).

2. Open a command window and `cd` to a development folder.  We'll be creating some repository folders underneath it:

    ```
    $ cd C:dev
    ```

3. You install Hg-Git by simply cloning its own repository off of Bitbucket. This creates a new folder named hg-git under the current directory, and underneath that you'll see a folder named hggit:

    ```
    $ hg clone https://bitbucket.org/durin42/hg-git
    ```

4. Mercurial needs to be told that this extension exists.  Update `C:\Users\<username>\mercurial.ini` and add the following (using whatever path you cloned to):

    ```
    [extensions]
    bookmarks =
    hggit = C:devhg-githggit
    ```

5. Confirm that Mercurial knows what hggit is:

    ```
    $ hg help hggit
    ```


## Initial Failure

You'll need to have a new empty repository created on GitHub, and you'll need to have the rights to push to it.  You'll need to have a local clone of the Bitbucket project you wish to migrate.

If you follow the advice on <a href="http://hg-git.github.com">the Hg-Git site</a>, you'll try to issue a special "`hg push`" command using a git-specific destination URL.  **This can push the history of commits only partially.  The GitHub repository will unforunately fail to show *who* performed each commit.**  Yikes.  When I got this far, I had to delete the GitHub repo and start over.

The problem is Mercurial uses one string to represent committer identity while Git uses two (name and email).  Additionally, GitHub pays special attention to the email part, allowing the site to recognize existing GitHub users and linking to their user page.  Since Hg-Git wasn't populating email addresses, GitHub had little to go on.

## Successfuly Migrating from Bitbucket to GitHub

**Instead** of using Hg-Git to perform the whole push, we'll follow some <a href="http://www.theleagueofpaul.com/hg-to-git">excellent Hg-Git advice</a> that lets us step in to fix committer identities along the way.

That site has a shell script (which can be executed on Windows using the Git Bash command prompt).  The script will use Hg-Git for only one small part of the process: converting a local Mercurial repository folder into an equivalent local git repository.  The only thing wrong with that git repository will be the user names and email addresses.  The script then takes a chainsaw to that local git history, fixing the identity of each committer.  Finally, the script uses regular git commands to do the final push to GitHub.

The downside is that you have to modify this script with a separate chunk for each real-world person that appears in the Mercurial history.  For instance, let's say we have history in Bitbucket for users named hgjohn and hgjane.  John Doe has a GitHub account already and prefers to be known there as "John Doe" with email address "john@example.com".  Jane Doe doesn't have a GitHub account yet, but later on if she does she'll want to be known as "Jane Doe" and "jane@example.com".  With these transformations ready, we modify the script to the following:

```
#!/bin/sh

#Usage:
# ./migrate_hg_to_git LocalRepoFolderName HgSourceRepoUrl GitDestinationUrl
#
# For example:
# ./migrate_hg_to_git MyProject https://bitbucket.org/Example/myproject git@github.com:Example/MyProject.git
#
# The git destination URL should be a new, empty repository for which you have push rights.
#
# Note that the filter-branch 'quoted' script must be updated to translate specific users found in your HG project.

hg clone $2 $1
cd $1
hg bookmark -r default master
hg gexport
mv .hg/git .git
rm -rf .hg
git init
git checkout master -f
git filter-branch --env-filter '

an="$GIT_AUTHOR_NAME"
am="$GIT_AUTHOR_EMAIL"
cn="$GIT_COMMITTER_NAME"
cm="$GIT_COMMITTER_EMAIL"

if [ "$GIT_AUTHOR_NAME" = "hgjohn" ]
then
    cn="John Doe"
    cm="john@example.com"
    an="John Doe"
    am="john@example.com"
fi
if [ "$GIT_AUTHOR_NAME" = "hgjane" ]
then
    cn="Jane Doe"
    cm="jane@example.com"
    an="Jane Doe"
    am="jane@example.com"
fi

export GIT_AUTHOR_NAME="$an"
export GIT_AUTHOR_EMAIL="$am"
export GIT_COMMITTER_NAME="$cn"
export GIT_COMMITTER_EMAIL="$cm"
'

git remote add origin $3
git push origin master
```

After saving this as "migrate_hg_to_git.sh" in our development folder, we execute the following command (all on one line).  This creates a local folder named 'MyProject', populates it with a Bitbucket clone, converts it to a Git repository, fixes the names, and pushes to GitHub:

$ `./migrate_hg_to_git.sh MyProject https://bitbucket.org/johnd/MyProject git@github.com:jdoe/MyProject.git`

Now when you view the project on GitHub, John's commits display his photo and link to his user page.
