---
title: 'Detecting My Confidently Incorrect `Span<T>` Model'
layout: post
---
Posts in this series:

1. [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/)
2. [Detecting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/)
3. Correcting My Confidently Incorrect `Span<T>` Model
4. Benchmarking Parsley's `Span<T>` Upgrade


As I covered in [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/), when developing software we frequently run into the problem of using an invalid mental picture or metaphor for some technical concept we're working with. It can lead us into a conceptual dead end, where unhelpful ideas favor yet more unhelpful ideas, none of which align with real world feedback.

This jarring mismatch between our ideas and reality is intensely frustrating. If we're not mindful of the symptoms, we're very likely to just habitually produce more and more similar but still invalid ideas, never getting unstuck. If we *are* mindful of the symptoms, though, we can detect the problem, recognize the invalid model as the root cause, take deliberate action to shake thoughts loose from the grip of momentum, reject the model, and rebuild it more carefully from scratch in small feedback-driven steps.

I encountered this problem on a massive scale over several weeks, while trying to apply the .NET concept of `Span<T>` to an old side project, the [Parsley text parsing library](https://github.com/plioi/parsley). I caught myself in the act of entertaining a Confidently Incorrect Mental Model by detecting the symptoms covered in the previous post and pivoted by wrenching myself out of that train of thought, forcing myself to take a series of smaller and simpler steps. That reset my train of thought to a much more fruitful sequence of tiny ideas, ultimately rebuilding my mental model of spans and unblocking the project.

Today, we'll cover the context of the work that got stuck, and what led me to realize that I had a Confidently Incorrect Mental Model. In the next post, we'll see what it took to build up a better model.


## `Span<T>`

The `Span<T>` type arrived in combination with the C# `ref struct` feature, which most developers will rarely reach for directly. `ref struct` and `Span<T>` may show up in your project here or there, but for the most part these are fundamental building blocks used *within* libraries and frameworks, and they explain some of the impressive performance gains in ASP.NET. A typical ASP.NET controller action receiving a JSON payload mapped to a domain model type, for instance, *benefits* greatly from ASP.NET's internal use of `Span<T>`, but doesn't require the controller's developer to even be aware of it.

Although `ref struct` and `Span<T>` directly deal in efficient use of memory, their main "elevator pitch" use case comes in how they offer dramatic improvements over the familiar `string.Substring(...)` operation. C# is garbage collected, and its `string` type is immutable. Both of those things are generally good, and generally fast enough to be worthwhile when weighing pros and cons of performance, correctness, and ease of use. Still, if you know that you are going to be making countless substring operations deep within a library or framework, those thousand cuts add up.

When ASP.NET needs to parse an incoming web request, it needs to chop up the incoming HTTP request into many meaningful pieces. It needs to make sense of all the incoming HTTP header lines, and in the case of a JSON payload in the body it needs to similarly chop that text blob up into many meaningful bits of punctuation, numbers, strings, and bools. A realistic HTTP request could easily get up into the hundreds if not thousands of substring operations, in a simple implementation. Multiply that by all of your incoming HTTP requests, and then multiply *that* across all of Azure, and you might just start to react to the old adage "Premature optimization is the root of all evil" with an informed "Sure, but now is the time, and time is money!"

In short, though, `Span<T>`, and more specifically `Span<char>`, simply *describes* a substring operation by referencing the starting character of some original chunk of text and noting how long you wish that substring would be. That's roughly equivalent to remembering two numbers. Compare that to the work performed by a true `string.Substring(...)` operation: determine the length of the substring, allocate whole new memory to store the characters, copy the characters one by one from the original string into the destination, with all of this working "on the heap" in need of garbage collector memory management. Remembering two numbers is far less work than all that. Additionally, as a `ref struct`, the compiler goes through great lengths to ensure that the `Span<char>` itself, the glorified pair of numbers, will never live "on the heap" and instead live "on the stack"; this happens to dramatically reduce the cost of setting aside room for it and subsequently cleaning it away when you're done with it. The compiler does a lot of heavy lifting here so that the finer points of the performance trickery happen behind the scenes.

The details aren't super important for this discussion of the Confidently Incorrect Mental Model and how to resolve that kind of challenge. All we need to know from here on out about these `Span<T>` details is:

1. They're great for replacing many big slow substring operations.
2. They come with a lot of intentionally strict, atypical compiler rules.


## Misplaced Confidence

When `Span<T>` first came out, I read a lot of promotional material and developer blogs about it. It seemed neat. The use case was compelling. Its promising performance impact was great evidence that the whole .NET Framework to .NET Core transition was worth it. Everything I read lined up nicely with my accurate and useful mental model for regular `struct`s in contrast to `ref struct`s, which lined up nicely with my accurate and useful mental model of .NET value types vs reference types. I gave an internal tech talk at work about it and included it in a public webinar for the same employer when motivating .NET Core migration projects. I could give anyone the elevator pitch off the top of my head. I could talk the talk. I didn't even feel like I was stretching the truth. Not to pat myself on the back, but I could draw *all* the boxes and arrows. This topic was in my wheelhouse and lined up nicely with my past experiences. Heck, I probably could have passed a job interview for a position that would deal entirely with `Span<T>` code.

> Me: I've got this picture in my head and everything! Of course I know spans!
>
> Narrator: He didn't know spans.


## The Demo Project: Parsley

Years ago on the original .NET Framework, I made a little library named [Parsley](https://github.com/plioi/parsley). Parsley is a text parsing library, letting you convert potentially-complex text into more meaningful C# objects. For instance, you could use it to transform a string containing a JSON object into an equivalent `Dictionary<string, object>`. Think "regexes, but more reader-friendly, and capable of recognizing more complex patterns".

After reading everything about `Span<T>` and its use under the hood in ASP.NET Core and `System.Text.Json`, it seemed like a perfect match. I could get my hands-on experience with `Span<T>` at the same time as revitalizing this old project.

First, just to divide-and-conquer, I upgraded the old solution in-place to .NET 6 before even thinking about applying `Span<T>`. I figured I'd be touching a lot of the code just to apply `Span<T>` and didn't want to complicate matters by mixing together two potentially big changes.

> After that, I started up a new branch, cracked my knuckles, **and immediately face planted over and over again**. For the life of me, I could not get anything involving `Span<T>` to work.

In hindsight, my past comfort with regular `struct`s, value types, and reference types made it too easy to read the docs on `Span<T>` and conclude that I had a correct mental model. I had a great deal of confidence, but that confidence was misplaced. The full picture, the one the compiler needs to enforce, was a gap in my mental model that I didn't realize was missing. My incomplete mental model in combination with this misplaced confidence had a direct impact on the code I tried to write and my constant frustration with the feedback I was getting from the compiler. I too often thought that some code change would be valid, but it wasn't. It was so bad that I could rarely write a single line of `Span<T>` code that *would* compile! Most of the early experience was a matter of typing a few characters worth of code only to find the compiler slapping me in the face: "You can't do *that*, buddy!"

It wasn't enough to just read the docs and memorize some list of things you're allowed to do. Sometimes that's enough, but here the full picture was too big, impactful, and subtle to get away with merely pattern-matching on the examples.

It got to the point where I honestly started thinking that `Span<T>` was such a subtle niche concept that it would only ever be used inside the .NET framework and ASP.NET Core, and only used indirectly by folks like me.

## Detecting the Symptoms

Eventually I made *some* headway but found myself getting stuck in cyclical thoughts. I'd have an idea, some high level architectural move that would get spans into the mix, and get slapped immediately by the compiler. Ok, what about *this*...? SLAP. Fine, what about *this*...? SLAP. What about... what about... what about... SLAP SLAP SLAP. My ideas kept being kind of similar to each other, slight variations on a theme, while each received the same feedback: "You can't do *that*, buddy!"

Ok wait. That sounds exactly like the "dead end thought bubbling" trap outlined in [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/). My train of thought was way *up* the ladder of abstraction and built atop my faulty mental model, so naturally each thought I had about what to do next was incorrect and more importantly *casting votes* in favor of more thoughts comparable to the ones before.

At this point, I realized the familiar sensation of the Confidently Incorrect Mental Model. I had both of its symptoms:

1. I honestly felt like I had a good mental model.
2. The *moment* my hands met the keyboard, all I felt was friction.

That conflict in the two symptoms was jarring because my own thoughts were clearly mismatched with reality. Thankfully, my thoughts while coding are so often mismatched with reality that I could recognize the feeling and make an appropriate pivot. I was stuck, and *thinking* was making it *worse*. In the next post, we'll see how I applied the antidote.
