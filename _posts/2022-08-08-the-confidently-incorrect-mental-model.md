---
title: 'The Confidently Incorrect Mental Model'
layout: post
---
Posts in this series:

1. [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/)
2. [Detecting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/)
3. [Correcting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/10/correcting-my-confidently-incorrect-span-t-model/)
4. [Benchmarking Parsley's `Span<T>` Upgrade](https://patrick.lioi.net/2022/08/11/benchmarking-parsley-span-t-upgrade/)

There is a class of software development challenge I'll call the Confidently Incorrect Mental Model. You think you do have a good mental model, a useful metaphor or a useful picture in your head, for what some technical concept is and how to work with it. You have a strong sense that you know what problems it is a good match for, what the solution will look like, and what you would expect to be valid or invalid usage of the thing. The moment you try to actually use it, though, you feel nothing but friction.

I've felt this friction frequently throughout my career. I've had this challenge with most front-end technologies from WPF to CSS, libraries like Entity Framework, and tools like git. These all require an accurate mental model to work with them, and so they naturally pose a risk of the Confidently Incorrect Mental Model, especially when first working with them.

You can run into the Confidently Incorrect Mental Model on just about any technical topic. You might read about a new library, or a neat design pattern, or a programming language. You think you've got the right idea, but your first attempts are nothing but pain. Anything that depends on your forming a mental picture or metaphor for what you're manipulating is a candidate for this challenge.

This experience is incredibly frustrating. You have an incorrect mental model, one that feels nearly complete and internally consistent. The mental model is in fact wrong, so every individual piece of evidence immediately and strongly conflicts with the model. You think the model is true, and here is constant proof that it is false. It's hard to hold that kind of contradiction in your head, so it hurts and is frustrating.

Because most technical concepts rely on a solid mental model, and because you could not possibly have a complete mental model when you're just starting out, it's incredibly common to face this challenge again and again. People early in their career could quite easily misdiagnose the problem, incorrectly declaring themselves not cut out for this stuff: "This job seems supernaturally hard at each step, so I must be bad at it."

Thankfully, there is hope! The symptoms are easy to detect, and there is a simple solution, albeit one that's easy to forget about in the frustrating heat of the moment.

## Symptoms

You can detect that you're in this situation by recognizing its two symptoms:

First, you do honestly feel like you get it. You have a picture in your head. That picture seems to match up with what you're reading about it in articles and documentation. That picture matches up with real world examples. That picture suggests what actions you should take right away to use it effectively and suggests that you'll be able to spot good and bad usages easily. Sure, you know you'll probably bump into a problem here or there along the way, but you know what success is going to look like, and you might even feel confident enough to sell someone else on the idea by echoing the elevator pitch you've received from experts.

Second, the moment your hands meet the keyboard, all you feel is friction. You're unsure what specific line of code comes next. It doesn't behave like you expected. You get confusing and surprising error messages at build time or at run time. Online guidance in response to those errors seems confusing or outright wrong. Overall, it's just a completely painful experience even though you expected it to be a smooth experience.

## Getting Lost in Thought

With those two symptoms, you are probably holding a Confidently Incorrect Mental Model. This moment of realization is half the battle. Without noticing the symptoms, your most natural reaction to the pain is very likely to be the exact opposite of what you need to do. Without noticing the symptoms, you're likely to just keep trying different variations on what you've been trying, rather than addressing the incorrect model itself. Instead, to address the incorrect model, you need to abruptly interrupt your most natural reaction.

> Your most natural reaction to this friction is to *think*, and you're going to have to interrupt that bad habit.

If you close your eyes and try to empty your thoughts, you'll find that they keep happening anyway. Thoughts just kind of bubble up from the deep, and the best you can do is choose to pay attention to them or discard them.

We have little to no agency over *what* thoughts bubble up, but the more thoughts we have about something, the more likely we are to bubble up related thoughts. You might insist that you have great agency here instead and attempt to prove it by deliberately listing cities to yourself one after another. At first, you might have some random thoughts mixed in, but after listing a few cities, the most likely next thought to bubble up is surely another city. Each thought casts a vote for what our next thought will be, and our next thought is more and more likely to align with those votes. "Dallas, I'm worried about tomorrow's presentation, New York, what should I have for lunch, Paris, Chicago, Houston, Boston..." Now, don't think about elephants. "Los Angeles, Elephants... dang it."

We do seem to have some agency over what to *do* with those thoughts as this thought-bubbling action goes on: after failing to suppress thinking about elephants, you can forcefully direct your attention back towards something else, at least for a short time. The elephants vote has been cast, though, and it'll take a fair number of new city votes before you really set aside the elephants for good.

We feel like we have great agency in our thought processes, but upon inspection it seems we're mostly just going along for the ride, with some slight course-correcting. We tell ourselves a story after-the-fact that we had some intentional train of thought arriving at some great solution.

So, thoughts seem to bubble up from nowhere, they seem to have this recency bias, and we seem to have some slight control not in their *generation* but in their *evaluation* and in the *casting of the next few votes*. We can spot a thought and either reject it by redirecting our attention, or else we can lean into it, casting another vote for thoughts *like that one*.

This process is happening as we tie our shoes, as we list cities, and as we work through complicated technical problems. The only difference is how fantastical the story is that we tell ourselves after-the-fact.

## The Antidote

What does this have to do with the Confidently Incorrect Mental Model? If you have such a model and are experiencing pain, that means you've been generating thoughts that are simply not useful. The model you have is the one at hand when thoughts bubble up trying to meet that model. "Try a line of code like this." "Google that error message and try the first confusing suggestion." "Call this other overload of the method that isn't working." Our model is wrong, so it started bubbling up ideas that do not align with the reality, but the series of thoughts have all cast confident votes to bubble up yet more similar thoughts. You've chased yourself into a mental cul-de-sac, and as long as you keep thinking, you're only going to run around that cul-de-sac in circles.

After some practice being self-aware of your own bubbling thoughts and after thinking about this post enough times, *those* votes will leave you more likely to bubble up the thought "Oh! This is the thing! This is the Confidently Incorrect Mental Model!" **The jarring and counterintuitive next move, then, is to forcefully wrench yourself out of that series of like thoughts.** You have to drag yourself, kicking and screaming, out of that cul-de-sac **by doing something that will force several new votes of a different kind**, thoughts that will help you to instead build up the correct mental model.

You need to set the *apparent* immediate task aside and climb down the ladder of mental abstraction to the absolute rock bottom basics, and take several smaller steps.


## Drawing Geometry in WPF, with a Forms-Over-Data Mental Model

For instance, I once needed to enable a user of a WPF application to mark up an image by drawing polygons on top of it. The user would click 4 corners one after another, with some visual feedback as the shape took... um... shape.

I had a basic understanding of WPF at that point, but less around drawing geometry and more around text boxes and buttons. My mental model of what WPF offered was therefore limited, and my intuitions were all wrong about how to approach the challenge. Searching around for other people's solutions to similar problems gave me some code to work with, but every adjustment I tried to make to that sample code was painful. Nothing behaved like I wanted it to. Pages of sample code seemed like a great starting point, but my mental picture of what that code was doing was simply wrong, and so each next idea I had was both wrong and very similar to the last wrong idea.

I was chasing myself into that cul-de-sac, and all I knew was that I was burning up the budget of this project trying to set up the UI interaction that would underpin months of subsequent work for a team whose size was about to ramp up quickly. Panic!

The solution here was to catch myself in the act, to first notice the 2 symptoms (confidence in model + friction at every step) and then pivot using the antidote (ruthlessly set it all aside and fall back to familiar trivial steps).

In this case, the trivial steps were to set aside the well-meaning third-party sample code, and draw a single line between two hard-coded coordinates. Commit. Receive a mouse click within the picture and log the coordinates. Commit. Receive 2 mouse clicks and collect both their coordinates. Commit. Draw a line between them. Commit. At this point, I was seeking more targeted guidance online for each tiny piece, increasing the chance of getting quick positive feedback. The incredibly small commits were forcing me to do one thing at a time.

> The *sequence* of like-minded tiny and valid steps involved a corresponding sequence of tiny and valid thoughts, each casting votes towards the next thoughts.

My *reset mental model* was starting to take shape and formed the context for the next vote-encouraged thoughts. Eventually the old invalid model was replaced with an effective model. I could cleanly implement the 4-sided polygon drawing needed for the app, and could clearly guide a teammate to implement similar features for a few other kinds of geometry needed by the app.


## Git, with a Subversion Mental Model

I'm comfortable using git now, but I had a great deal of difficulty learning it. My prior experience was with Subversion, and even that experience involved very little work with branches. I had a mental model that included source control being an ordered series of *diffs*, with branches being "a glorified giant folder copy to a subfolder". It was a weak model, *and one completely out of alignment with what git is really doing*.

After many frustrating experiences of getting into trouble with git, asking for help to fix things, and walking away from those conversations even more confused, I needed to wrench myself away from the invalid mental model, go back to absolute basics, and build up a new mental model from scratch.

I found articles and videos where git experts really showed what's going on inside the *.git* folder when you execute some branch related commands, which shook me loose from my old Subversion model of what a branch *is*.

I limited myself to a very small subset of commands, things I knew I needed to use every day and that generally worked as expected.

I switched to a git history viewer with an explicit refresh button rather than an automated refresh after each action. I could look at the viewer, predict the effect of the next command, type the command character by character, issue the command, refresh explicitly, and immediately get feedback:

- "Yup, just like I expected! On to the next tiny step..." **or**
- "Stop the presses! Detect what it did vs what I expected right now before I move anything else!"

In other words, I forced myself away from the series of bubbled-up thoughts for complex commands that were based on my invalid Subversion model, favoring instead a series of much smaller, much simpler thoughts. I could get real feedback on each of those tiny thoughts as I built up and adjusted the better mental model from scratch, casting votes towards similar thoughts of valid commands and successful results, until I could finally climb back up the ladder of abstraction to handle more complex concepts like merges, rebases, and confidently *correct* branch juggling. Suddenly *I* was the one other people came to when *they* accidentally force-pushed against `main` on Release Day.


## The Confidently *Correct* Mental Model

We have a powerful pattern of thought that is usually a good match for problem solving, but one that can get stuck in a dead end, and when we detect that we are stuck in that dead end we can wrench ourselves out of it by forcefully going back to basics and taking incredibly small steps.

We have a powerful, naturally ability to bubble up new ideas, evaluate them, and cast votes for the ideas that will bubble up next. This ability lets us accomplish great things.

It can also get us stuck in a mental dead end when the sequence of thoughts is built atop an incorrect mental model. The sequence of unhelpful thoughts yields more unhelpful thoughts.

By forcefully setting aside the chain of thoughts and by instead taking a sequence of much smaller and much more fundamental steps, we can start a new chain of thoughts that *are* valid, resulting in a better mental model, resulting in more useful thought generation to solve the problem at hand.

In the [next post](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/), we'll see how this problem cropped up while I was trying to learn about C#'s `Span<T>`.