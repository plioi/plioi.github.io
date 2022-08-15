---
title: "Building a Language"
layout: post
---


I've had an on-again, off-again relationship with a hobby project for the last few years.  I'm (slowly) implementing a compiler for a new language, called <a href="https://github.com/plioi/rook">Rook</a>.  I recently gave a presentation at the <a href="http://www.polyglotprogrammers.org/">Polyglot Programmers of Austin</a> during the mid-meeting break.  The presentation was less focused on Rook itself, as it's constantly in flux, and instead focused on how you might go about implementing a similar project yourself.  Whether you want to implement a full-blown programming language, or the next time-saving structured-text-processing tool like <a href="http://lesscss.org/">Less</a>, much of the same priciples and patterns apply.

As <a href="http://www.headspring.com/headspringers-at-austin-code-camp/">Nolan Egly mentioned last week</a>, I've submitted the longer version of this presentation for this year's <a href="http://austincodecamp2012.com/">Austin Code Camp</a> on June 9.

Although the language's syntax, priorities, and long term vision are a little wibbly/wobbly at the moment, here's a basic rundown of what I have working so far.


## Hello, World!

In this example, we print some text to standard out.  Note that the `Main` function doesn't have to live inside of a boilerplate class.  Since the body of the function is just one little function call, we also don't bother surrounding it with `{` curly braces `}`:

```
void Main()
    Print("Hello, World!")
```

## Everything's an Expression

Well, most things are expressions.  Most of the things you see in a Rook program have a value, even things like `if/else` conditional logic.  Here, the value of the whole `if/else` is the value of the branch that was actually executed.  Since the `Factorial` function's body is just a single expression, we have no need to bother with more `{` curly braces `}` or an explicit `return` keyword:

```
int Factorial(int n)
    if (n==0)
        1
    else
        n * Factorial(n-1)
    
int Main()
    Factorial(10)
```

## Blocks: Fancy Expressions

Sometimes you just need to go into 'statement mode' and declare some local variables to work with.  Rook blocks are special compound expressions.  They are surrounded with `{` curly braces `}` and can contain zero or more local variable declarations followed by one or more body expressions.  The value of the whole block is the value of the last body expression evaluated.

The local variables can have an explicit type declaration if you want one, but when you leave it off it will be inferred just like C#'s 'var' keyword.  Here, Main() will return 50:

```
int Main()
{
    int height = 5
    length = 10
    
    height * length
}
```

## The Billion Dollar Feature

Allowing reference types to be nullable by default in languages like Java and C# has been called <a href="http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare">The Billion Dollar Mistake</a>.  Honestly, how often would it be meaningful for your domain objects to be nulls?  Most of the time, when your Customer/Account/Whathaveyou objects are null, you have encountered a bug and are moments away from a NullReferenceException cropping up *nowhere near* the actual point where the mistake was made.

Nullability should be something you *ask for*, when it has *meaning* in your system.  Also, when you look at a method signature for a method someone else wrote, it would be nice to know right away whether or not the other programmer is taking on the responsibility of handling nulls.  Just like `int` is a different type from `int?` in C#, `string` is a different type from `string?` in Rook:

```
//Value types can be made nullable, like in C#:
int a = 1
int? b = null

//But even reference types are non-nullable by default in Rook:
string str = null // Compile error!
string? nullableStr = null // OK
```

## Collections

In C#, `IEnumerable<T>` has become extremely commonplace.  Any arbitrarily-sized collection that you could loop through implements this interface.  It's useful, prevalent, *and the length of its name drives me bonkers*.  **The most commonly-used things should take the fewest characters to write.**  In Rook variable declarations, `int*` is shorthand for `IEnumerable<int>`.

C#'s basic built in collection type, the one that get special syntax support and shorthand in variable declarations, is mutable.  This is The Million Dollar Mistake.  It's not quite as offensive as nullability-by-default, but it does encourage some brittleness.  If a library call returns an array of something, and you mess with the result, you're probably abusing the intent of the library developers.  This might even give you unintended access to the data that the library is trying to maintain privately.  Let's use things like `List<T>` for mutation, and keep our array-like things immutable.  Again, the most commonplace things should take the fewest characters to write, so I'd rather devote shorthand like `int[]` to something I can really feel safe using everywhere.

In Rook, `int[]` is shorthand for `Vector<int>`, where `Vector<T>` is an *abstract* base class for an immutable array-like collection.  Since it's abstract, there's the potential for multiple implementations benefiting from the same built-in syntax.

Because `Vector<T>` isn't a concrete type, we can play some neat tricks to keep these immutable collections efficient.  `Vector<T>` supports Python-style slicing operations with the [:] operator, which gives results similar to `String.Substring(...)`.  The operation returns a `Slice<T>`, which also happens to implement `Vector<T>`.  This structure allows the slicing operation to have O(1) performance characteristics (taking a large slice of a large collection is as fast as taking a small slice of a small collection).

**Letting the language favor interfaces and abstract types right from the get-go opens the door to neat trickery like fast slicing.**  We preach coding to abstractions when *using* languages, and our languages should start following that advice as well.

Also similar to strings, operations like `Append()` produce a whole new collection instead of modifying the original:

```
int[] digits = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
int[] firstFive = digits[0:5]
int[] lastFive = digits[5:10]
int[] interiorFive = digits[3:8]

firstFive.Append(100) // Equals [0, 1, 3, 4, 100]

interiorFive.With(2, 100) // Equals [3, 4, 100, 6, 7]
```

## Austin Code Camp

All of this is in constant flux as I figure out what I really want this to *be* in the long run, but the implementation and patterns involved have become more and more stable in the past year.  Come on by the Austin Code Camp on June 9 to see how this seemingly-complex task can be broken down into managable pieces.  Be there and be square!
