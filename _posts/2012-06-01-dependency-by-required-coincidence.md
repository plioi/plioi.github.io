---
title: "Dependency by Required Coincidence"
layout: post
---


Software developers talk a lot about dependencies.  Dependencies are necessary, but each one carries with it some intellectual weight that can threaten a system's stability over time.  Your code is only as stable as its least-stable dependency, so we need to opt-into them with care.

Usually when we talk about dependencies in .NET, we limit ourselves to discussing the ones that are known statically, at compile time.  Assembly A depends on assembly B because when B is removed, A fails to compile.  Class A depends on Class B because when B is removed, A fails to compile.  The nice thing about such static dependencies is that you can find exactly what lines of code truly depend on the the assembly/class in question: delete it and see what turns all red and squiggly in Visual Studio.

Unfortunately, there's another form of dependency.  It's the dynamic and sneaky cousin of our familiar static variety: **Dependency by Required Coincidence**.

## A Gentlemen's Agreement

With a Dependency by Required Coincidence, you are writing code intended to be consumed dynamically in some very precise way by another system.  The agreement only works when you give in completely to the demands of the other system, when nearly every line of the class you control is written with full (or hopefully-full!) knowledge of how the other system will consume it.  When the dynamic consumer doesn't provide the behavior you need, it is therefore *your fault* and you are expected to bend your code around the problem.

Don't get me wrong: dynamic languages/platforms/etc are not inherently bad.  It's only when some dynamic consumer of your code forces a complex contract upon you that you are in serious trouble.

Does your code take part in one of these "gentlemen's agreements" with a system that is likely to change or be replaced?  If so, you've got yourself a required coincidence problem.  Your code must contain properties, methods, and types that are unused by the rest of your code but exist because you just happen to know exactly *how* the other system will require them to exist at runtime.

## MVVM in WPF

I see this most often when working with the MVVM pattern in WPF applications.  *(I don't have experience with the MVVM pattern(s) outside of WPF, so for the rest of this post whenever I say "MVVM", assume I mean "MVVM in WPF")*

> MVVM was designed to make use of data binding functions in WPF to better facilitate the separation of view layer development from the rest of the pattern by removing virtually all GUI code (“code-behind”) from the view layer. (<a href="http://en.wikipedia.org/wiki/Model_View_ViewModel">source</a>)


This approach seems to have resulted from the following train of thought:

1. I remember having to work with unmainainable code-behinds when I worked with WinForms.  Therefore, code-behinds must be avoided.
2. The defining qualities of code-behinds are that they are code files which can directly access control state and can directly receive controls' events via public methods.
3. From (1) and (2), accessing control state and receiving events from controls must therefore be bad too.
4. Somebody else had better access control state and receive events for me: a data binding system.  Instead of a code-behind, I'll provide a code-behi-... er... a *view model*.  This will be a class whose properties will describe the data to be displayed, and I'll enter into a Required Coincidence Contract with the binding system so that my code will be properly manipulated by the binding system at runtime.


I find the premise here faulty, so maybe I've set up a straw-man argument, but I'm unaware of motivations for MVVM aside from an aversion to code-behind hell.  Most of us have had to work with painfully-unmaintainable code-behinds, but I hold that these pains weren't due to the fact that the code-behind classes access control state and receive events.  Rather, the problems came about when the code-behinds did just about anything *else*.

This leaves me thinking that we've entered into this elaborate contract with the WPF data binding system just so that we can throw the baby out with the bathwater.  UI controls are inherently stateful/eventy things, and we've opted for a Required Coincidence just to remove stateful/eventy code.  Two examples and their workarounds come to mind:

**First, my view model is not allowed to receive events.**  If an item is removed from a list box control bound to one of my collections, WPF will mess with my collection for me.  That's all well and good, but when my code just plain needs to react to some interesting event, it may not be covered by the finite list of things that WPF data binding is written to *do* to my view model.  Furthermore, the binding system admits the importance of "eventy" code by requiring me to publish view model state change events so the data binder can receive them!

As a workaround, you can use something like <a href="http://caliburnmicro.codeplex.com/">Caliburn.Micro</a>.  Caliburn.Micro is a third-party framework which builds on MVVM so that you can, among other things, set up a control's events to trigger calls against public methods on your view model.  An event handler!

**Second, my view model is not allowed to access control state.**  UI controls can be pretty intricate.  Even a lowly text box control has to deal with a lot of possibilities.  By default it will need to handle different kinds of keystrokes, some contributing to the text box content, some highlighting text, some performing other tasks such as cut/copy/paste, or tabbing to another control, etc.  When your application needs to customize the details of a control's behavior, you just plain have to have access to its low-level state and events.

As a workaround, you can use an <a href="http://www.codeproject.com/Articles/28959/Introduction-to-Attached-Behaviors-in-WPF">attached behavior</a>.  These are classes that are suspiciously/exactly like code-behinds!

With MVVM, WPF isn't the Windows Presentation Foundation.  It's the Workaround Proliferation Framework.

By surrendering control to the WPF data binding system, we set ourselves up for a nasty surprise the day that WPF falls by the wayside: our view model classes and attached behavior classes will be of absolutely no use to WPF's replacement, but they will have absorbed a scary amount of application logic along the way that will have to be rewritten!

## Instability by Required Coincidence

To sum up, systems like WPF's data binding make you enter a Dependency by Required Coincidence.  You must create a great deal of code with hopefully-full knowledge of how the other system will consume it.  Your code is shaped by the thing it indirectly depends on, and may become useless when that unstable dependency is inevitably replaced.
