---
title: "Detect, Reflect, Decomplect"
layout: post
---


This is part 6 of an N-part series on identifying and resolving incidental complexity in your software projects.  This post assumes you've already read the earlier posts:

1. <a href="http://patrick.lioi.net/2012/07/13/essential-vs-incidental-complexity/">Essential vs Incidental Complexity</a>
2. <a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">Identifying Incidental Complexity</a>
3. <a href="http://patrick.lioi.net/2012/07/29/scope-creep/">Scope Creep</a>
4. <a href="http://patrick.lioi.net/2012/08/04/spaghetti-alla-code/">Spaghetti alla Code</a>
5. <a href="http://patrick.lioi.net/2012/08/11/primitive-obsession/">Primitive Obsession</a>


The act of "complecting" two things is to intertwine or braid them together in such a way that creates complexity.  Most incidental complexity in our code comes from getting separate concepts twisted up in ways that make things more complex without moving us any closer to our goals.  Sometimes we intertwine concepts so tightly that we don't even realize it's happened.  In this post, we'll see how I finally untied a code knot that has been staring at me for about two years.

## A Quick Review

<a href="http://patrick.lioi.net/2012/07/21/identifying-incidental-complexity/">A few weeks ago</a>, we saw some improvements made to the TypeChecker class.  This class traverses a tree representing all the source code input to the compiler.  If the input source code has an if-expression, then the tree will contain a node (an instance of the `If` class) that describes what the raw source code looked like.  As TypeChecker traverses the tree, it attempts to find an appropriate type for each node.  Since Rook's if-expressions behave like C#'s `?:` ternary operator, the type of an `If` is the type of the positive and negative branch expressions.

In that post, the original implementation for if-expression type checking looked like this:

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

This implementation used a needlessly-complex return type which motivated extra-complicated `return` statements.  That return type *complected* success (a new tree node with a meaningful Type property) with failure (a list of CompilerError messages to propagate up the tree).  Braiding those two ideas led to some verbose sanity checks after each step the method performed, and led to some verbose error-message-propagation in each `return` statement.

I improved this situation by untying these ideas.  TypeChecker started tracking a single growing list of error messages encountered so far, so that returned objects in the tree traversal methods didn't need to pass error messages back to their caller.  At this point, we started returning `null` every time the method found an error:

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

## Return Statements are "Extract Method" Fences

Surely this change was an improvement, but it didn't dramatically improve the health of the TypeChecker class.  TypeChecker is a beast.  The TypeCheck methods for several of the tree nodes are suspiciously similar to each other, so I started out by trying to factor out common helper methods.

I failed immediately every time I tried to do so.

The problem was that most of these methods followed the following pattern:

1. Do one step of the work.
2. If that step failed, return early (null).
3. Do one step of the work.
4. If that step failed, return early (null).
5. ...


Methods will multiple returns are not inherently *bad*.  I usually limit myself to using multiple returns when a method starts with some initial "guard" statements: "if this call was kind of pointless, return early".  Doing so can help to simplify the remainder of the function by clarifying assumptions up front.  Returning multiple times throughout the *meat* of a method, though, is usually a problem.

> Each time you leave a `return` statement in the body of a longish method, you are really installing a fence.  You aren't going to be able to successfully apply an "Extract Method" refactoring *across* those fence boundaries (try it!), so your method will always be longish with respect to the number of fences.

These fences have kept the TypeCheck methods too longish for about 2 years.  I tried to find ways around this countless times, always resulting in something even more complex than I started with.

## Still Complecting Failure with Success

Even after the previous improvement to the meaning of returned objects from these methods, I was still complecting success (non-null return) with failure (null return).  Sure, the error-message-collecting code was out of the way now, but I still had to inspect the result of each call to a `TypeCheck` method.  In the `If`-expression example above, I couldn't trust the returned objects from the three calls to other `TypeCheck` methods.  I would call it, inspect the result, and then use that to decide whether I should propagate failure by returning null early.

Then it hit me.  What if we could just keep on going if we ran into trouble during the other `TypeCheck` calls?  Compiler errors aren't exceptions: you don't want to just stop looking for errors when you encounter a single problem.  By returning early, I wasn't just making the code more complex.  I was actually limiting the amount of useful output I was ultimately reporting to the user.  Imagine how annoying it would be if the C# compiler just discarded every other error message it could have reported to you!

## Non-null Nulls

Just removing the early return statements wouldn't be enough, of course.  By continuing the process with nulls in play, I'd immediately start running into `NullReferenceException`s.

Even though the methods weren't returning nulls themselves anymore, the `Type` property on the returned objects could still be null.  Before type checking, every node in the tree had a Type property initialized to null, meaning a type hadn't been found for the node yet.  Some of the methods in the type checker had to inspect those Type properties in order to make decisions, and nulls would start to become a problem at runtime.

