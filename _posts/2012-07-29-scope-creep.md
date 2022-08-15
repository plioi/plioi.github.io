---
title: "Scope Creep"
layout: post
---


Two weeks back, I said that I was encountering <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Incidental Complexity</a> in a hobby project, and last week I demonstrated <a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">the first of several revisions</a> meant to identify and remove that incidental complexity.

Last week focused on identifying an abstraction that was providing no value at all and needed to be removed.  This week, we'll see an abstraction that was right in its inception, but had become increasingly complex as it took on more and more functionality.

## Rook's Representation of Scope

We all take scope for granted.  A locally-declared variable is visible only within the block it was declared, a method only exists within the class block it is declared in, function parameters only exist within their function, etc.  When we're looking at a line of code and we refer to a variable that *wasn't* declared in the closest containing block, we instinctively look to the next outer block, and the next outer block, until we find it.

If you're writing a compiler, you need some way to represent this concept.  For any point in a given program, you need to be able to look up the type of an identifier applicable at that point.

It turns out that you can easily represent nested scopes using a stack of dictionaries.  Each item in the stack is a dictionary, and a single such dictionary includes the identifier/type pairs that are defined in a single code block.  When the compiler encounters a new block with its own local variables, for instance, it pushes a new dictionary onto the top of the stack and fills it with those declarations.  Upon exiting the block the compiler pops that dictionary off of the stack because those names no longer exist.

When inspecting an expression, the compiler can look up the type of any identifier by starting at the top: if the top dictionary has the identifer, we're done; if it doesn't, the search continues down the stack until it is found.

> In other words, when a human looks for an identifier by moving their eyes to the next-outer-block, the compiler accomplishes the same thing by inspecting the next dictionary down the stack.

## The Troubled Implementation

That's exactly how Rook's own Scope type started: a basic stack of dictionaries, in which key lookup walks through the stack until a match is found.  This served me well, until it started absorbing lots of special cases.  

First, take a quick scan through <a href="https://github.com/plioi/rook/blob/e5ba86001a06b28bdae3e4eb41d31641d08d1d4c/src/Rook.Compiling/Scope.cs">the troubled Scope implementation.</a>  It's not the ugliest thing in the world, but it's certainly not *clean*.  A few problems jump out at me:

1. **It's null-happy:** Since *all* Scope objects have a reference to their surrounding 'parent' scope (aka the next scope down the stack), we have to store parent=null in the global scope.  In several methods, we have to inspect whether parent is null in order to know what *kind* of scope we're dealing with.  There's nothing about "parent == null" that says "this is the global scope", so while reading through this you have to constantly remind yourself what null means.
2. **The localNonGenericTypeVariables field is weird and confusing:** even if you read through all its usages, it's really hard to know what this member field is even used for.  Why does it exist?  What *kind* of scope is it applicable for?  What if I call `TreatAsNonGeneric` against the wrong *kind* of scope?  Ugh.
3. **The CreateTypeVariable field is weird and confusing:** if you were to trace the long series of calls along which this object gets passed in order to arrive here, it would only confuse things further.  We go to great lengths to trick all the Scope instances into having a reference to the same shared Func, just so it can be called by `FreshenGenericTypeVariables`, and *that* method doesn't even really belong in this class in the first place!
4. **Inconsistency in construction:** depending on what *kind* of scope you want, you'll either call a static factory (`CreateRoot`), an instance factory (`CreateLocalScope`), or a 'Try' instance method with an `out` parameter (`TryGetMemberScope`).
5. **What in the heck is a 'root' scope, anyway?**


## Recognizing the Problem

All in all, it is exceedingly easy to misunderstand and misuse this class because it has accumulated so many mixed up concepts over time.

I emphasized the word 'kind' a few times now, and that's a clue as to what the real problem is.  When working within this class, and when using this class from elsewhere in the system...

* you must constantly think about what *kind* of scope you are dealing with.
* you must constantly think about what *category* of scope you are dealing with.
* you must constantly think about what *class* of scope you are dealing with.


Aha!  We've got one class that is trying to represent several classes of scopes!  This isn't the only thing wrong with Scope, and we'll get to the rest in future posts, but the problem to solve this week is to split this class into a few meaningful classes, and clean up as much as we can in the process.  The result won't be perfect, but it will put us into a better position for future refactorings.

> We'll split `Scope` into several classes over the course of several commits.  Each commit will have a small... um... *scope* in order to let our source control document the train of thought.

## Commit #1

<a href="https://github.com/plioi/rook/blob/5b18f455b79b956f8933c73aedfcf33434f55cf1/src/Rook.Compiling/Scope.cs">Extract LambdaScope subclass from Scope, as it is the only kind of scope that can treat type variables as non-generic.</a>

It turns out that type checking lambda expressions is unusual compared to other kinds of expressions.  The fact that lambda expression parameters don't have explicit type declarations means that we have to infer them from how the lambda is *used* and we have to generate some temporary placeholders for those as-yet-unknown types before we've collected enough info to nail them down.

The details are irrelevant for this post, but the important part is the realization that **the `TreatAsNonGeneric` method and the `localNonGenericTypeVariables` list were only ever used when we knew we were dealing with a lambda declaration's scope.**  Therefore, I extracted `LambdaScope` as a subclass of Scope.  This special method and list now live only in `LambdaScope`, which at least gives some clue as to their purpose.

## Commit #2

