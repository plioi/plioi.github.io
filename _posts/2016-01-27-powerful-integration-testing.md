---
title: Powerful Integration Testing
layout: post
---
The [Fixie test framework](https://fixie.github.io/) has been in production use for over a year now, and I've had a chance to work with it on a number of real world projects as well as a large project developed for training purposes. In the last few months, I've refined my recommended integration testing strategy in light of what I've learned. Today, we'll see how to structure your tests so that they can adhere to a few driving principles:

  * Matching production as closely as possible.
  * Helping our teammates fall into the pit of success.
  * Brevity.

We'll integrate Fixie with AutoMapper, Respawn, StructureMap, FluentValidation, Mediatr, AutoFixture, and Entity Framework to see how it all fits together for a realistic testing experience. Fair warning: by being realistic, this example will be fairly _long_, but seeing it all really work together is kind of the point.

## The Sample Application

Imagine a typical ASP.NET MVC application for maintaining a contact list.

I will assume some familiarity with Mediatr, which keeps our controller tiny. If you haven't seen it before, think of Mediatr as a bit of reflection that allows controller actions to be short: "Please, somebody, handle this request!" Mediatr finds the corresponding "Handler" class and invokes it to do the work.

Why talk about Mediatr in an article on testing? By letting us pull the meat of a feature out of the controller and into separate classes, we wind up with classes that are easier to construct and invoke from a test than an ASP.NET MVC controller would be. No HttpContext, no insane mocking. In other words, Mediatr enables the meat of an action to be pulled out to the side to be tested in isolation, and lets the controller focus entirely on routing concerns.

Here's our sample domain:

```cs
namespace ContactList.Core.Domain
{
    using System;

    public abstract class Entity
    {
        public Guid Id { get; set; }
    }

    public class Contact : Entity
    {
        public string Name { get; set; }
        public string Email { get; set; }
        public string Phone { get; set; }
    }
}
```

&#8230;and our controller:

```cs
namespace ContactList.Features.Contact
{
    using System.Web.Mvc;
    using MediatR;

    public class ContactController : Controller
    {
        private readonly IMediator _mediator;

        public ContactController(IMediator mediator)
        {
            _mediator = mediator;
        }

        //other actions...

        public ActionResult Edit(ContactEdit.Query query)
        {
            var model = _mediator.Send(query);

            return View(model);
        }

        [HttpPost]
        public ActionResult Edit(ContactEdit.Command command)
        {
            if (ModelState.IsValid)
            {
                _mediator.Send(command);

                return RedirectToAction("Index");
            }

            return View(command);
        }
    }
}
```

We'll also pull a funny trick so that most of the "Contact Edit" feature can go into a single file. Instead of having many similarly named files like ContactEditQuery, ContactEditQueryHandler, ContactEditCommand, ContactEditCommandHandler&#8230; we'll introduce one wrapper class named after the feature, ContactEdit, and place short-named items within it, each named after their role:

```cs
namespace ContactList.Features.Contact
{
    using System;
    using AutoMapper;
    using ContactLists.Core;
    using Core.Domain;
    using FluentValidation;
    using MediatR;

    public class ContactEdit
    {
        public class Query : IRequest<Command>
        {
            public Guid Id { get; set; }
        }

        public class QueryHandler : IRequestHandler<Query, Command>
        {
            private readonly ContactsContext _database;

            public QueryHandler(ContactsContext database)
            {
                _database = database;
            }

            public Command Handle(Query message)
            {
                var contact = _database.Contacts.Find(message.Id);

                return Mapper.Map<Contact, Command>(contact);
            }
        }

        public class Command : IRequest
        {
            public Guid Id { get; set; }

            public string Name { get; set; }
            public string Email { get; set; }
            public string Phone { get; set; }
        }

        public class Validator : AbstractValidator<Command>
        {
            public Validator()
            {
                RuleFor(x => x.Name).NotEmpty();
                RuleFor(x => x.Email).EmailAddress();

                RuleFor(x => x.Name).Length(1, 255);
                RuleFor(x => x.Email).Length(1, 255);
                RuleFor(x => x.Phone).Length(1, 50);
            }
        }

        public class CommandHandler : RequestHandler<Command>
        {
            private readonly ContactsContext _database;

            public CommandHandler(ContactsContext database)
            {
                _database = database;
            }

            protected override void HandleCore(Command message)
            {
                var contact = _database.Contacts.Find(message.Id);

                Mapper.Map(message, contact);
            }
        }
    }
}
```

Where does our Entity Framework DbContext subclass, ContactsContext, get saved? So that we never have to think about it again, we'll establish a Unit of Work per web request with a globally-applied filter attribute:

```cs
public class UnitOfWork : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext filterContext)
    {
        var database = DependencyResolver.Current.GetService<ContactsContext>();

        database.BeginTransaction();
    }

    public override void OnActionExecuted(ActionExecutedContext filterContext)
    {
        var database = DependencyResolver.Current.GetService<ContactsContext>();

        database.CloseTransaction(filterContext.Exception);
    }
}
```

DbContext doesn't provide these convenient BeginTransaction() / CloseTransaction(Exception) methods: they're custom. We need to deal with the web request throwing an exception before the end of the request, as well as the case that the request succeeds to this point and _then_ fails during SaveChanges(), committing only when all of that actually works:

```cs
public class ContactsContext : DbContext
{
    private DbContextTransaction _currentTransaction;

    ...

    public void BeginTransaction()
    {
        if (_currentTransaction != null)
            return;

        _currentTransaction = Database.BeginTransaction(IsolationLevel.ReadCommitted);
    }

    public void CloseTransaction()
    {
        CloseTransaction(exception: null);
    }

    public void CloseTransaction(Exception exception)
    {
        try
        {
            if (_currentTransaction != null && exception != null)
            {
                _currentTransaction.Rollback();
                return;
            }

            SaveChanges();

            _currentTransaction?.Commit();
        }
        catch (Exception)
        {
            if (_currentTransaction?.UnderlyingTransaction.Connection != null)
                _currentTransaction.Rollback();

            throw;
        }
        finally
        {
            _currentTransaction?.Dispose();
            _currentTransaction = null;
        }
    }
}
```

Assume, as well, that we're using StructureMap as our IoC container, and that in order to get one nested container per web request, we leverage a package like [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) to handle that challenging setup for us.

> To sum up, the application has one nested IoC container per web request, one transaction per web request, and one DbContext per web request. We see code defining our form's model, validation rules for that model, and handlers that actually do the work of fetching and saving a contact. Now, we're ready to test the Contact Edit feature.

## The Testing Convention

We customize Fixie by adding a Convention subclass to our project.

```cs
public class TestingConvention : Convention
{
    public TestingConvention()
    {
        Classes
            .NameEndsWith("Tests");

        Methods
            .Where(method => method.IsVoid() || method.IsAsync());

        ClassExecution
            .Wrap<InitializeAutoMapper>();

        CaseExecution
            .Wrap<ResetDatabase>()
            .Wrap<NestedContainerPerCase>();

        Parameters
            .Add<AutoFixtureParameterSource>();
    }
}
```

Each of the classes this references, InitializeAutoMapper, ResetDatabase, NestedContainerPerCase, and AutoFixtureParameterSource, are custom classes included in the test project. We'll see them each in detail later.

At a glance though, we can describe our testing style to a new team member by scanning this class:

> A test class is a class whose name ends with "Tests". A test method is any public void or async method within such a class. Whenever a test class runs, we'll ensure AutoMapper has already been initialized. Whenever a test case runs, we'll first reset the contents of the database, and we'll wrap the whole test case in a nested IoC container. When test methods have parameters, they'll be created and filled by AutoFixture.

## AutoMapper

When using [AutoMapper](https://www.nuget.org/packages/AutoMapper), to make your property mapping code error- and future-proof, you want to ensure that it is initialized _once_ and that it will automatically enlist any AutoMapper profile classes. Here's a wrapper for AutoMapper's own initialization code. I'd include this in any web application and invoke it during application startup:

```cs
public class AutoMapperBootstrapper
{
    private static readonly Lazy<AutoMapperBootstrapper> Bootstrapper = new Lazy<AutoMapperBootstrapper>(InternalInitialize);

    public static void Initialize()
    {
        var bootstrapper = Bootstrapper.Value;
    }

    private AutoMapperBootstrapper()
    {
    }

    private static AutoMapperBootstrapper InternalInitialize()
    {
        var profiles = typeof(AutoMapperBootstrapper)
            .Assembly
            .GetTypes()
            .Where(type => type.IsSubclassOf(typeof(Profile)))
            .Select(Activator.CreateInstance)
            .Cast<Profile>()
            .ToArray();

        Mapper.Initialize(cfg =>
        {
            foreach (var profile in profiles)
                cfg.AddProfile(profile);

            cfg.Seal();
        });

        return new AutoMapperBootstrapper();
    }
}
```

Here, we're playing a few tricks to ensure that AutoMapper definitely only gets initialized _once_. Our production code needs to use this to initialize AutoMapper early in its life.

In order to match production as closely as possible during our tests, we ought to execute the same code at test time, very early in the life of any particular test class execution. The first thing our testing convention needs, then, is a definition for how to ensure AutoMapper gets initialized before any test class runs. We saw the rule mentioned earlier, and here is its implementation, which we drop into the test project near the TestingConvention:

```cs
public class InitializeAutoMapper : ClassBehavior
{
    public void Execute(Class context, Action next)
    {
        AutoMapperBootstrapper.Initialize();
        next();
    }
}
```

In other words,

> Whenever we're about to run a test class, call AutoMapperBootstrapper.Initialize() first, and then proceed with actually running the tests in that test class.

Since we already ensured that AutoMapper.Initialize() will only ever really initialize things once, there's no real cost due to invoking this once per test class. We just needed to invoke the bootstrapper at least once very early in the life of each test.

Additionally, we'll add a test that will fail if any of our custom AutoMapper rules don't make sense:

```cs
public class AutoMapperTests
{
    public void ShouldHaveValidConfiguration()
    {
        Mapper.AssertConfigurationIsValid();
    }
}
```

## Respawn

It's important that integration tests be independent, and when your tests hit a database that means we need to start each test from a well known state. The simplest well-known state is _empty_, and that's where [Respawn](https://www.nuget.org/packages/Respawn) comes into the picture. Respawn can nuke every row from our database prior to each test. The only records that exist are the ones our test puts there.

Our convention claims to reset the database with every test case. We saw the rule mentioned earlier, and here is its implementation:

```cs
public class ResetDatabase : CaseBehavior
{
    public void Execute(Case context, Action next)
    {
        var checkpoint = new Checkpoint
        {
            SchemasToExclude = new[] { "RoundhousE" },
            TablesToIgnore = new[] { "sysdiagrams" }
        };

        checkpoint.Reset(ContactsContext.ConnectionString);

        next();
    }
}
```

In other words,

> Whenever we're about to run a test case, call Respawn's Reset(&#8230;) first, and then proceed with actually running the test case.

## StructureMap

Earlier, I said that I apply the package [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) to a web application to help get the ball rolling on integrating StructureMap with MVC. I like to customize the code that package places in my system before proceeding. As with AutoMapper, I want the production code to include a class that simply initializes StructureMap _exactly once_ at application startup:

```cs
public static class IoC
{
    private static readonly Lazy<IContainer> Bootstrapper = new Lazy<IContainer>(Initialize, true);

    public static IContainer Container => Bootstrapper.Value;

    private static IContainer Initialize()
    {
        return new Container(cfg =>
        {
            cfg.Scan(scan =>
            {
                scan.TheCallingAssembly();
                scan.LookForRegistries();
                scan.WithDefaultConventions();
                scan.With(new ControllerConvention());
            });
        });
    }
}
```

I claimed that installing [StructureMap.MVC5](https://nuget.org/packages/StructureMap.MVC5) will set up "one nested IoC container per web request" in our production code. Each web request will get its own little bubble of dependency creation. For instance, this gives me exactly one DbContext per web request, which satisfies our Unit of Work pattern.

I want my tests to mimic production as much as possible, so I similarly want one nested IoC container per test case. A test case mimics one user interaction, so it better run in the same kind of environment as one actual user interaction!

Our convention claims to set up one nested container per test case. We saw the rule mentioned earlier, and here is its implementation:

```cs
public class NestedContainerPerCase : CaseBehavior
{
    public void Execute(Case context, Action next)
    {
        TestDependencyScope.Begin();
        next();
        TestDependencyScope.End();
    }
}

public static class TestDependencyScope
{
    private static IContainer _currentNestedContainer;

    public static void Begin()
    {
        if (_currentNestedContainer != null)
            throw new Exception("Cannot begin test dependency scope. Another dependency scope is still in effect.");

        _currentNestedContainer = IoC.Container.GetNestedContainer();
    }

    public static IContainer CurrentNestedContainer
    {
        get
        {
            if (_currentNestedContainer == null)
                throw new Exception($"Cannot access the {nameof(CurrentNestedContainer)}. There is no dependency scope in effect.");

            return _currentNestedContainer;
        }
    }

    public static void End()
    {
        if (_currentNestedContainer == null)
            throw new Exception("Cannot end test dependency scope. There is no dependency scope in effect.");

        _currentNestedContainer.Dispose();
        _currentNestedContainer = null;
    }
}
```

In other words,

> Wrap each test case in a new nested IoC container dedicated to that test case, just like the one that each web request has in production.

Note that this works by invoking the IoC class. In other words, every single test case exercises the same IoC setup that we're applying in production. Again, the use of Lazy<T> saves us from actually being wasteful about it.

## AutoFixture

Within a test, we often need to construct a sample object and fill in its properties with fake data. Doing so explicitly is annoying and fails to be future-proof, so we can defer to [AutoFixture](https://www.nuget.org/packages/AutoFixture) to construct-and-fill our dummy test objects for us.

We _could_ invoke this tool explicitly within our tests, but we can do better. Our convention claims that test method parameters come from AutoFixture. We saw the rule mentioned earlier, and here is its implementation:

```cs
public class AutoFixtureParameterSource : ParameterSource
{
    public IEnumerable<object[]> GetParameters(MethodInfo method)
    {
        // Produces a randomly-populated object for each
        // parameter declared on the test method, using
        // a Fixture that has our customizations.

        var fixture = new Fixture();

        CustomizeAutoFixture(fixture);

        var specimenContext = new SpecimenContext(fixture);

        var allEntitiesAttribute =
           method.GetCustomAttributes<AllEntities>(true).SingleOrDefault();

        if (allEntitiesAttribute != null)
        {
            return typeof(Entity).Assembly.GetTypes()
                .Where(t => t.IsSubclassOf(typeof(Entity)))
                .Where(t => !t.IsAbstract)
                .Except(allEntitiesAttribute.Except)
                .Select(entityType => new[] { specimenContext.Resolve(entityType) })
                .ToArray();
        }

        var parameterTypes = method.GetParameters().Select(x => x.ParameterType);

        var arguments = parameterTypes.Select(specimenContext.Resolve).ToArray();

        return new[] { arguments };
    }

    private static void CustomizeAutoFixture(Fixture fixture)
    {
        var propertyBuilders = typeof(PropertyBuilder)
            .Assembly
            .GetTypes()
            .Where(t => !t.IsAbstract && typeof(ISpecimenBuilder).IsAssignableFrom(t))
            .Select(Activator.CreateInstance)
            .Cast<ISpecimenBuilder>();

        foreach (var propertyBuilder in propertyBuilders)
            fixture.Customizations.Add(propertyBuilder);
    }
}

[AttributeUsage(AttributeTargets.Method)]
class AllEntities : Attribute
{
    public AllEntities()
    {
        Except = new Type[] { };
    }

    public Type[] Except { get; set; }
}
```

We'll see this AllEntities attribute come into the picture a bit later. At this point, we can focus on the primary purpose of the AutoFixtureParameterSource. In other words,

> Whenever at test case has input parameters, the parameters will be constructed and filled with fake values using AutoFixture, including any AutoFixture customizations found in the test project.

## FluentValidation

We saw a [FluentValidation](https://www.nuget.org/packages/FluentValidation) validator class earlier as a part of our Edit feature. For tests, I like to include a few extension methods so that we can make expressive assertions about validation rules:

```cs
using System;
using System.Linq;
using MediatR;
using Should;
using static Testing;

public static class Assertions
{
    public static void ShouldValidate<TResponse>(this IRequest<TResponse> message)
    {
        var validator = Validator(message);

        validator.ShouldNotBeNull($"There is no validator for {message.GetType()} messages");

        var result = validator.Validate(message);

        var indentedErrorMessages = result
            .Errors
            .OrderBy(x => x.ErrorMessage)
            .Select(x => "    " + x.ErrorMessage)
            .ToArray();

        var actual = String.Join(Environment.NewLine, indentedErrorMessages);

        result.IsValid.ShouldBeTrue(
            $"Expected no validation errors, but found {result.Errors.Count}:{Environment.NewLine}{actual}");
    }

    public static void ShouldNotValidate<TResponse>(this IRequest<TResponse> message, params string[] expectedErrors)
    {
        var validator = Validator(message);

        validator.ShouldNotBeNull($"There is no validator for {message.GetType()} messages");

        var result = validator.Validate(message);

        result.IsValid.ShouldBeFalse("Expected validation errors, but the message passed validation.");

        var actual = result.Errors
            .OrderBy(x => x.ErrorMessage)
            .Select(x => x.ErrorMessage)
            .ToArray();

        actual.ShouldEqual(expectedErrors.OrderBy(x => x).ToArray());
    }
}
```

FluentValidation has a few built-in assertion helpers, but in my experience they make it far too easy to write a test that is _green_ even while the validation rule under test is _wildly wrong_. Our own assertion helpers make it clear: with this sample form submission, we get exactly these expected failure messages.

## Create a Testing DSL

C# 6 includes a feature in which the static methods of a class can be imported to a code file. A using directive can now take on the form _using static Some.Static.Class.Name;_

When you use such a using directive, your code file gets to call the static members of that class without having to prefix them with the class name. For our tests, we'll take advantage of this new syntax to define a little [Domain Specific Language](https://en.wikipedia.org/wiki/Domain-specific_language). In our test project, near the TestingConvention, we'll add a static class of helper methods. Note how these greatly leverage the infrastructure we've already set up:

```cs
public static class Testing
{
    private static IContainer Container => TestDependencyScope.CurrentNestedContainer;

    public static T Resolve<T>()
    {
        return Container.GetInstance<T>();
    }

    public static object Resolve(Type type)
    {
        return Container.GetInstance(type);
    }

    public static void Inject<T>(T instance) where T : class
    {
        Container.Inject(instance);
    }

    public static void LogSql()
    {
        Resolve<ContactsContext>().Database.Log = Console.Write;
    }

    public static void Transaction(Action<ContactsContext> action)
    {
        using (var database = new ContactsContext())
        {
            try
            {
                database.BeginTransaction();
                action(database);
                database.CloseTransaction();
            }
            catch (Exception exception)
            {
                database.CloseTransaction(exception);
                throw;
            }
        }
    }

    public static void Save(params object[] entities)
    {
        Resolve<SaveAtMostOncePolicy>().Enforce();

        Transaction(database =>
        {
            foreach (var entity in entities)
                database.Set(entity.GetType()).Add(entity);
        });
    }

    private class SaveAtMostOncePolicy
    {
        private bool _hasAlreadySaved;

        public void Enforce()
        {
            if (_hasAlreadySaved)
                throw new InvalidOperationException(
                    "A test should call Save(...) at most once. Otherwise, " +
                    "it is likely that duplicate records will be created, as each " +
                    "call to this method uses a distinct DbContext.");

            _hasAlreadySaved = true;
        }
    }

    public static TResult Query<TResult>(Func<ContactsContext, TResult> query)
    {
        var result = default(TResult);

        Transaction(database =>
        {
            result = query(database);
        });

        return result;
    }

    public static IValidator Validator<TResult>(IRequest<TResult> message)
    {
        var validatorType = typeof(IValidator<>).MakeGenericType(message.GetType());

        return Container.TryGetInstance(validatorType) as IValidator;
    }

    public static void Send(IRequest message)
    {
        Send((IRequest<Unit>)message);
    }

    public static TResult Send<TResult>(IRequest<TResult> message)
    {
        var validator = Validator(message);

        if (validator != null)
            message.ShouldValidate();

        TResult result;

        var database = Resolve<ContactsContext>();
        try
        {
            database.BeginTransaction();
            result = Resolve<IMediator>().Send(message);
            database.CloseTransaction();
        }
        catch (Exception exception)
        {
            database.CloseTransaction(exception);
            throw;
        }

        return result;
    }
}
```

We've got a lot going on here.

First, we can interact with the one nested IoC container per test case, in order to resolve types in our tests. If one of our tests needs to swap in a fake implementation of some interface, it can call Inject(&#8230;).

Second, if we ever want a test to temporarily dump the generated Entity Framework SQL to the console, we can call LogSql() inside that test.

Third, we have some helpers for working against the database. Transaction(&#8230;) lets you interact with the database in a dedicated transaction. Save(&#8230;) lets you initialize your recently-respawned database with a few entities. Save(&#8230;) also protects your fellow teammates from mistakenly abusing the DbContext involved by requiring that you save all your sample entities with a single DbContext. Query(&#8230;) lets you inspect your database during assertions after exercising the system under test.

Fourth, Validator(&#8230;) lets us get a handle on the same validator instance that would be used prior to entering a controller action. We want a test case to mimic a controller action. If our validation rule tests construct a validator explicitly, they would be missing out on the chance to catch poor validator declarations that reference the wrong model. This helper method lets us exercise the same validator lookup that would happen right before a controller action gets invoked in production.

Lastly, we integrate with Mediatr in our tests with the Send(&#8230;) helper. Our tests can send a message through Mediatr just like our controller actions can, and when our tests do so they operate in their own Unit of Work just like production. Additionally, our test will only be allowed to Send(&#8230;) when the sample form would have passed validation, which keeps us from writing misleading tests for scenarios that would never happen in production.

## Actually Write A Test, Darnit!

Enough infrastructure, let's write some tests! First, I want to be confident that when the user goes to edit a Contact, they'll see the form with the right values populated for that selected Contact:

```cs
namespace ContactList.Tests.Features
{
    using Core.Domain;
    using ContactList.Features.Contact;
    using static Testing;
    using Should;

    public class ContactEditTests
    {
        public void ShouldDisplaySelectedContact(Contact contactToEdit, Contact anotherContact)
        {
            Save(contactToEdit, anotherContact);

            var selectedContactId = contactToEdit.Id;

            var result = Send(new ContactEdit.Query { Id = selectedContactId });

            result.Id.ShouldEqual(selectedContactId);
            result.Name.ShouldEqual(contactToEdit.Name);
            result.Email.ShouldEqual(contactToEdit.Email);
            result.Phone.ShouldEqual(contactToEdit.Phone);
        }

        ...

    }
}
```

Recall what all is actually going on here.

Respawn steps in to nuke any existing database records. We're starting this test from a clean slate every time it runs.

AutoFixture steps in to fully populate our incoming Contact instances with essentially random data. I don't care what the property values are. I only care that they're filled in with something.

We set up the well-known state of our database with the Save(&#8230;) helper. This works in its own DbContext and its own transaction, since this is just setup code rather than the system under test.

We execute the system under test by calling Send(&#8230;), passing in the same request object that would be used in production to select a Contact for editing. This operates within the nested container mimicking production, in a database transaction mimicking production. We're not just calling the handler's Execute(&#8230;) method. We're exercising the whole pipeline that our controller action would execute in production.

Finally, we just assert that the view model we got back is fully populated with the _right_ Contact's information. Since we saved two Contacts and fetched one, we have confidence that our query actually works.

## Testing Validation Rules

Most validation rule tests out there are _horrifically_ useless. They say things like, "With this sample form, the such and such property should report some error of some kind." Such a test _seems_ to be testing something, but it's so vague that you wind up being able to get a passing test even when everything is buggy.

Instead, let's actually assert that the validation rule fails for the reason we think it is set up to fail, by asserting on the error message too!

```cs
public class ContactEditTests
{
    ...

    public void ShouldRequireMinimumFields()
    {
        new ContactEdit.Command()
            .ShouldNotValidate("'Name' should not be empty.");
    }

    public void ShouldRequireValidEmailWhenProvided(ContactEdit.Command command)
    {
        command.Email = null;
        command.ShouldValidate();

        command.Email = "test@example.com";
        command.ShouldValidate();

        command.Email = "test at example dot com";
        command.ShouldNotValidate("'Email' is not a valid email address.");
    }

    ...
}
```

"Oh, but that's brittle!" you say? Without it, your validation rule tests are such a misleading time bomb that I'd rather you not write them at all, thankyouverymuch.

## Yet More Testing

We still need to test that our user can save changes when submitting their form:

```cs
public class ContactEditTests
{
    ...
    
    public void ShouldSaveChangesToSelectedContact(Contact contactToEdit, Contact anotherContact)
    {
        Save(contactToEdit, anotherContact);
        
        var selectedContactId = contactToEdit.Id;

        Send(new ContactEdit.Command
        {
            Id = selectedContactId,
            Name = "John Smith",
            Email = "jsmith@example.com",
            Phone = "555-123-0000"
        });

        var actual = Query(db => db.Contacts.Find(selectedContactId));

        actual.Id.ShouldEqual(selectedContactId);
        actual.Name.ShouldEqual("John Smith");
        actual.Email.ShouldEqual("jsmith@example.com");
        actual.Phone.ShouldEqual("555-123-0000");
    }
}
```

Again, we're not just testing the handler's Execute(&#8230;) method. We're working in a fresh database, with automatically populated sample records, within a production-like nested IoC container and a production-like Unit of Work. Send(&#8230;) will fail the test if the sample form wouldn't really have passed validation in production, so we have confidence that the scenario actually makes sense. Our assertion rightly uses its _own_ transaction so that we don't fool ourselves by misusing Entity Framework change tracking: we assert on the reality of the operation's effects on the world. Lastly, we've demonstrated that we're affecting the _right_ record. It would be _very_ difficult for this test to pass incorrectly.

## Automatic Persistence Testing

One last thing. We saw a strange attribute, AllEntities, referenced within our AutoFixture parameter customization earlier. I use it in one special test:

```cs
using System.Linq;
using ContactList.Core.Domain;
using ContactList.Tests;
using static Testing;

public class EntityTests
{
    [AllEntities]
    public void ShouldPersist<TEntity>(TEntity entity) where TEntity : Entity
    {
        Save(entity);

        Transaction(db =>
        {
            var loaded = db.Set<TEntity>().Single();

            loaded.ShouldMatch(entity);
        });
    }
}
```

Despite it's size, a lot is happening here.

This test method is called once for every single Entity subclass in the system. Every entity gets its own individual pass or fail.

For each entity, we get a test that attempts to fully populate, save, and reload a record, asserting that every single property on it "round tripped" without loss to and from the database.

The ShouldMatch assertion helper just takes two objects of the same type, and asserts that they have the same JSON representation, giving us a quick way to deeply compare all the properties for equality:

```cs
public static class Assertions
{
    public static void ShouldMatch<T>(this T actual, T expected)
        => Json(actual).ShouldEqual(Json(expected), "Expected two objects to match on all properties.");

    private static string Json<T>(T actual)
        => JsonConvert.SerializeObject(actual, Formatting.Indented);

    ...
}
```

Imagine the effect of having this test in your project. You embark on a new feature that needs a new table. You add an Entity subclass and run your build. This test fails, telling you the table doesn't exist yet. You add a migration script to create the table and run your build. This test fails, telling you that you have a typo in a property name. You fix it and run your build. This test passes. You can reliably save and load the new entity. Then you start to write your actual feature with its own tests. You and your teammates, _cannot forget_ to do the right thing at each step.

## Expressive Testing

All in all, this approach to integration testing leaves me with a great deal of confidence in the system under test, with very little code having to appear in each test. A new team member cannot forget to start their tests from a clean slate, they cannot forget that they need to avoid misusing DbContext when setting up sample records, they don't have to come up with silly random values for properties, they cannot forget to exercise the full IoC container/transaction/validation/Mediatr pipeline, they cannot forget to test that their entities actually persist, and their resulting tests are clear and concise, telling a story about how the feature should behave.