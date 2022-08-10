---
title: 'Benchmarking Parsley\'s Span&lt;T&gt; Upgrade'
layout: post
---
Posts in this series:

1. [The Confidently Incorrect Mental Model](https://patrick.lioi.net/2022/08/08/the-confidently-incorrect-mental-model/)
2. [Detecting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/09/detecting-my-confidently-incorrect-span-t-model/)
3. [Correcting My Confidently Incorrect `Span<T>` Model](https://patrick.lioi.net/2022/08/10/correcting-my-confidently-incorrect-span-t-model/)
4. [Benchmarking Parsley's `Span<T>` Upgrade](https://patrick.lioi.net/2022/08/11/benchmarking-parsley-span-t-upgrade/)

In the last few posts, we've seen how to detect when we're operating on a Confidently Incorrect Mental Model, and how to pivot to trusted fundamentals and feedback systems until [flow](https://en.wikipedia.org/wiki/Flow_(psychology)) takes over to rebuild a sound mental model. Today we'll see the impact that kind of work can have over the course of a complex rearchitecting of a project.

Recall that the [Parsley text parsing library](https://github.com/plioi/parsley) lets you chop up text into meaningful pieces based on expected patterns, roughly like a regex, producing C# objects that are then easier to work with. We encounter that kind of work all the time. Newtonsoft.Json, for instance, needs to chop up JSON text into meaningful pieces and present them to you as C# objects. That library is specific to JSON, created bespoke for JSON text patterns, but the underlying concept is the same. Parsley, on the other hand, enables you to do that kind of text juggling work for *whatever* format you need to deal with.

Parsley is not unique. Text parsing is a well-defined space, standing on the shoulders of computer science's giants. Parsley uses a specific technique, recursive descent parsing using parsing combinators (a $20 phrase meaning "lots of tiny little lambda functions calling each other"). Even *that* technique has been well explored in the .NET community already. The existing libraries [Sprache](https://github.com/sprache/Sprache) and [Pidgin](https://github.com/benjamin-hodgson/Pidgin) dominate this space and with good reason.

So, Parsley is not unique. Why yet another parsing combinator thingy? My intention was not really to make a better tool in a space that's already so well taken care of. My goals were to learn about `Span<T>`, to improve my understanding of the metaphors used in the previous few articles (the "idea bubbling" we experience while coding, the "ladder of abstraction" we move up and down while coding, and the notion of "casting votes" for the next most appropriate thoughts).

Specific to the library, though, my goals included proving that `Span<T>` was not only a good fit for the job, but a great one. I wanted to demonstrate that you can get performance comparable to the other giants in the field without having to reach for other complex optimization techniques. Highly optimized code tends to be very hard to work with safely. I wanted to see if you could make a competitive tool in this space reaching only for the safe and relatively simple code that `Span<T>` provides.

The results were a bit shocking.

## The Benchmark

There are lies, damned lies, and software benchmarks. Let's spell out the benchmark scenario so you can decide if it's valid.

As a general-purpose parsing library, we'd like to prove that this works for more than what a typical regex would do. We want to parse a programming language or complex document format with these libraries. Since JSON is ubiquitous and well understood, it's a good target for our benchmark.

The Pidgin library is highly optimized and relies on the same fundamental computer science concepts as Parsley. If we can *remotely* compete with them using only `Span<T>` for optimization, that's a huge with for spans.

For context, though, it's good to also see what you can get with a bespoke JSON parser. We have two reference tools here. Newtonsoft.Json is an exceptional library with an incredible reach among .NET projects. It represents a great balance of speed and utility, while being constrained by backward compatibility goals. It's not the absolute fastest option, and it doesn't claim to be. It does, however, need to run fast enough to drive a ton of .NET projects in production. The newer System.Text.Json parser, however, was written specifically with `Span<T>` optimization in mind. Its entire purpose is to be a highly optimized bespoke JSON parser.

By running our benchmark scenarios against System.Text.Json, Newtonsoft.Json, Pidgin, and Parsley, we'll get a realistic view of where we really stand in the big picture. While Newtonsoft.Json and System.Text.Json provide real world contxt, Pidgin is our primary target of comparison going into this benchmarking. Prior to running the benchmarks, I figured we'd see System.Text.Json far out in the lead, followed by Newtonsoft.Json and Pidgin, with Parsley in an unambiguous last place. I figured that careful use of spans would help close the gap some, but that we'd still be in last place due to all of the serious performance work that has gone into the other tools. Spans could only do so much for a library that just stitches together countless recursive lambda expressions, right?

The benchmarks rely on [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet). This tool makes writing benchmarks about as easy as writing unit tests. It also takes into account things like deliberately warming up the scenarios' impact on the garbage collector, so we don't accidentally fool ourselves when comparing the data across scenarios.

For each of the 4 libraries in question, we parse the same few JSON documents. Sample documents include long arrays of many items, deep recursion of nested objects, and typical objects like you'd expect in a realist HTTP request.


## Parsley Before `Span<T>`

First, I ran these benchmarks with the older, pre-`Span<T>` implementation of Parsley against the 3 representative competitors:

```md
.NET SDK=6.0.203
  [Host]     : .NET 6.0.5 (6.0.522.21309), X64 RyuJIT
  DefaultJob : .NET 6.0.5 (6.0.522.21309), X64 RyuJIT


|           Method |     sample |         Mean |      Error |     StdDev | Ratio | RatioSD |     Gen 0 |    Gen 1 |    Gen 2 | Allocated |
|----------------- |----------- |-------------:|-----------:|-----------:|------:|--------:|----------:|---------:|---------:|----------:|
| System.Text.Json |  Recursion |    714.70 us |   0.548 us |   0.513 us |  1.00 |    0.00 |   42.9688 |  41.0156 |  41.0156 |    145 KB |
|  Newtonsoft.Json |  Recursion |    407.40 us |   1.306 us |   1.158 us |  0.57 |    0.00 |   37.5977 |  12.2070 |        - |    181 KB |
|          Parsley |  Recursion |  4,905.50 us |  96.465 us |  85.514 us |  6.86 |    0.12 |  625.0000 | 359.3750 |  78.1250 |  3,286 KB |
|           Pidgin |  Recursion |    542.59 us |   0.256 us |   0.214 us |  0.76 |    0.00 |   55.6641 |   1.9531 |        - |    229 KB |
|                  |            |              |            |            |       |         |           |          |          |           |
| System.Text.Json | Repetition |    336.81 us |   0.655 us |   0.613 us |  1.00 |    0.00 |   24.9023 |   5.8594 |        - |    103 KB |
|  Newtonsoft.Json | Repetition |  1,098.68 us |   1.651 us |   1.544 us |  3.26 |    0.01 |  103.5156 |  50.7813 |        - |    641 KB |
|          Parsley | Repetition | 17,533.33 us | 114.683 us | 107.274 us | 52.06 |    0.32 | 2031.2500 | 718.7500 | 250.0000 | 12,500 KB |
|           Pidgin | Repetition |  1,843.99 us |   4.215 us |   3.943 us |  5.47 |    0.02 |   56.6406 |  25.3906 |        - |    311 KB |
|                  |            |              |            |            |       |         |           |          |          |           |
| System.Text.Json |    Typical |     18.36 us |   0.026 us |   0.023 us |  1.00 |    0.00 |    1.3733 |        - |        - |      6 KB |
|  Newtonsoft.Json |    Typical |     53.99 us |   0.190 us |   0.178 us |  2.94 |    0.01 |    9.7656 |   1.5259 |        - |     40 KB |
|          Parsley |    Typical |    707.22 us |   1.427 us |   1.114 us | 38.53 |    0.09 |  167.9688 |   0.9766 |        - |    689 KB |
|           Pidgin |    Typical |     98.51 us |   0.488 us |   0.381 us |  5.37 |    0.02 |    5.0049 |        - |        - |     21 KB |

```

The first interesting thing is that our expected winner, System.Text.Json, was not always a definitive winner after all. Newtonsoft.Json was faster at handling deeply recursive objects, but needed slightly more memory. The Repetition scenario told a different story, with Newtonsoft.Json taking much more time and memory while System.Text.Json came out way ahead. In the typical scenario, as expected, System.Text.Json knocked it out of the park both in speed and memory allocated.

Alas, the old pre-`Span<T>` implementation of Parsley was just ridiculous here. It was never optimized, and was full of big, slow `string.Substring(...)` operations. The apparently simple case of parsing a long array took *52 times* longer to run than System.Text.Json, put tons more pressure on the garbage collector by keeping a lot of objects stuck in memory through to Gen 2, and had to allocate *121 times* more memory than System.Text.Json. Similarly, Parsley was in a distant fourth place in the typical scenario.

Parsley wasn't in the race at all yet. It wasn't even in the stadium!

As funny as that was in the moment, this offered a ray of hope. System.Text.Json was meant to be faster because of its use of spans. Perhaps applying spans to Parsley would at least bring us into the stadium.


## Parsley After `Span<T>`

Next, I ran these benchmarks with the newer, `Span<T>`-optimized implementation of Parsley against the same 3 representative competitors:

```md
.NET SDK=6.0.203
  [Host]     : .NET 6.0.5 (6.0.522.21309), X64 RyuJIT
  DefaultJob : .NET 6.0.5 (6.0.522.21309), X64 RyuJIT


|           Method |     sample |        Mean |    Error |   StdDev | Ratio |    Gen 0 |   Gen 1 |   Gen 2 | Allocated |
|----------------- |----------- |------------:|---------:|---------:|------:|---------:|--------:|--------:|----------:|
| System.Text.Json |  Recursion |   713.92 us | 0.624 us | 0.521 us |  1.00 |  42.9688 | 41.0156 | 41.0156 |    145 KB |
|  Newtonsoft.Json |  Recursion |   407.69 us | 1.231 us | 1.091 us |  0.57 |  37.5977 | 12.6953 |       - |    181 KB |
|          Parsley |  Recursion |   617.93 us | 0.493 us | 0.411 us |  0.87 |  31.2500 |  0.9766 |       - |    131 KB |
|           Pidgin |  Recursion |   542.82 us | 1.580 us | 1.320 us |  0.76 |  55.6641 |  1.9531 |       - |    229 KB |
|                  |            |             |          |          |       |          |         |         |           |
| System.Text.Json | Repetition |   329.27 us | 0.393 us | 0.348 us |  1.00 |  24.9023 |  5.8594 |       - |    103 KB |
|  Newtonsoft.Json | Repetition | 1,108.79 us | 3.247 us | 3.038 us |  3.37 | 103.5156 | 50.7813 |       - |    641 KB |
|          Parsley | Repetition |   665.34 us | 1.790 us | 1.674 us |  2.02 |  58.5938 | 29.2969 |       - |    326 KB |
|           Pidgin | Repetition | 1,836.44 us | 1.543 us | 1.205 us |  5.58 |  54.6875 | 25.3906 |       - |    311 KB |
|                  |            |             |          |          |       |          |         |         |           |
| System.Text.Json |    Typical |    18.65 us | 0.025 us | 0.024 us |  1.00 |   1.3733 |       - |       - |      6 KB |
|  Newtonsoft.Json |    Typical |    53.81 us | 0.068 us | 0.064 us |  2.88 |   9.7656 |  1.1597 |       - |     40 KB |
|          Parsley |    Typical |    37.88 us | 0.038 us | 0.032 us |  2.03 |   5.6763 |  0.1221 |       - |     23 KB |
|           Pidgin |    Typical |    99.73 us | 0.212 us | 0.199 us |  5.35 |   5.0049 |       - |       - |     20 KB |
```

Whoa! 

Deep recursion is still fastest in Newtonsoft.Json, but with lower memory requirements in Parsley. We're even using less memory than System.Text.Json.

The repetition scenario heavily favored the span-heavy tools System.Text.Json and Parsley.

Parsing typical documents is mostly a toss-up now, though System.Text.Json excels. This isn't surprising, as that's exactly its motivating use case.

Overall, applying spans alone not only brought Parsley into the stadium, but also made it highly competitive in this benchmark. As a result, the different tools can instead be evaluated for their ease of use, user-facing feature set, and communities rather than merely their performance characteristics. That's much more interesting, and can result in much better diversity of features, than having to choose a tool by performance alone.

## Conclusion

For Parsley, we can see that spans alone allowed us to arrive at highly competitive performance. This combined with the relative simplicity of the codebase means we've struck a nice balance between speed and maintainability. If this library has a bug, the simple implementation and meaningful test coverage will come in handy.

Let's zoom out to the bigger picture of this project upgrade and my mental model for `Span<T>`.

We see the importance of feedback systems throughout this process. First, the existing test coverage and compiler errors provided constant course-correcting feedback at each tiny step. Here at the end, the benchmarks similarly provided an important feedback system as I made the last few adjustments to the library. By that point I had rebuilt my mental model on a firm foundation, and I was able to refine that mental model using the performance testing as valuable feedback. Being able to repeatedly propose a possible adjustment to my use of spans and witness the results in both behavior and performance helped to flesh out my mental model.

The application of `Span<T>` was hitting a wall until I recognized the symptoms of the Confidently Incorrect Mental Model: honestly thinking that I understood, but then experiencing nothing but friction. After recognizing the symptom, I could see that if I didn't make a drastic change to my thinking, I could only continue to generate invalid ideas. After forcing myself to go back to fundamentals that I knew to be true and performing many tiny steps in that low-level high-confidence train of thought, I could finally build up a solid mental model and complete the upgrade successfully.
