---
title: 'Tiny Steps: Creating Fixie 2.0'
layout: post
---
With the recent release of .NET Core, it&#8217;s time to upgrade the [Fixie test framework](https://fixie.github.io/). Fixie needs to support the new project structure, tooling, and cross-platform behavior introduced by .NET Core: not only should developers of this project benefit from all the new things, but more importantly end users should also be able to use Fixie to test their own .NET Core projects, and even do so while developing them outside of Windows. That&#8217;s no small feat. Right now, I&#8217;m about halfway through that effort, which you can track on [Fixie&#8217;s GitHub issue #145](https://github.com/fixie/fixie/issues/145). For the bigger picture, see the new [Roadmap](https://github.com/fixie/fixie/wiki).

Recently, I gave a talk about .NET Core and the challenges it poses for test frameworks like this one.

&#8212; Well, that&#8217;s a lie. &#8212;

It&#8217;s not about that at all. It&#8217;s _really_ about how to recognize a big family of architectural problems in software projects, and how to go about solving those problems in tiny steps:

<iframe src="https://player.vimeo.com/video/495046446?title=0&byline=0&portrait=0" width="640" height="360" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe>
