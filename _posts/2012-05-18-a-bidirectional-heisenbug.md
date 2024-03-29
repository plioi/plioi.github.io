---
title: "A Bidirectional Heisenbug"
layout: post
---


I think I may have encountered the world's first bidirectional Heisenbug.

The Heisenberg Uncertainty Principle in physics deals with the problem of observing very small things.  If you want to detect an electron, to "see" where it's going, you have to basically "shine a light on it" by letting it interact with a photon.  At those scales, simply shining a light on something is enough to push it off course!  In short, *observing something can change the thing being observed*.

A <a href="http://en.wikipedia.org/wiki/Heisenbug">Heisenbug</a> is a software bug that seems to correct itself the moment you start trying to find its root cause.  This can be particularly frustrating.  You know the bug is really there.  You've seen its effects.  The moment you start stepping through your program to see it in action, though, everything starts working again.  **Although initially frustrating, this strange behavior can actually provide vital clues about a bug's root cause.**

I encountered the epitome of a Heisenbug today.

The application has a data entry grid reminiscent of a spreadsheet.  The bug report stated that when adding a new record to this grid, some of the necessary data would not get saved successfully, and that lack of data would be apparent on one of the other screens.

Sure enough, I could reproduce the issue consistently and I knew what part of the project to look into first.  I set up some breakpoints in the suspicious part of the code, and *unlike* a typical Heisenbug, I could see some real clues about what was going wrong.  I made a prediction, that if I made a particular column visible in this grid, then I would be able to see the effects of the problem without having to go to the second screen.  This prediction turned out to be correct: I included the column in the grid, and immediately saw the inconsistency created by my earlier attempts to add records to the grid.

I continued attempting to trigger the bug, and step through the suspect code, but everything started to look correct.  The data I expected to be missing was right there, clear as day, in my debugger.  Just before frustration got the best of me, though, I asked myself to look back over every step I had taken, in case it would explain the sudden disappearance of the bug.

**Between the failure scenarios and the working scenarios, I had included that extra column in the grid!**  It turns out that the suspect code depended on the presence of certain columns in order to function properly.

That discovery didn't just explain the bug's sudden disappearance, it actually provided the main clue as to what was really going wrong, and suggested where I needed to apply the fix.  If the bug hadn't disappeared, it probably would have taken *longer* to actually hunt it down.

The Heisenbug-ness of this issue was actually bidirectional.  Not only did the bug stop happening when I attempted to look at it, but the official Steps-to-Reproduce The Issue involve deliberately trying to avoid seeing its effects!
