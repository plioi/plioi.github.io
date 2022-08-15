---
title: "Primitive Obsession"
layout: post
---


This is part 5 of an N-part series on identifying and resolving incidental complexity in your software projects.  This post assumes you've already read the earlier posts:

1. <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Essential vs Incidental Complexity</a>
2. <a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">Identifying Incidental Complexity</a>
3. <a href="http://patrick.lioi.net/2012/07/29/scope-creep/">Scope Creep</a>
4. <a href="http://patrick.lioi.net/2012/08/04/spaghetti-alla-code/">Spaghetti alla Code</a>


## Assumed Disaster, Assumed Success

My younger brother will be starting on a PhD in San Diego this fall, and we were recently talking about the drive he'll be taking from Austin.  Wearing the Overprotective Sibling Hat, I was asking about his contingency plans should something go wrong: "If your vehicle breaks down, who are you planning to call for help?" and the like.  I could tell I was starting to annoy, so I said "Oh, sorry, that's the software guy talking.  Something can always go wrong."  In response, he said, "Well I'm a mathematician.  Everything is always perfect."

What did I mean by "Something can always go wrong"?  Even the smallest decisions a programmer makes can have far-reaching consequences, making a system more complex than it needs to be and opening the door for bugs.  This week, we'll see how "Primitive Obsession" affected a very small decision in my compiler, and that poor decision spread like a weed.  It caused me to write awkward methods across a class hierarchy, forced me to pass around more objects from one function call to the next when doing so shouldn't have been necessary, and opened the door to what would have been **extremely** troublesome bugs.

## Primitive Obsession

<a href="http://c2.com/cgi/wiki?PrimitiveObsession">Primitive Obsession</a> is when you focus too much on simple data types like ints, bools, and strings when it would be more helpful and communicative to use classes composed of those simpler types.

For instance, I once worked on a system that dealt with phone numbers, and for a long time we just passed phone numbers around as strings.  The downside was that most functions receiving these strings had to be paranoid that they were being passed a string that wasn't really a phone number, like "", "Dr. Martin Van Nostrand", or "555-0100" (which looks like a phone number but is reserved for TV).

Suspicious of incoming phone number strings, most functions would start by sanity checking the input by calling one of several static validation functions.  It was annoying, bug prone, and inflated the size of the code.

After introducing a smarter PhoneNumber class, we had less to worry about: validating and parsing a string was the only way to get an instance, so if you were passed one of these you could assume that it was already usable.  Even better, lots of those free-floating static string-twiddling methods now had a new home.

## Type Variables

In my compiler, I over-ephasized the importance of a simple int, and that obsession led to some problems.  Before we get to that, though, we should talk a little about Type Variables.

When a compiler is type-checking your code, it often has to reason about the type of an expression for which there is not yet enough information.  When type checking an 'if' condition, we know that the condition's type had better be boolean, but at the moment we are inspecting the if statement there might not be enough information about the types of variables appearing in that condition.  Maybe the condition refers to a lambda expression parameter, and we won't be able to nail down the type of that parameter until we *later* discover the `Func` type of the lambda as a whole.

> When the compiler has to start reasoning about types it doesn't know about yet, it has to introduce a temporary placeholder: a stand-in for the real type that we will eventually nail down to something concrete.  We (unfortunately) call these Type Variables.

It's easy to get type variables confused with generics, because there's a definite overlap.  Consider a generic method in C#:

```cs
public T Echo<T>(T input)
{
    return input;
}
```

When a compiler makes its first pass through this code, it can store the definition of Echo using a type variable everywhere we see a `T` to mean "Echo does stuff with some type but we don't know what that type is yet."

An important quality of these *generic type variables* is that every time we call Echo, we need to nail down the type of `T` all over again.  A first call to Echo(1) nails down the type variable to int, and a subsequent call to Echo("Hello!") nails down the type variable to string.

We can't, however, conclude that type variables are synonymous with generic type parameters like `T` above.  There are lots of reasons a compiler won't yet know about a type; these placeholders are not limited to explicitly-declared generic type parameter like Echo's `T`.  Some of these type variables are *nongeneric*, meaning their behavior is a little different from the Echo example.  With Echo, we got to rediscover the concrete type with each call.  When we're not dealing with generics, though, we can only nail things down *once*.

This is pretty complicated.

Fortunately, there are only a few details of this discussion that are necessary to understand the rest of this post:

1. A compiler needs to reason about types before it has enough information to know exactly what the type of an expression really is.
2. In these cases it generates a temporary placeholder for a type, called a Type Variable.
3. Eventually we gather clues to nail down each Type Variable to something specific like bool, string, Customer, ...
4. Some type variables behave like familiar generics, and some are *special* with their own behavior.


## Obsessing Over Integers

Very early in development of my type checker, I was guilty of Primitive Obsession.  I needed a TypeVariable class that implemented my DataType interface so they could be used alongside concrete types like Integer, Boolean, and the like.  Similar to a .NET Guid, I needed to simply generate *some new unique thing* that wouldn't be considered equal to the ones generated before.

