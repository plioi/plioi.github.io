---
title: "Opening the Black Box"
layout: post
---


Lambda expressions are unusual in the way they are turned into objects at runtime, compared to other kinds of expressions. The object corresponding with your lambda could either be a delegate type, like `Func<T>`, or an expression type like `Expression<Func<T>>`. The object you'll get to work with depends on what you are passing the lambda *to*:

```cs
public void ReceiveLambdaAsDelegate(Func<int, int> function) {...}

public void ReceiveLambdaAsExpression(Expression<Func<int, int>> expression) {...}

...

ReceiveLambdaAsDelegate(x => x*x);
ReceiveLambdaAsExpression(x => x*x);
```

In the first case, our lambda is received as a delegate type, a function that can be invoked with an integer to receive its square. In this usage, it's just shorthand for a method.

In the second case, our lambda is received as an expression type, meaning that the `expression` parameter's value is an object that *describes what the lambda's source code looks like*. The value of the `expression` parameter is a tree-like object, effectively saying "I am a lambda expression with an x parameter. My body is a multiplication operation, where both the left and right operands are the x parameter." This description of the source code is verbose and traversable, so we can reason about it in a way that would be too difficult to do with the mere string `"x => x*x"`. We can also take this tree, compile it at runtime, and evaluate it for some given input.

This ability has been leveraged, for example, to make strongly-typed HTML helpers: `Html.EditorFor(model => model.ProductName)` will generate an appropriate snippet of HTML by traversing the tree of objects which *describe* the text "model.ProductName". Our HTML snippet will include the given name. By compiling/evaluating the expression at runtime, we can also pick an appropriate control to output based on the type of the property, and can include the current value of the property within that control.

**Because the language designers exposed a *tiny* bit of the compiler's internal data structure, we have access to a great deal of information *about* our code, allowing us to phase out a lot of otherwise monotonous and error-prone busywork.**

## Roslyn

In a future version of .NET, a great deal *more* of the compiler's internal data structures will be exposed as an official part of the framework. This project is called <a href="http://msdn.microsoft.com/en-us/roslyn">Roslyn</a>. As part of this project, **the C# compiler is being rewritten in C#**. They should have called it <a href="http://en.wikipedia.org/wiki/Ouroboros">Ouroboros</a>.

Compilers are usually a bit of a black box. Text goes in, compiled assemblies come out, and throughout that process we don't get to see the intermediate data structures, similar to `Expression<T>`, that are used to perform the compilation.

Since the Roslyn compiler will be implemented with .NET, the development team has the opportunity to expose the whole set of data structures to us as an API, similar to the way expression trees are currently exposed to us for lambdas. As with lambdas, trees are used to represent larger constructs like statements, methods, and classes. During compilation, this tree is traversed and queried to do things like type inference, type checking, IL generation, and the like. By exposing these intermediate data structures to us, we'll be able to benefit from the wealth of information stored in them.

Tools like ReSharper and your favorite HTML-ifying code snippet highlighter need to effectively reinvent some of the wheels found in our current black box. With Roslyn, we'll be able to instead develop smart code-twiddling tools without having to reinvent 80% of a compiler first.

## A Shinier Mirror

You can think of these trees as providing **suped-up reflection**. Reflection lets us traverse already-compiled code for *some* of the information the compiler had at its disposal along the way. We'll be able to traverse these trees for similar information, *before* it gets squashed down into measly, stinking IL instructions:

```cs
//Discovering all method declarations in a given tree:
IEnumerable<MethodDeclarationSyntax> methods =
    tree.Root
        .DescendentNodes()
        .OfType<MethodDeclarationSyntax>().ToList();
```

## Unintended Consequences (The Good Kind)

When lambdas and the expression API were first developed, the language designers didn't necessarily know ahead of time what we'd do with them. HTML helpers in MVC, for instance, were a creative and unanticipated reaction to the language feature. Now that more of the compiler's inner workings will be exposed to us, I'm looking forward to a flurry of creative and powerful code-twiddling tools.
