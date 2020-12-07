---
title: Powerful Integration Testing
layout: post
---
The [Fixie test framework](https://fixie.github.io/) has been in production use for over a year now, and I&#8217;ve had a chance to work with it on a number of real world projects as well as a large project developed for training purposes. In the last few months, I&#8217;ve refined my recommended integration testing strategy in light of what I&#8217;ve learned. Today, we&#8217;ll see how to structure your tests so that they can adhere to a few driving principles:

  * Matching production as closely as possible.
  * Helping our teammates fall into the pit of success.
  * Brevity.

We&#8217;ll integrate Fixie with AutoMapper, Respawn, StructureMap, FluentValidation, Mediatr, AutoFixture, and Entity Framework to see how it all fits together for a realistic testing experience. Fair warning: by being realistic, this example will be fairly _long_, but seeing it all really work together is kind of the point.

## The Sample Application

Imagine a typical ASP.NET MVC application for maintaining a contact list.

I will assume some familiarity with Mediatr, which keeps our controller tiny. If you haven&#8217;t seen it before, think of Mediatr as a bit of reflection that allows controller actions to be short: &#8220;Please, somebody, handle this request!&#8221; Mediatr finds the corresponding &#8220;Handler&#8221; class and invokes it to do the work.

Why talk about Mediatr in an article on testing? By letting us pull the meat of a feature out of the controller and into separate classes, we wind up with classes that are easier to construct and invoke from a test than an ASP.NET MVC controller would be. No HttpContext, no insane mocking. In other words, Mediatr enables the meat of an action to be pulled out to the side to be tested in isolation, and lets the controller focus entirely on routing concerns.

Here&#8217;s our sample domain:

{% gist b15b1a6df8d7c5c0cf51 %}

&#8230;and our controller:

{% gist 3feaf52304df5c9aeea1 %}

We&#8217;ll also pull a funny trick so that most of the &#8220;Contact Edit&#8221; feature can go into a single file. Instead of having many similarly named files like ContactEditQuery, ContactEditQueryHandler, ContactEditCommand, ContactEditCommandHandler&#8230; we&#8217;ll introduce one wrapper class named after the feature, ContactEdit, and place short-named items within it, each named after their role:

{% gist 9e51f1e86a2f5b552694 %}

Where does our Entity Framework DbContext subclass, ContactsContext, get saved? So that we never have to think about it again, we&#8217;ll establish a Unit of Work per web request with a globally-applied filter attribute:

{% gist 07101c973abf2676703b %}

DbContext doesn&#8217;t provide these convenient BeginTransaction() / CloseTransaction(Exception) methods: they&#8217;re custom. We need to deal with the web request throwing an exception before the end of the request, as well as the case that the request succeeds to this point and _then_ fails during SaveChanges(), committing only when all of that actually works:

{% gist dbf6cab74aac2c6ee767 %}

Assume, as well, that we&#8217;re using StructureMap as our IoC container, and that in order to get one nested container per web request, we leverage a package like [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) to handle that challenging setup for us.

> To sum up, the application has one nested IoC container per web request, one transaction per web request, and one DbContext per web request. We see code defining our form&#8217;s model, validation rules for that model, and handlers that actually do the work of fetching and saving a contact. Now, we&#8217;re ready to test the Contact Edit feature.

## The Testing Convention

We customize Fixie by adding a Convention subclass to our project.

{% gist 1e62c7be7a81af65ef64 %}

Each of the classes this references, InitializeAutoMapper, ResetDatabase, NestedContainerPerCase, and AutoFixtureParameterSource, are custom classes included in the test project. We&#8217;ll see them each in detail later.

At a glance though, we can describe our testing style to a new team member by scanning this class:

> A test class is a class whose name ends with &#8220;Tests&#8221;. A test method is any public void or async method within such a class. Whenever a test class runs, we&#8217;ll ensure AutoMapper has already been initialized. Whenever a test case runs, we&#8217;ll first reset the contents of the database, and we&#8217;ll wrap the whole test case in a nested IoC container. When test methods have parameters, they&#8217;ll be created and filled by AutoFixture.

## AutoMapper

When using [AutoMapper](https://www.nuget.org/packages/AutoMapper), to make your property mapping code error- and future-proof, you want to ensure that it is initialized _once_ and that it will automatically enlist any AutoMapper profile classes. Here&#8217;s a wrapper for AutoMapper&#8217;s own initialization code. I&#8217;d include this in any web application and invoke it during application startup:

{% gist 27af636fec78792dfbb2 %}

Here, we&#8217;re playing a few tricks to ensure that AutoMapper definitely only gets initialized _once_. Our production code needs to use this to initialize AutoMapper early in its life.

In order to match production as closely as possible during our tests, we ought to execute the same code at test time, very early in the life of any particular test class execution. The first thing our testing convention needs, then, is a definition for how to ensure AutoMapper gets initialized before any test class runs. We saw the rule mentioned earlier, and here is its implementation, which we drop into the test project near the TestingConvention:

{% gist 3cfc74efb5118afdd373 %}

In other words,

> Whenever we&#8217;re about to run a test class, call AutoMapperBootstrapper.Initialize() first, and then proceed with actually running the tests in that test class.

Since we already ensured that AutoMapper.Initialize() will only ever really initialize things once, there&#8217;s no real cost due to invoking this once per test class. We just needed to invoke the bootstrapper at least once very early in the life of each test.

Additionally, we&#8217;ll add a test that will fail if any of our custom AutoMapper rules don&#8217;t make sense:

{% gist 73d036a4fcb8eeefb72a %}

## Respawn

It&#8217;s important that integration tests be independent, and when your tests hit a database that means we need to start each test from a well known state. The simplest well-known state is _empty_, and that&#8217;s where [Respawn](https://www.nuget.org/packages/Respawn) comes into the picture. Respawn can nuke every row from our database prior to each test. The only records that exist are the ones our test puts there.

Our convention claims to reset the database with every test case. We saw the rule mentioned earlier, and here is its implementation:

{% gist 643e4378205f23c6a826 %}

In other words,

> Whenever we&#8217;re about to run a test case, call Respawn&#8217;s Reset(&#8230;) first, and then proceed with actually running the test case.

## StructureMap

Earlier, I said that I apply the package [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) to a web application to help get the ball rolling on integrating StructureMap with MVC. I like to customize the code that package places in my system before proceeding. As with AutoMapper, I want the production code to include a class that simply initializes StructureMap _exactly once_ at application startup:

{% gist 0b318fd56f9ab2b0b0e2 %}

I claimed that installing [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) will set up &#8220;one nested IoC container per web request&#8221; in our production code. Each web request will get its own little bubble of dependency creation. For instance, this gives me exactly one DbContext per web request, which satisfies our Unit of Work pattern.

I want my tests to mimic production as much as possible, so I similarly want one nested IoC container per test case. A test case mimics one user interaction, so it better run in the same kind of environment as one actual user interaction!

Our convention claims to set up one nested container per test case. We saw the rule mentioned earlier, and here is its implementation:

{% gist 3ee4aaac59dad90e2ff4 %}

In other words,

> Wrap each test case in a new nested IoC container dedicated to that test case, just like the one that each web request has in production.

Note that this works by invoking the IoC class. In other words, every single test case exercises the same IoC setup that we&#8217;re applying in production. Again, the use of Lazy<T> saves us from actually being wasteful about it.

## AutoFixture

Within a test, we often need to construct a sample object and fill in its properties with fake data. Doing so explicitly is annoying and fails to be future-proof, so we can defer to [AutoFixture](https://www.nuget.org/packages/AutoFixture) to construct-and-fill our dummy test objects for us.

We _could_ invoke this tool explicitly within our tests, but we can do better. Our convention claims that test method parameters come from AutoFixture. We saw the rule mentioned earlier, and here is its implementation:

{% gist 043458bf30be1d96d884 %}

We&#8217;ll see this AllEntities attribute come into the picture a bit later. At this point, we can focus on the primary purpose of the AutoFixtureParameterSource. In other words,

> Whenever at test case has input parameters, the parameters will be constructed and filled with fake values using AutoFixture, including any AutoFixture customizations found in the test project.

## FluentValidation

We saw a [FluentValidation](https://www.nuget.org/packages/FluentValidation) validator class earlier as a part of our Edit feature. For tests, I like to include a few extension methods so that we can make expressive assertions about validation rules:

{% gist 2cb4b4255cd1b301b4c3 %}

FluentValidation has a few built-in assertion helpers, but in my experience they make it far too easy to write a test that is _green_ even while the validation rule under test is _wildly wrong_. Our own assertion helpers make it clear: with this sample form submission, we get exactly these expected failure messages.

## Create a Testing DSL

C# 6 includes a feature in which the static methods of a class can be imported to a code file. A using directive can now take on the form _using static Some.Static.Class.Name;_

When you use such a using directive, your code file gets to call the static members of that class without having to prefix them with the class name. For our tests, we&#8217;ll take advantage of this new syntax to define a little [Domain Specific Language](https://en.wikipedia.org/wiki/Domain-specific_language). In our test project, near the TestingConvention, we&#8217;ll add a static class of helper methods. Note how these greatly leverage the infrastructure we&#8217;ve already set up:

{% gist 481eaa2f474a14b6c936 %}

We&#8217;ve got a lot going on here.

First, we can interact with the one nested IoC container per test case, in order to resolve types in our tests. If one of our tests needs to swap in a fake implementation of some interface, it can call Inject(&#8230;).

Second, if we ever want a test to temporarily dump the generated Entity Framework SQL to the console, we can call LogSql() inside that test.

Third, we have some helpers for working against the database. Transaction(&#8230;) lets you interact with the database in a dedicated transaction. Save(&#8230;) lets you initialize your recently-respawned database with a few entities. Save(&#8230;) also protects your fellow teammates from mistakenly abusing the DbContext involved by requiring that you save all your sample entities with a single DbContext. Query(&#8230;) lets you inspect your database during assertions after exercising the system under test.

Fourth, Validator(&#8230;) lets us get a handle on the same validator instance that would be used prior to entering a controller action. We want a test case to mimic a controller action. If our validation rule tests construct a validator explicitly, they would be missing out on the chance to catch poor validator declarations that reference the wrong model. This helper method lets us exercise the same validator lookup that would happen right before a controller action gets invoked in production.

Lastly, we integrate with Mediatr in our tests with the Send(&#8230;) helper. Our tests can send a message through Mediatr just like our controller actions can, and when our tests do so they operate in their own Unit of Work just like production. Additionally, our test will only be allowed to Send(&#8230;) when the sample form would have passed validation, which keeps us from writing misleading tests for scenarios that would never happen in production.

## Actually Write A Test, Darnit!

Enough infrastructure, let&#8217;s write some tests! First, I want to be confident that when the user goes to edit a Contact, they&#8217;ll see the form with the right values populated for that selected Contact:

{% gist d4bf27f5774ea5f7d8a7 %}

Recall what all is actually going on here.

Respawn steps in to nuke any existing database records. We&#8217;re starting this test from a clean slate every time it runs.

AutoFixture steps in to fully populate our incoming Contact instances with essentially random data. I don&#8217;t care what the property values are. I only care that they&#8217;re filled in with something.

We set up the well-known state of our database with the Save(&#8230;) helper. This works in its own DbContext and its own transaction, since this is just setup code rather than the system under test.

We execute the system under test by calling Send(&#8230;), passing in the same request object that would be used in production to select a Contact for editing. This operates within the nested container mimicking production, in a database transaction mimicking production. We&#8217;re not just calling the handler&#8217;s Execute(&#8230;) method. We&#8217;re exercising the whole pipeline that our controller action would execute in production.

Finally, we just assert that the view model we got back is fully populated with the _right_ Contact&#8217;s information. Since we saved two Contacts and fetched one, we have confidence that our query actually works.

## Testing Validation Rules

Most validation rule tests out there are _horrifically_ useless. They say things like, &#8220;With this sample form, the such and such property should report some error of some kind.&#8221; Such a test _seems_ to be testing something, but it&#8217;s so vague that you wind up being able to get a passing test even when everything is buggy.

Instead, let&#8217;s actually assert that the validation rule fails for the reason we think it is set up to fail, by asserting on the error message too!

{% gist f0a1a59297de20b95557 %}

&#8220;Oh, but that&#8217;s brittle!&#8221; you say? Without it, your validation rule tests are such a misleading time bomb that I&#8217;d rather you not write them at all, thankyouverymuch.

## Yet More Testing

We still need to test that our user can save changes when submitting their form:

{% gist 0bb9b41a5b1005cbab70 %}

Again, we&#8217;re not just testing the handler&#8217;s Execute(&#8230;) method. We&#8217;re working in a fresh database, with automatically populated sample records, within a production-like nested IoC container and a production-like Unit of Work. Send(&#8230;) will fail the test if the sample form wouldn&#8217;t really have passed validation in production, so we have confidence that the scenario actually makes sense. Our assertion rightly uses its _own_ transaction so that we don&#8217;t fool ourselves by misusing Entity Framework change tracking: we assert on the reality of the operation&#8217;s effects on the world. Lastly, we&#8217;ve demonstrated that we&#8217;re affecting the _right_ record. It would be _very_ difficult for this test to pass incorrectly.

## Automatic Persistence Testing

One last thing. We saw a strange attribute, AllEntities, referenced within our AutoFixture parameter customization earlier. I use it in one special test:

{% gist 7ab37ebe3d7a86fc4583 %}

Despite it&#8217;s size, a lot is happening here.

This test method is called once for every single Entity subclass in the system. Every entity gets its own individual pass or fail.

For each entity, we get a test that attempts to fully populate, save, and reload a record, asserting that every single property on it &#8220;round tripped&#8221; without loss to and from the database.

The ShouldMatch assertion helper just takes two objects of the same type, and asserts that they have the same JSON representation, giving us a quick way to deeply compare all the properties for equality:

{% gist f96a7bb659afdd171dc5 %}

Imagine the effect of having this test in your project. You embark on a new feature that needs a new table. You add an Entity subclass and run your build. This test fails, telling you the table doesn&#8217;t exist yet. You add a migration script to create the table and run your build. This test fails, telling you that you have a typo in a property name. You fix it and run your build. This test passes. You can reliably save and load the new entity. Then you start to write your actual feature with its own tests. You and your teammates, _cannot forget_ to do the right thing at each step.

## Expressive Testing

All in all, this approach to integration testing leaves me with a great deal of confidence in the system under test, with very little code having to appear in each test. A new team member cannot forget to start their tests from a clean slate, they cannot forget that they need to avoid misusing DbContext when setting up sample records, they don&#8217;t have to come up with silly random values for properties, they cannot forget to exercise the full IoC container/transaction/validation/Mediatr pipeline, they cannot forget to test that their entities actually persist, and their resulting tests are clear and concise, telling a story about how the feature should behave.