I implemented TypeVariable as a trivial wrapper around an int, and made it so that each time you created a new one it would use a unique int that hadn't been used yet.  The first time you needed a placeholder for a type, it would be Unknown Type #0, the second would be Unknown Type #1, etc.

> I thought I was avoiding Primitive Obsession by wrapping this int in a meaningfully-named class that took part in my domain.  However, my mistake was that I *continued to think of these things as glorified ints*.

Because I always thought of them as a primitive type, it never occurred to me that I might want to put more information or behavior in that class.  When my code started to care about the distinction between *generic* and *nongeneric* type variables hinted at above, *I couldn't see that that concept belonged in the TypeVariable class* and instead I threw that concept around wherever I could to get the system to work.

The generic/nongeneric distinction wound up in the Scope classes that I've been discussing here the last few weeks.

Scope is *supposed* to be a dictionary-like type that stores the declared type of identifiers in scope.  If the code being compiled has a local `int x` varaible and a global `Echo` function, this dictionary-like thing would have a key `"x"` with value `"int"` and a key `"Echo"` with value `"Func<T, T>"`.

That's all Scope was supposed to be, but it also took on the responsibility of knowing which TypeVariable instances were generic and which ones weren't.  **Wat.**

Here's what the troubled version looked like.  Note especially the LambdaScope class, which took on the awkward responsibility of remembering which TypeVariables were *special*:

```cs
public interface Scope
{
    bool TryIncludeUniqueBinding(Binding binding);
    bool TryGet(string identifier, out DataType type);
    bool Contains(string identifier);
    bool IsGeneric(TypeVariable typeVariable);
}

public class GlobalScope : Scope
{
    ...

    public bool IsGeneric(TypeVariable typeVariable)
    {
        return true;
    }
}

public class LocalScope : Scope
{
    ...
 
    public bool IsGeneric(TypeVariable typeVariable)
    {
        return parent.IsGeneric(typeVariable);
    }
}

public class LambdaScope : Scope
{
    private readonly List<TypeVariable> localNonGenericTypeVariables;

    ...

    public bool IsGeneric(TypeVariable typeVariable)
    {
        return !localNonGenericTypeVariables.Contains(typeVariable)
            && lambdaBodyScope.IsGeneric(typeVariable);
    }

    public void TreatAsNonGeneric(IEnumerable<TypeVariable> typeVariables)
    {
        localNonGenericTypeVariables.AddRange(typeVariables);
    }
}
```

Holy cow, that is complex!  The different implementations of IsGeneric work together to make up for the fact that we couldn't just *ask* each TypeVariable whether or not it was had generics-behavior, and that all grew from my preoccupation with these things being glorified ints.

What's worse is that these implementations of IsGeneric only give accurate answers when you *happen* to call them correctly.  Note how GlobalScope always returns true.  If I ever called that implementation for a TypeVariable that really *isn't* generic, I'd be working with a lie.  The compiler didn't *happen* to rely on any liars yet, but if it ever did I'd be faced with some incomprehensible bugs.

## Fixing TypeVariable and Scope

One simple change to the TypeVariable abstraction was all I needed to fix this mess.

The first commit addressed the main problem: <a href="https://github.com/plioi/rook/commit/bc56fe6495d0fe56702ee825673b9ad74b56d977">TypeVariables know whether or not they are generic type variables upon construction. Scope no longer tracks or cares about whether type variables are generic.</a>  In this commit, TypeVariable gained a new IsGeneric property, all of the IsGeneric implementations were removed from Scope classes, and usages got cleaned up.  I even got to remove a unit test that in hindsight *asserted the horribly bug-prone behavior mentioned earlier*.  I was so confused by my own complexity that I was *asserting that it should behave dangerously*!

After this commit, I realized that the LambdaScope class was now completely pointless.  It wrapped an instance of another Scope class and all the methods now just defered to that other instance.  LambdaScope no longer added any new ideas, <a href="https://github.com/plioi/rook/commit/e965b694fe9b7daa41367632be33a570365fa8f8">and could be removed</a>.

> In addition to all of this cleanup and simplification, I'm starting to see how TypeVariable could start to absorb some more misplaced behavior from the still-too-complex TypeChecker class.  The fact that it's getting *easier* to start seeing what else can be improved is a sign that the last few weeks' worth of refactorings are actually starting to pay off!

## Beware Primitive Obsession

As we've seen, Primitive Obsession isn't just about focusing on simple types when your own domain objects are better.  Even when I made my ints into TypeVariables, I was still thinking in terms of trivial types.

When focusing too much on trivial types, we forget to place behaviors or related information in their most appropriate home.  This mistake leads us to place behavior and state *just about anywhere it'll work*, leading to more mistakes.

> A tiny bit of obsession with a tiny type can grow through your project like a weed, spreading complexity to everything it touches.

To resolve these problems, try to recognize that your simple types may deserve to be classes of their own.  Once you've given these concepts a useful name in your domain, don't stop there!  Make these new types smarter, taking on behavior appropriate to the type so that the rest of your system doesn't have to compensate for them.
