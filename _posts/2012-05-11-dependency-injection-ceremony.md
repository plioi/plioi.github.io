---
title: "Dependency Injection Ceremony"
layout: post
---


The vast majority of the time, I enjoy working with <a href="http://en.wikipedia.org/wiki/Dependency_injection">dependency injection</a>.  Sometimes, however, I find myself rolling my eyes at the amount of "ceremony" boilerplate code it takes in order to take advantage of it.  *Disclaimer: This is more a limitation of C# having been created before DI containers became common than a problem with DI itself.*


DI and its benefits have been written about for years, but the short version is that your classes can accept interface arguments through their constructors, and these arguments represent the class's dependencies.  A simple example can be seen in the <a href="https://github.com/DarthFubuMVC/fubumvc-examples/blob/master/src/Actions/HandlerStyle/SimpleWebsite/Handlers/Movies/ListHandler.cs">FubuMVC samples</a>:

```cs
public class ListHandler
{
    private readonly IRepository _repository;

    public ListHandler(IRepository repository)
    {
        _repository = repository;
    }

    public ListMoviesViewModel Execute()
    {
        return new ListMoviesViewModel { Movies = _repository.Query<Movie>() };
    }
}
```

Here, the ListHandler class is declaring, incidentally to the compiler but primarily to the *reader*, that it depends on a repository to do its work.  Rather than construct a `new Repository()` itself, it assumes a pre-constructed `IRepository` will be provided.

At a minimum, this is convenient in the case that the `Repository` constructor itself has potentially-tricky-to-instantiate arguments, each of which may have their own constructors to call, ad nauseum.

More importantly, since your class no longer has to 'new up' the dependencies itself, it is decoupled from the concrete implementations of those dependencies.  At test time, you can pass in a mock or stub implementation to prove that your class makes use of the abstraction as expected, and at run time you can trust a tool like <a href="http://docs.structuremap.net/">StructureMap</a> to create and pass in the real implementation.  When looking at a class written in this style, you can quickly see what it depends on by looking at its constructor, rather than looking through the entire class definition.

## Playing Devil's Advocate

The ListHandler class is easy to read, easy to understand, so easy to test that it might not be worth the bother, and it absolutely follows the Single Responsibility Principle.  It's simple, basically the "Hello, World!" of Dependency Injection, but simple is a good thing to shoot for whenever we put our hands on a keyboard.  It's a Good Class.

However, when I find myself writing similar single-responsibility classes with a dependency or two passed to the constructor, a little imaginary Devil pops up on my left shoulder, accompanied by his Angel twin on my right.  The Devil wispers, "Why in the *heck* are you writing all that boilerplate!?" and the little Angel looks at the class and says, "Yeah, I hate to say this, but I think you should be listening to *that* guy."

What does this class tell the reader?  First, it says "I depend on a repository."  Then it says "And when I say repository, I mean repository."  Followed by "No, seriously dude, we're talking about a repository here."  When we inject an interface like this, we pay the syntax tax by saying 'repository' **six times in a row**.  12 lines so that we could say the 1 line that we actually wanted to say.

Of course, this is not a problem with DI itself.  This is a problem of the language not knowing DI would be so prevalent.  What we really want, in these single-method dependency-injected classes, is effectively a global function, with some arguments specially-marked as dependencies.  Consider a hypothetical C# from the future:

```cs
public ListMoviesViewModel ListHandler(inject IRepository repository)
{
    return new ListMoviesViewModel { Movies = _repository.Query<Movie>() };
}
```

Such a function, if it could be declared, could be called with zero arguments when we want a real IRepository injected, and could be passed an IRepository at test time.  What do you think?  Would this kind of language support be useful in practice?
