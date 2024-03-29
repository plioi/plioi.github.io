---
title: "Meta-Yak Shaving"
layout: post
---


<a href="http://joi.ito.com/weblog/2005/03/05/yak-shaving.html">Yak Shaving</a> is a term used to describe one of two situations: when you're performing some seemingly-wasteful activity that ultimately helps you reach your actual goal, or when you fixate on a mostly-wasteful activity while procrastinating about your actual goal.

With software, this usually presents itself as spending *way* too much time refactoring without making meaningful forward progress.  Don't get me wrong - refactoring should be a daily activity to ensure that you don't get overwhelmed by mounting complexity, but when you're refactoring perfectly-fine code again and again because it's kinda fun to do so, you're yak shaving.

On a project for which time and money are valuable to the interested parties, yak shaving is unacceptable.  However, I think that developers should feel free to do this on a side project for the sake of *practice*.  If I violently overthink a refactoring effort on a project that affects nobody else, then when I encounter a similar problem on a for-pay project I won't *need* to waste time coming to the same conclusion.  "When I had infinite time and zero dollars on the line, I solved this problem like so.  So let's just do that this time right away!"

## Code Metrics

I place relatively little value on code metrics, especially when they are used to drive decisions.  For instance, if you're deciding what to do with your day *purely* by inspecting a code coverage report, you are staring directly into the business end of an inverted horse cart.  Instead, good testing habits will give you high coverage anyway.  Better yet, high code coverage born from good habits will be **meaningfully**-high; it's easy to make changes that *trick* a metric into looking better than it is, giving you a false sense of security.

The little value I do place on metrics comes when I want to take an occasional snapshot of a project, in case it can point out some useful patterns or trouble areas.  I did this for my <a href="https://github.com/plioi/parsley">yak shaving project</a> and found some interesting things about my own coding style.

## Lines of Code

LOC counts are largely useless as a code metric, but comparing lines of library code versus lines of test code is interesting: Parsley has 585 lines of library code, and *847* of test code.  I was expecting a 50/50 split.  This project lends itself to having a lot of unit tests: it doesn't go out-of-process for anything, and has no UI.  Still, this ratio is pretty darn high.  I'm glad I've put in the effort, though, for an unusually-high test/library ratio, as the problem being solved involves a great deal of potential edge cases and, more importantly, the problem being solved is nearly impossible to step through with a debugger.

Lesson learned: unit testing can help to mitigate the risks of using a hard-to-step-through design.

## Depth of Inheritance

Most of the inheritance in this project is interface-based.  The largest example of implementation-inheritance is only 3 levels deep, and <a href="https://github.com/plioi/parsley/blob/2d113754b6e3e68a43e624987fb7ed5c4dd99870/src/Parsley/TokenKind.cs">the classes involved</a> have a very narrow focus.  This is an example of the Template Method pattern.  There's the 'real' class and then a few convenience subclasses that fill in a tiny bit of the work.

Lesson learned: with infinite yak shaving time, I avoid most implementation-inheritance.

## Conditional Logic

Another surprise: the 'else' keyword appears only 4 times in those 1432 lines.  Weird.  I certainly don't consciously think of 'else' as somehow *bad*.  The explanation comes in seeing how else conditional logic presents itself.

First, polymorphism hides some conditional logic, as it implies statements like "if the runtime type of x is T1 then call T1's implementation, **else** if the runtime type of x is T2 then call T2's implementation, **else**..."

Second, I find that I have several methods that take on the following form:

```cs
public ReturnType Method(Input input)
{
    if (some condition)
        return something;

    return somethingElse;
}
```

In every way that matters, there's an 'else' in that code block.  It's just implied.  I think my accidental aversion to 'else' is really a consequence of two other things:

1. My class methods are very short and make few decisions. They have cyclomatic complexity around 1.6, meaning each method makes about 1 decision before returning, giving little reason to require nested if statements.  Also, my methods tend to be very short, averaging 3 lines.  Most of my if statements therefore take the form "if (condition) return A; else return B;" in which 'else' is superfluous.
2. ReSharper suggests removing 'else' whenever it is superfluous.


Lessons learned: with infinite yak shaving time, I a) perform extract-method refactorings until my methods are extremely short, with each method making about 1 decision; b) use interface-inheritance and Template Method to simplify the most complex logic; and c) discard as many characters as I can get away with, such as when ReSharper suggets removing 'else'.

## Mutation

This one isn't a lesson learned because it's always on my mind already, but when given infinite yak shaving time I avoid mutable state like the plague.  Most of my class fields are declared as readonly, and most of my auto-properties are declared as `{ get; private set;}` and only get set within the constructor, making them effectively readonly too.  Most of my local variables are given a value when they are declared and then never get overwritten.

I found about 10 mutable variables in this whole project, and they were all local to a method, so the state change wouldn't have any affect on the complexity of the rest of the library.  My aversion to mutation leads me to write code in the functional-style for the most part, but when that *inevitably* gets awkward during an intensively bit-twiddling method, I know it's time to escape into statements-and-mutation-mode.

The only interesting bit of mutation comes in the <a href="https://github.com/plioi/parsley/blob/2d113754b6e3e68a43e624987fb7ed5c4dd99870/src/Parsley/TokenStream.cs">TokenStream</a> class, which repeatedly mutates a <a href="http://patrick.lioi.net/2012/06/15/public-private-super-private/">super-private</a> IEnumerator by calling its `MoveNext()` method.  Amusingly, this admittedly-complex class exists precisely so that you can traverse an IEnumerable in an apparently-mutation-free way.  My worst offender in the fight against mutable state exists in order to cover up some mutable state!

## Conclusion

With infinite time and no budget, I write code in a particular style.  It drifts over the years as I gain experience, but certain trends are already apparent, and this experience influences my work on for-pay projects.  I recommend setting aside one project-of-no-consequence for a similar effort, so that you can also discover your own trends.  **What do you find yourself doing, when you have the opportunity to write something that is truly to your liking?**
