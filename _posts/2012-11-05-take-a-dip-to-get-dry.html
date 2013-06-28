---
title: Take a DIP to get DRY
layout: post
---

The DRY principle is pretty simple.  Don't Repeat Yourself.  When you find yourself reaching for the copy/paste commands, you're probably creating some technical debt.  If you copy/paste a method or class from one part of your application to another with the intent to make a small change in the copy, there's a very big chance that those two implementations will begin to drift.  Maybe a bug needs to be fixed in both, but the developer performing the fix doesn't know that the copy took place: they'll fix some occurrences of the bug but allow others to continue.  Even when you know about both copies, you may find yourself doing double the work each time you need to make a change to one of them.  It's costly in development time, brain cycles, and QA time.

Strangely, I find that when the technical debt brought on by a DRY violation is <em>particularly</em> bad, the solution involves deliberately causing yet more duplication in the short term.  Sometimes, you need to take a DIP to get DRY.<!--more-->

<h2>A DRY Violation in the Wild</h2>
I was faced with a large DRY violation this week.  Earlier in the history of this project, a class was written in the UI layer to perform some relatively complex validation to ensure the user properly filled out a set of options.  The validation rules weren't rocket science, but they also weren't trivial rules like 'Field A is required' or 'Field B cannot exceed 20 characters'.  All told, it was about 150 lines of logic split out over about 10 methods.  Eventually, the same rules needed to be applied in a related EXE that has no UI, and at that point in time the logic was copied, pasted into an assembly that the other EXE had access to, and tweaked slightly to fit in its new home.  Naturally, the problem got worse since then, as both versions drifted away from each other.

Once this problem was discovered, I was tasked with resolving the duplication by phasing out the deprecated UI implementation.  The new UI would defer to the newer implementation, so both EXEs would share the same code.  Because of the drift, it wasn't clear whether doing this the quick-and-dirty way would be remotely safe: there was a good chance the new implementation was missing important rules enforced in the UI version.  Viewing a diff between the two files wasn't helpful, as so much had been tweaked, renamed, and refactored on both sides.  You could tell they were effectively doing the same thing, but the details were too difficult to discern all at once.

<h2>Duplicate It Purposely</h2>
When faced with this kind of DRY violation, my first step is to make the situation more clear by <em>deliberately introducing even more blatant duplication</em>.  I Duplicate It Purposely (DIP).  I start with a diff between the two files, and identify the low-hanging fruit: code that is only superficially different due to things like variable and method renames.  I start by making superficial changes to the deprecated code, making it look more and more like the version I want to keep around.  The workflow might look like the following:

<blockquote>
<ol>
<li>View a diff.  Rename variable.  Rename variable.  Rename method.  Commit.</li>
<li>View a diff.  Rename method.  Extract method.  Reorder members. Commit.</li>
<li>View a diff.  Inline method.  Extract variable.  Commit.</li>
<li>View a diff...</li>
</ol>
</blockquote>
After these quick, low-risk, automatable refactorings are applied to the deprecated code, the diff becomes more illuminating.  You may finally see that the two implementations truly are doing the same thing, in which case it's time to drop the original in favor of the second.  In my case, on the other hand, I got to a point where it was clear there were a few places where the logic actually did differ.  In fact, the differences highlighted a real bug in the version I had intended to <em>keep</em>, and I never would have realized that just comparing the two original files to each other.  Armed with a useful diff, I was able to alter the newer code to cover the missing scenarios.

At that point, I had two code files that were almost character-for-character the same.  The class names were different, but that was about it.  I could now safely drop the original and point the UI at the newer, corrected, shared version.

Even when fixing a small-scale DRY violation, such as refactoring two suspiciously-similar methods within a single class, I find myself doing this.  I start by making the duplication painfullly obvious by making them look as much like each other as possible.  That extreme duplication highlights the actual differences, giving useful hints as to how to best rephrase both implementations.