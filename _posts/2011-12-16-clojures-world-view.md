---
title: "Clojure's World View"
layout: post
---


The programming language you use has a very large impact on the way you think about problems. Despite superficial similaries, each language comes with its own set of opinions about what is Important, what is Clean, what is Right, and what is Dangerous.

It's a good idea to take a look at the opinions of languages outside of your comfort zone, in case any of those tools can be incorporated into your toolbelt. My language of choice is C#, partly because it has similarly been influenced by a wide range of other languages. You *can* write Java in C#, but you don't *have* to.

<a href="http://clojure.org/">Clojure</a>'s view of the world is dramatically different from C# and Java, even though it runs on the JVM and can interop with Java libraries. There are two main areas where Clojure can be contrasted with C#: its syntax and its approach to state changes.

Today, we'll see how Clojure's approach to syntax can help us achieve brevity in C#. Next week, we'll see how Clojure's opinions can shape the way we think about state change.

## **Clojure Syntax**

Clojure is a dialect of Lisp, which means the first thing you'll notice (is all the (nested (parentheses))). The look and feel of Lisp dialects can be initially confusing to newcomers. Instead of saying `1 + 2 + 3`, we say `(+ 1 2 3)`. The name of an operation comes before its arguments, never between them. This inside-out syntax has an interesting effect: **everything looks the same**. Every construct in the language, from simple expressions, to conditional logic, to function and type definitions `(looks like this)`.

The fact that everything looks the same as everything else is a double-edged sword. I *like* that different constructs in C# *look* different. I find it helps me "chunk" up the screen as I read through something unfamiliar.

On the other hand, the extreme simplicity of Clojure syntax gives us <a href="http://en.wikipedia.org/wiki/Homoiconicity">homoiconicity</a>, meaning that every bit of Clojure code is also a description of a data structure, namely a list of lists. `(* 3 (+ 2 1))` is both an arithmetic expression to be evaluated **and** a list containing three items: the symbol `*`, the number `3`, and another list containing 3 more items.

Because all Clojure code is *also* a literal for a Clojure list, code generation becomes orders of magnitude simpler with Clojure than with C#. In fact, code generation is so common in Lisp dialects that it gets special treatment. A **macro** is a function whose job it is to take in code as its input and produce new code as its output. Even most of the seamingly-built-in keywords are in fact implemented as code-twiddling functions called early in the execution process.

## **Hypothetical C# Macros**

Imagine we had a version of C# that didn't have <a href="http://msdn.microsoft.com/en-us/library/bb384054.aspx">auto properties</a>, but did have macros. With macros, we could effectively add a new keyword to the language, implementing it in a library, to give us the same shorthand:

```cs
using Some.Imaginary.Macro.Library;

public class Sample
{
    prop public string Name;
}
```

Our hypothetical macro function **public macro Code prop(Code c) { ... }** would receive as its input a description of the code between 'prop' and the semicolon, returning a description of the more elaborate property code. The compiler could call that function early on, generating the code we would have otherwise written:

```cs
public class Sample
{
    private string _name;
    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }
}
```

This hypothetical C# wouldn't need **using() { ... }** blocks, or even **foreach** loops, because those keywords could be implemented in a library in terms of other existing constructs. Most language features that we think of as 'syntax sugar' wouldn't have to be thought of by the language designers in advance. With Clojure, developers invent their own new syntax sugar as a part of their daily thought process.

## **Back to Reality**

A future version of C#, codenamed <a href="http://msdn.microsoft.com/en-us/roslyn">Roslyn</a>, will be exposing some of the compiler's inner guts to us as a library. As with Clojure, we'll be able to treat C# code as both a set of instructions and as a data structure. I **seriously doubt** this means that C# will gain macros, but I wouldn't be surprised if we do end up seeing some powerful tools aimed at the same target as macros.

The closest thing to macros that we have today in C# is lambda expressions and their corresponding expression trees. HTML helpers in MVC, for instance, leverage the code-twiddling power of lambda expressions in order to prevent us from reaching for our copy/paste shortcuts, saving us from having to maintain monotonously-verbose view files. Once you hand developers a powerful code-traversal tool like this, you empower them to <a href="http://patrick.lioi.net/2011/12/02/opening-the-black-box/">invent powerful shorthand of their own</a>.

**When you find yourself writing suspiciously-similar code in C# ask yourself, "What would Clojure do?"**