To resolve this, I had to apply the Null Object pattern before I could actually remove the early return statements.  With this pattern, you introduce a class that represents the simplest, most-null-like version of a type as possible.  In this case, I needed something that implemented my `DataType` abstract class while being completely uninteresting on its own.  Instances of this new <a href="https://github.com/plioi/rook/commit/52686db2b801f414c932f92e390f44ab68c7040b">`UnknownType`</a> class could then be liberally sprinkled over the whole tree prior to type checking.  As the new and improved type checker encountered errors, it could safely proceed to find yet more errors knowing full well that in the worst case it would be working with `UnknownType` instead of nulls.

> Using the Null Object pattern, I could safely remove the problematic early-return statements.  The type checker now finds all the errors it can, and our return values no longer complect success with failure.

The many `TypeCheck` methods are now much shorter, easier to read, and lack the fences that would stand in the way of further refactorings.

## Before and After the Return Value Decomplection

**If-Expression Type Checking**

*Before:*

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

*After:*

```cs
public Expression TypeCheck(If conditional, Scope scope)
{
    var position = conditional.Position;
    var condition = conditional.Condition;
    var bodyWhenTrue = conditional.BodyWhenTrue;
    var bodyWhenFalse = conditional.BodyWhenFalse;

    var typedCondition = TypeCheck(condition, scope);
    var typedWhenTrue = TypeCheck(bodyWhenTrue, scope);
    var typedWhenFalse = TypeCheck(bodyWhenFalse, scope);

    Unify(typedCondition.Position, NamedType.Boolean, typedCondition.Type);
    Unify(typedWhenFalse.Position, typedWhenTrue.Type, typedWhenFalse.Type);

    return new If(position, typedCondition, typedWhenTrue, typedWhenFalse, typedWhenTrue.Type);
}
```

**Class Declaration Type Checking**

*Before:*

```cs
public TypeChecked<Class> TypeCheck(Class @class, Scope scope)
{
    var Position = @class.Position;
    var Name = @class.Name;
    var Methods = @class.Methods;

    var localScope = scope.CreateLocalScope();

    foreach (var method in Methods)
        if (!localScope.TryIncludeUniqueBinding(method))
            return TypeChecked<Class>.DuplicateIdentifierError(method);

    var typeCheckedMethods = TypeCheck(Methods, localScope);

    var errors = typeCheckedMethods.Errors();
    if (errors.Any())
        return TypeChecked<Class>.Failure(errors);

    return TypeChecked<Class>.Success(new Class(Position, Name, typeCheckedMethods.Functions()));
}
```

*After:*

```cs
public Class TypeCheck(Class @class, Scope scope)
{
    var position = @class.Position;
    var name = @class.Name;
    var methods = @class.Methods;

    var localScope = CreateLocalScope(scope, methods);

    return new Class(position, name, TypeCheck(methods, localScope));
}
```

**Function Declaration Type Checking**

*Before:*

```cs
public TypeChecked<Function> TypeCheck(Function function, Scope scope)
{
    var Position = function.Position;
    var ReturnType = function.ReturnType;
    var Name = function.Name;
    var Parameters = function.Parameters;
    var Body = function.Body;
    var DeclaredType = function.DeclaredType;

    var localScope = scope.CreateLocalScope();

    foreach (var parameter in Parameters)
        if (!localScope.TryIncludeUniqueBinding(parameter))
            return TypeChecked<Function>.DuplicateIdentifierError(parameter);

    var typeCheckedBody = TypeCheck(Body, localScope);
    if (typeCheckedBody.HasErrors)
        return TypeChecked<Function>.Failure(typeCheckedBody.Errors);

    var typedBody = typeCheckedBody.Syntax;
    var unifyErrors = Unify(typedBody.Position, ReturnType, typedBody.Type);
    if (unifyErrors.Any())
        return TypeChecked<Function>.Failure(unifyErrors);

    return TypeChecked<Function>.Success(new Function(Position, ReturnType, Name, Parameters, typedBody, DeclaredType));
}
```

*After:*

```cs
public Function TypeCheck(Function function, Scope scope)
{
    var position = function.Position;
    var returnType = function.ReturnType;
    var name = function.Name;
    var parameters = function.Parameters;
    var body = function.Body;
    var declaredType = function.DeclaredType;

    var localScope = CreateLocalScope(scope, parameters);

    var typedBody = TypeCheck(body, localScope);

    Unify(typedBody.Position, returnType, typedBody.Type);

    return new Function(position, returnType, name, parameters, typedBody, declaredType);
}
```

## Detect, Reflect, Decomplect

When you find yourself looking at suspiciously-complex code,

1. **Detect** whether you have two or more ideas all tied up and twisted together.
2. **Reflect** on what a successful untwisting would look like.  Where will each idea live once they are separated?  Will you lose functionality in the process?  Will you gain some functionality that has been hindered by the complexity?  Will this effort open the door to further cleanup?
3. **Decomplect** the hell out of that mess and bask in the glory of your cleaner codebase.
