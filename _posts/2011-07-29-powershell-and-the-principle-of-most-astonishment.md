---
title: "PowerShell and the Principle of Most Astonishment"
layout: post
---


When designing a user interface, it's important to avoid conflicts between what an action actually does and what the user *expects* it to do. Our goal is to minimize surprises to ensure a smooth end-user experience. This is called the <a title="Principle of Least Astonishment" href="http://en.wikipedia.org/wiki/Principle_of_least_astonishment">Principle of Least Astonishment</a>.

This principle should also be important to language designers. Just like a slick, AJAX-y web page is a user interface for the end user, a programming language's syntax is a user interface for developers. When the rules of a language conflict with a typical developer's mental model of what the rules *might* be, we have a problem every bit as serious as a confusing user-facing dialog.

After running into some hurdles using PowerShell, it seems that this language follows the Principle of *Most* Astonishment. Because its actual model conflicts with the mental model most developers take for granted in similar languages, it is very easy to run into some frustrating bugs.

Programming lanaguages can be dropped into several broad categories when they encourage similar mental models. For the Imperative Object-Oriented Languages, this makes it easy to start with Python and then learn Ruby, Java, and the like: they all have roughly similar notions of imperative control flow, classes, methods, arguments, return values, and I/O. Since PowerShell is an imperative .NET language, it is all too easy to make incorrect assumptions about how PowerShell will behave, even in the simplest of programs.

When learning PowerShell, it's important to avoid bringing all this imperative language baggage along for the ride. Start by realizing that **PowerShell functions are not functions**! Rather, they are commands.

PowerShell is meant to replace old-school BAT files and as such its syntax and semantics lean heavily towards command-line instructions where I/O is king. In most imperative languages, functions take in arguments and return values, occasionally performing side-effects like I/O. In PowerShell, commands take in arguments, and then **output values throughout their execution**. The caller of a PowerShell command can often pretend that it is returning a value upon its completion, but it is always more accurate to say that all of the side-effect output of a command is made available to the caller upon its completion. Consider a simple function which intends to calculate some values while only returning a single value:

```
function Output-And-Return($x)
{
    $x
    ($x + 1)
    return $x + 2
}
```

If we call this command with $x = 1, the caller receives an *array* as the return value. The array contains three values: [1, 2, 3]. What's going on here? First, every expression that gets evaluated will be included in the output. When multiple values are output like this, they get packaged up into an array for you. Second, using the 'return' keyword with a value on the right-hand-side is misleading. Use the 'return' keyword only when you need to exit from a command early. The above code is actually equivalent to the following:

```
function Output-And-Return($x)
{
    $x
    ($x + 1)
    $x + 2
}
```

"return 5" really means "include 5 in the array of output and then return control to the caller". Imagine how frustrating it can be to run into a problem like this. You start adding print statements to your buggy function in order to diagnose what is happening, and the very act of writing that output changes its return value!

There are ways to prevent expressions from being returned to the caller. You can use the built-in command **write-host** to write output to the console without it being included in the output array. Assignment statements are also suppressed from the output because they represent intermediate work being performed.

Any developer who approaches the first Output-And-Return definition above will make false assumptions about the behavior of the command. By recognizing that the PowerShell model differs from a typical model of functions-returning-values, you too can avoid mind-boggling debugging sessions.
