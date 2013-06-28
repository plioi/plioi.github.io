---
title: The Boiling Backlog
layout: post
---

Software development processes tend to be too prescriptive, leading to waste. For instance, most Agile training prescribes fixed-sized iterations ending with a retrospective meeting. Blindly following this structure may waste time: either you hold an expensive meeting when there isn't enough to discuss, or you motivate the team to hold back ideas for improvement until the next meeting. By taking retrospectives out of the process, you may instead enable the team to make improvements constantly.

The only prescriptive advice I can give is to **ruthlessly remove waste from your process**. What remains ought to communicate useful information to the team and its stakeholders. Anything more than that is just software development *theater*. Approach your development process the same way you approach a bit of ugly code: refactor away anything redundant or overly-complex.

Each project's process is going to vary in response to the project's constraints. The process I've been using on [my side project](https://github.com/plioi/fixie) is especially low-tech. It is only ideal for this particular project. If I blindly applied it to some other project, I'd be falling into the same trap as everyone who ever sold a prescriptive Agile ScrumMaster certificate. On this project, my constraints are:

<ol>
<li>The team is very small (1 person).</li>
<li>The overall vision is well known. Even with little planning, I know where I'm heading.</li>
<li>The high risk requirements were vetted early on with a [proof of concept](http://www.headspring.com/patrick/strongly-typed-whiteboarding/).</li></ol>

Fitting these constraints, the ["simple as possible, but no simpler"](http://c2.com/cgi/wiki?EinsteinPrinciple) process that has served me well the last 2 months is a specific variation on Kanban. My Kanban board has four swim lanes: Backlog, Doing, Publish, and Done:

<a href="http://www.headspring.com/wp-content/uploads/2013/05/kanban.png"><img src="http://www.headspring.com/wp-content/uploads/2013/05/kanban.png" alt="Fixie&\#039;s Kanban has four lanes: Backlog, Doing, Publish, and Done" width="363" height="271" class="aligncenter size-full wp-image-6525" /></a>

Since this is a one-person project, tools like JIRA, PivotalTracker, or Trello are overkill. Sticky notes will always be faster to work with than issue tracking software, as long as all (1) team members have access to the board. I often dramatically reorder the backlog in a few seconds to match my current plans, which would be tedious with a mouse.

Most of the notes are simply the name of a feature. I don't bother forcing them into the Agile template "As a &lt;type of user&gt;, I want &lt;some goal&gt; so that &lt;some benefit&gt;." If I did, a note for feature X would always expand into "As a Patrick, I want X so that I can have X." I wouldn't gain any new insight or communication from that exercise.

I limit the Doing lane to have one task at a time, because anything else would be a lie. I can only do one thing at a time.

Some tasks deserve special treatment. Documenting my progress here is as important as making progress in the first place, so I gave the blog writing tasks their own Publish lane. Like Doing, this only ever has one incomplete task.  Unlike Doing, Publish tasks can stack up. If I've written ahead 2 or 3 articles, they pile up here as a reminder of the order I wish to publish them.

I've been picturing the Backlog lane as a pot of boiling water. At the start of the project, I had identified 3 "Epic" features encompassing the whole project. The first of these Epics started to split apart into a few concrete tasks and a few medium-sized wish list tasks. At any time, I could pull one small, concrete task over to Doing. As I discover more about what I need, vague tasks split into smaller concrete tasks and "bubble up" to the top of the lane. The more I learn about a task, the higher it floats up the Backlog. Now that I have reached the end of the first Epic, the second one is naturally splitting into several medium tasks and a few specific tasks have made their way to the surface.

<blockquote>The boiling/bubbling action in the Backlog has helped in two unexpected ways. First, I'm never at a loss for what to do next because there is always at least one manageable task to claim from the top. Second, because only so many tasks can fit in the lane, the board naturally resists my attempts to plan too much in advance.</blockquote>

I've deliberately let the Done lane become overstuffed with completed tasks. I didn't empty it out until the first of the 3 Epics was complete. There's some kind of psychological trick about seeing your successes pile up. If I just discarded them as soon as they were done, I'd probably feel less motivated.

I don't have iterations, as they seem to artificially slow things down. I'd rather be in a constant state of pulling the next task.

A process this small won't work for everyone, but should serve as an example of just how low-tech and simple you can get. It gives me the information I need while putting zero obstacles in my path. I get to focus on one thing at a time, and I let the board tell me when it's time to plan ahead.

What does your current process look like? How is it serving your project's constraints, and what obstacles is it putting in your way?