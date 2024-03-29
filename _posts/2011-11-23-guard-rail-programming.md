---
title: "Guard Rail Programming"
layout: post
---


Rich Hickey, creator of the Clojure programming language, recently gave a talk titled <a href="http://www.infoq.com/presentations/Simple-Made-Easy">Simple Made Easy</a>. About 15 minutes in, Hickey expresses some strong opinions on Test-Driven Development that initially struck me as blasphemy: *"I think we're in this world I'd like to call **Guard Rail Programming**... 'I can make change because I have tests!' Who does that? Who drives their car around, banging against the guard rails? Do the guard rails help you get to where you want to go?"*

Yikes. I'm kind of partial to tests. I get nervous working with a codebase that doesn't have some decent test coverage. I cringe at the thought of whole teams committing untested code all day. Coding without tests is coding without a net, and represents a great deal of expensive risk. If I had a nickel for every time a failing test saved me from introducing errors, I would have all the nickels.

At first, I mistook Hickey's point to be one of outright rejection of automated tests, but of course the reality is more interesting than that. What Hickey is really (ahem) *railing* against is the notion that the act of writing tests first will somehow do my *thinking* for me, guiding me to a solution without having to reason about my problem along the way. Sometimes forward progress isn't just about keeping your keyboard active while you guess-and-check your way to a solution. It's test-driven development, not test-adjacent crank-turning (I must admit, though, **TACT** is a pretty sweet acronym).

Tests *are* an invaluable tool, so wield them deliberately. Use them to:

* verify the current state of the system
* confirm expectations
* motivate breaking a problem into smaller pieces
* exercise your API to detect pain points
* prevent regression
* ...

Rather than thinking of tests as guard rails, think of them as support beams in a mining operation. You dig a little, you put up supports, and you dig a little more. You know what direction you are heading. The supports are a necessary component of the operation, but they are neither the gold nor the pickaxe.
