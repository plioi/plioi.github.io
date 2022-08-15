---
title: "Identifying Incidental Complexity"
layout: post
---


Last week, I mentioned that I was encountering <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Incidental Complexity</a> in a hobby project.  <a href="https://github.com/plioi/rook">Rook</a> is a compiler for a simple programming language, and lately it seems that adding new features is getting harder, suggesting that some of my abstractions are working against me.  Adding new features to this project *should* be getting easier and easier. Over the next few weeks, I'll be writing about the refactorings that get this project back on track.

**Incidental complexity shows up when you are using the wrong abstractions.**  You'll find yourself writing more lines of code *serving* the abstraction than solving the real problem.  Your abstractions should be *removing* obstacles, not throwing obstacles in your face on every other line.

## A Complex Visitor

This week, I've identified an abstraction in Rook's type checker that needed to be removed completely.  The `TypeChecker` class was recently created when I refactored to use the <a href="http://en.wikipedia.org/wiki/Visitor_pattern">Visitor Pattern</a>.  One downside to refactoring to the Visitor Pattern is that, upon success, you are left with a potentially-large class.  The large class has a single overall purpose, but that purpose is rather involved.  There are some pretty tough pros and cons to consider when deciding whether or not to use a visitor, but one of the benefits in this case is that I could finally *see* that I had a great deal of suspicious duplication of code.

With the Visitor Pattern, you have a tree of objects where the nodes of the tree can be of different types.  You have an algorithm you wish to apply to the whole tree, and you want a clean way to recursively traverse all those different node types.  You wind up creating a class that has one method for each concrete tree node type.

The `TypeChecker` class is a visitor that traverses the tree with the goal of building up a brand new, slightly modified tree.  The input tree represents all the Rook source code that has been parsed, and the output tree produced by `TypeChecker` represents that same Rook source code, but with a concrete `DataType` filled in for each node.

When the `TypeChecker` succeeded, you'd get a tree with type information.  When it failed, you'd get a list of error messages instead.  More on that oddity later...

## Type Checking 'if' Expressions

Consider a simple if-expression in Rook, which is equivalent to the `? :` ternary operator in C#:

```
if (count > 0)
  x
else
  y
```

These four lines form a single expression.  The value of the expression as a whole is the value of the true branch (x) or the value of the false branch (y), whichever is actually evaluated at runtime.

The requirements for type checking these expressions are rather simple, but the implementation of those checks had a great deal of incidental complexity.  To pass the type checker, an if expression:

1. must have a condition expression of type bool.
2. must have well-typed body expressions.
3. must have body expressions whose types are the same as each other.


When these requirements are met, the type of the whole if-expression is the type of one of its branches' body expressions.  In the sample above, `count > 0` must be a boolean expression, and `x` had better have the same type as `y`.

## The Troubled Implementation

It just takes a few clear sentences to describe the type checking of if-expressions, but the implementation of those rules is jam-packed with incidental complexity:

```cs
public TypeChecked<Expression> TypeCheck(If conditional, Scope scope)
{
    var Position = conditional.Position;
    var Condition = conditional.Condition;
    var BodyWhenTrue = conditional.BodyWhenTrue;
    var BodyWhenFalse = conditional.BodyWhenFalse;

    var typeCheckedCondition = TypeCheck(Condition, scope);
    var typeCheckedWhenTrue = TypeCheck(BodyWhenTrue, scope);
    var typeCheckedWhenFalse = TypeCheck(BodyWhenFalse, scope);

    if (typeCheckedCondition.HasErrors || typeCheckedWhenTrue.HasErrors || typeCheckedWhenFalse.HasErrors)
        return TypeChecked<Expression>.Failure(new[] { typeCheckedCondition, typeCheckedWhenTrue, typeCheckedWhenFalse }.ToVector().Errors());

    var typedCondition = typeCheckedCondition.Syntax;
    var typedWhenTrue = typeCheckedWhenTrue.Syntax;
    var typedWhenFalse = typeCheckedWhenFalse.Syntax;

    var unifyErrorsA = Unify(typedCondition.Position, NamedType.Boolean, typedCondition.Type);
    var unifyErrorsB = Unify(typedWhenFalse.Position, typedWhenTrue.Type, typedWhenFalse.Type);

    if (unifyErrorsA.Any() || unifyErrorsB.Any())
        return TypeChecked<Expression>.Failure(unifyErrorsA.Concat(unifyErrorsB).ToVector());

    return TypeChecked<Expression>.Success(new If(Position, typedCondition, typedWhenTrue, typedWhenFalse, typedWhenTrue.Type));
}
```