<a href="https://github.com/plioi/rook/blob/fb9fa6c9853fd1af1b069f5e83246ba806baf266/src/Rook.Compiling/Scope.cs">Rename 'root' scope to 'global' scope because that's what it is.</a>

The 'root' scope was special because it contains built-in definitions, like operators, which should be visible to all parts of a program being compiled.  It never deserved to be called 'root', and should have always been called 'global' to match common terminology and communicate intent.

## Commit #3

<a href="https://github.com/plioi/rook/blob/d4628e28608437500fe80a4df45f8befdb42bed8/src/Rook.Compiling/Scope.cs">Extract GlobalScope subclass from Scope.</a>

Since global scope objects are special due to their initial contents, I extracted another subclass.  The old static factory method CreatGlobalScope became the constructor of the new `GlobalScope` class.

## Commit #4

<a href="https://github.com/plioi/rook/blob/5038da7a86aff5b42d1818cc8af83ba20c764d06/src/Rook.Compiling/Scope.cs">Extract TypeMemberScope subclass from Scope.</a>

Back when I started this series of posts, I mentioned that it was getting harder and harder to add new features due to the growing amount of incidental complexity.  The worst offender was adding the ability to type check method calls.  Type checking function calls ("Function(arg)") had been simple, but type checking method calls ("instance.Method(arg)") had been *shockingly* difficult, and it all came down to the unhelpful version of the `Scope` class.

It turns out that type checking method calls involves yet another *kind* of scope.  When type checking an expression like `username.Trim()`, we look up the type of `username` in the local scope, but what if we then tried to look up `Trim` there too?  If we find "Trim" in the local scope, *that's just an unfortunate coincidence!*  It would be some local variable that happens to be the same name as the method we're calling.

Before I understood this detail, I was running into mind-boggling behavior.  My tests included a code snippet containing a top-level function whose name and arguments were coincidentally the same as those of an instance method.  Type checking calls to that method would work by accident: we were looking in the wrong scope for the Func type of the code being called, and we found the coincidentally-correct answer by accidentally looking up the type of the top-level function!

*Where* should we look up method names?  Well, we can consider the known type of the instance to the left of the "." operator.  For any type, there is a sort of "member scope" containing the names of all types members and their associated Func types.

The offending method here was `TryGetMemberScope`, which constructs this kind of scope.  This method is further complicated by the notion that it might fail for unknown types, so it uses the 'Try' pattern with an `out` parameter.  To make things worse, it doesn't even really make use of any member variables, so it doesn't even need to live in this class!

With this commit, I first moved this method to the one place it was actually used, and then extracted `TypeMemberScope` as another scope.  Like `GlobalScope`, these don't have a 'parent' to defer to when lookups fail.

One really cool thing about this refactoring was that after moving the method over to the class it belonged in, it sort of simplified-and-collapsed-into the only method that made use of it.

> You know you've moved a method to its proper home when, upon arrival, it evaporates.  Every part that disappeared was incidental complexity.

## Commit #5

<a href="https://github.com/plioi/rook/blob/249d33890a621f4c90a6b80e8c5563fcb0020820/src/Rook.Compiling/Scope.cs">Split Scope class into abstract Scope and concrete LocalScope classes, as the 'parent' field is only applicable in local scopes.</a>

This commit addresses the complaint that the class is 'null happy'.  Local scopes defer to their surrounding scope when they don't find an identifier, but global and type-member scopes don't have any surrounding scope to defer to.  By splitting the class into local and non-local versions, the implementations of both got much simpler.  `LocalScope`'s methods are all one-liners, and it's pretty clear what they're doing and why they're doing it.

## Commit #6

<a href="https://github.com/plioi/rook/blob/0efcd783fbec0e62624b4a5716d34122dd97cb8a/src/Rook.Compiling/Scope.cs">TypeChecker is responsible for freshening generic type variables upon Name lookups, rather than having Scope perform that operation.</a>

By the end of the previous commit, we had all the classes we needed: abstract `Scope`, `GlobalScope`, `TypeMemberScope`, `LocalScope`, and `LambdaScope`.  The abstract base class had one glaring offender left to phase out: the suspicious `CreateTypeVariable` field mentioned before, which was passed and passed and passed along to get here just so we could use it in the `FreshenGenericTypeVariables` method.

This 'freshen' method comes into play when type checking calls to functions that have generic type parameters, like `IEnumerable<T>`'s `Select<T>` extension method.  I'd explain the details, **but I don't have to**.  We're talking about scope today, and **this operation has as much to with scope as a washing machine does**.  This operation belongs in the `TypeChecker` class.

> Once this method moved back where it belongs, we could drop all the convoluted passing-along of the `CreateTypeVariable` field.

The results of this commit aren't perfect.  I'm not thrilled that I'm accomplishing this with subclassing: I'd rather Scope be an interface and avoid implementation inheritence.  I hate using `override` and `base`.  The benefit of the subclassing approach is that it was a tool to help me separate these concepts from each other in small steps.  Now that things are separated by purpose, I'm left with something I can actually work with.  I may be able to futher clean things up and avoid subclassing down the road.

## Addressing Scope Creep

<a href="http://en.wikipedia.org/wiki/Scope_creep">Scope creep</a> usually deals with the problem of a project's requirements growing and changing in an unwieldy way, but the same thing happens to your classes in response to those requirements changes.

> It's important to cultivate a negative vicseral reaction to growing complexity, rather than to just accept it as necessary.

When one of your classes starts to absorb competing concerns, you may be able to turn things around by recognizing what types are hidden in there, wanting to escape and become classes of their own.
