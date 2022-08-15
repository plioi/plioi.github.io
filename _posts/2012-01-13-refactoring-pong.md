---
title: "Refactoring Pong"
layout: post
---


One of my first successes with refactoring was in a text-mode Pong clone.  My parents got <a href="http://en.wikipedia.org/wiki/Turbo_Pascal">Turbo Pascal</a> for my brother and me when I was in middle school.  The first nontrivial thing I made was Pong.  Instead of plotting individual pixels, the screen was drawn with plain text.  An asterisk was the ball, and the paddles were drawn with vertical | bars.

I was extremely proud of the way the ball would bounce around the screen.  It would detect when the ball hit a side wall or a paddle, changing direction.  At first, the code that handled this collision detection was enormous.  I didn't know what I was doing, so every step of the way was very verbose.  I didn't quite start out with the right data structure to represent the ball, so every time I wanted to *say* something about it, I'd write another awkward function that *worked* but made things ever-more *complicated*..

We've all experienced that feeling on at least one past project.  "This thing is getting away from us, and fast!"

Every once in a while, I'd realize that a few lines here or there could be simplfied.  I'd see that if I made a particular helper function, then the body of my 4-page switch statement could be shortened to 1 page.  I chipped away at the stone, a little bit at a time.  Holy crap, that was fun.

At one point I realized that if the ball's *speed* in the X and Y directions could be *negative*, I didn't have to track *direction* as a separate concept via an enum.  After that, all the ball-bouncing code collapsed on its own, as if by magic, down to a few lines.  Best of all, those few lines made sense.  No gargantuan switch statement.  No awkwardly-phrased statements.  **I was finally saying what I meant.**

On real-world software projects, we can't allow ourselves to get into the trap of perpetually-refactoring-everything for no good reason.  It is fun and intellectually satisfying, but once you let yourself lose site of where you are *headed*, you might be introducing some very expensive waste.  Still, you have to weigh the cost of this effort against the cost of *not* doing it.  **Refactoring while you make forward progress**, and not as an occasional recess from work, may help you to make these decisions more responsibly.

If you're writing Pong, though, just have fun.