After attempting to type check the condition, true branch, and false branch, we jump through some strange hoops to collect any error messages that were encountered so far.  Then, we declare three more variables with suspiciously-similar names to the variables already in scope.  Next, we 'Unify' the condition expression's type with Boolean and 'Unify' the true-branch's type with the false-branch's type (`Unify` calls are basically assertions like "These two types had better be the same as each other").  *Again*, we jump through some weird hoops to collect any error messages encountered before returning a result.  **On success, the result is a new If tree node with its DataType property populated, and on failure we instead return a list of error messages.**

Reread that last sentence.  It's weird.

We're returning *either* a tree node of type If, or a list of error messages.  In a statically-typed language, returning either one thing or a completely different thing is not strictly doable.  To handle that, **I created an abstraction that had big consequences on the complexity of the type checker.**  The return type, `TypeChecked<Expression>`, is the unfortunate abstraction I needed to remove.

A `TypeChecked`-thing is either a) a thing with a known DataType plus an empty list of error messages, or b) a null thing plus a nonempty list of error messages.  Since this was the return type of the many `TypeCheck` methods, those methods had to do a lot of explicit testing of intermediate results to see which of the two states each result was in.  In the case of failures, these methods had to do a lot of explicit combining of error messages so that the returned object included all errors encountered along the way.

**The fact that this is hard to explain in English should have been my first clue that I was working with a bad abstraction.  A lot of these words just don't show up in the requirements.**

I was writing way more lines of code serving `TypeChecked<T>`'s idiosyncrasies than I was writing to actually implement the requirements!  What, exactly, did this thing buy me?  Also, it's wasteful to spend so much of our time combining and recombining these collections of error messages as they get passed up the recursive call chain.  It's probably not wasteful in CPU cycles (the collections are very small in practice), but was extremely wasteful in brain cycles.

## Don't let the door hit you on the way out!

Let's kick this abstraction out of the project entirely.

The `TypeChecker` class should simply own a growing list of error messages encountered during the tree traversal, so `TypeChecked<T>` would no longer need to store a list of its own.  That would make `TypeChecked<T>` a trivial wrapper of a node, adding nothing of value to that node.  I removed the class, and change occurrences of `TypeChecked<T>` to simply T.

Now, each `TypeCheck` method just returns null when no properly-typed node can be produced, and non-null when one can be produced.  That subtle shift in the meaning of returned objects lets us get rid of a lot of the cruft that was accumulating.  The new version of the if-expression type checker is much more to the point:

```cs
public Expression TypeCheck(If conditional, Scope scope)
{
    var Position = conditional.Position;
    var Condition = conditional.Condition;
    var BodyWhenTrue = conditional.BodyWhenTrue;
    var BodyWhenFalse = conditional.BodyWhenFalse;

    var typeCheckedCondition = TypeCheck(Condition, scope);
    var typeCheckedWhenTrue = TypeCheck(BodyWhenTrue, scope);
    var typeCheckedWhenFalse = TypeCheck(BodyWhenFalse, scope);

    if (typeCheckedCondition == null || typeCheckedWhenTrue == null || typeCheckedWhenFalse == null)
        return null;

    var unifiedA = Unify(typeCheckedCondition.Position, NamedType.Boolean, typeCheckedCondition.Type);
    var unifiedB = Unify(typeCheckedWhenFalse.Position, typeCheckedWhenTrue.Type, typeCheckedWhenFalse.Type);

    if (!unifiedA || !unifiedB)
        return null;

    return new If(Position, typeCheckedCondition, typeCheckedWhenTrue, typeCheckedWhenFalse, typeCheckedWhenTrue.Type);
}
```

It's not perfect, but it's a clear step in the right direction.  **To sum up, when you're working *for* your abstractions, they're not working for you.  Render them pointness, and then drop them like the bad habit they are.**
