---
title: "Streamlined Integration Tests"
layout: post
---


Last week, we saw how <a href="http://fixie.github.io/">Fixie</a> can be used to <a href="http://patrick.lioi.net/2013/06/12/dry-test-inheritance/">simplify NUnit's treatment of inheritance</a>.  This week, we'll see how to use it to streamline integration tests in systems that leverage IoC containers.

> Today's code samples work against <a href="http://nuget.org/packages/Fixie/0.0.1.62">Fixie 0.0.1.62</a>. The customization API is in its infancy, and is likely to change in the coming weeks.

## Setting the Scene

Assume, for the sake of argument, that you've already weighed the pros and cons of using IoC containers and have decided to use one in an ASP.NET MVC application.  During app startup, the IoC container is configured to provide the real implementations of your dependencies, but at test time you have the option of providing fakes.  You've also configured MVC to defer to the container whenever MVC wishes to instantiate a controller.

Voilà!  Now your controllers can declare their dependencies by simply accepting them via constructor parameters:

```cs
public class FeedbackController : Controller
{
    private readonly ISupportTicketService _support;
    private readonly ITwitter _twitter;

    public FeedbackController(ISupportTicketService support, ITwitter twitter)
    {
        _support = support;
        _twitter = twitter;
    }

    public void SubmitComplaint(ComplaintForm form)
    {
        _support.CreateTicket(form.Handle, form.Complaint);

        _twitter.SendPrivateMessage(
            form.Handle,
            "Thanks for your feedback! Our support " +
            "staff will contact you shortly.");
    }
}
```

This approach gives you something very powerful: a way to say, "I need a thing.  I don't care where it came from.  I don't even care if it's real.  Get me one so I can interact with it."  To make such a request, you simply accept an argument in the controller constructor.

## Typical Integration Testing

Now consider the testing of classes which accept such dependencies in their constructors.  In *unit* tests for such a class, you can pass in a fake implementation or a mock if the dependency is complex, resource intensive, or unreliable in a way that distracts from the behavior you are trying to test.  In *integration* tests, though, you're far more likely to want to pass in the *real* implementations of the constructor arguments, to demonstrate something closer to the true behavior you'll experience in production.

The integration test classes therefore likely call into some helper class or base class method in order to obtain a test-friendly version of the IoC container.  It might be *identical* to the IoC configuration used at runtime, so that the tests hit real databases and web services, for instance.  More likely, this test-specific IoC configuration would be *mostly* like the real thing but with a select few fakes still in the mix.  Perhaps you want to test your system's interaction with your database and internal web services, while still faking out unreliable third-party systems.

> Running your integration tests should exercise as much of the real system as is reasonably possible.  However, running your integration tests shouldn't trigger a hurricane of unintentional spam tweets to all of your CEO's followers.  Better to still use a fake ITwitter, eh?

## Leveraging IoC in Tests

**We use IoC containers to streamline the code-under-test, so why do we throw that train of thought out the window when we write the tests themselves?**

If I want to, I should be able to declare test classes whose constructors similarly accept *their* dependencies too.  I shouldn't have to explicitly call out to a test helper method to get the integration-test-friendly IoC container, just so I can bypass all the intended brevity and instead explicitly ask the container for instances of things I want to interact with!

Here's a sample test class in Fixie that leverages IoC just as much as the familiar controller does:

```cs
using System.Linq;
using Should;

namespace IntegrationTests
{
    public class FeedbackControllerTests
    {
        readonly FeedbackController _controller;
        readonly ISupportTicketRepository _tickets;

        public FeedbackControllerTests(FeedbackController controller, ISupportTicketRepository tickets)
        {
            _controller = controller;
            _tickets = tickets;
        }

        public void ShouldCreateSupportTicketWhenCustomerSubmitsComplaint()
        {
            _controller.SubmitComplaint(new ComplaintForm { Handle = "@plioi", Complaint = "Needs more cowbell." });

            var ticket = _tickets.GetAll().Single();

            ticket.Handle.ShouldEqual("@plioi");
            ticket.Complaint.ShouldEqual("Needs more cowbell.");
            ticket.Status.ShouldEqual(TicketStatus.Open);
        }
    }
}
```

We don't have to throw out all the benefits of IoC when writing tests for things that take part in IoC.  There are a few noteworthy details in this example:

**Familiar constructor injection** - The test class accepts dependencies via the constructor.  In this case, we inject an ISupportTicketRepository to inspect the effects of our controller.  We also inject the controller under test itself, allowing the IoC container to build it just like it would in the running application.

**No test base/helper class** - Just like our controllers, our test classes avoid explicitly interacting with the IoC container.  Instead, they focus on the behavior being tested.

**Fakes when you wanted them** - The intent of these tests involves hitting a real web service backed by a real database in our test environment, while avoiding real-world tweeting.

## Take Control of Test Fixture Construction

Clearly, we've missed a step.  How could a test framework possibly know which container to use, or how to configure it for real web services but fake tweeting?  The answer lies in our Fixie convention (assume that IoCContainer is some newfangled third-party IoC container, similar to StructureMap):

```cs
using System;
using Fixie;
using Fixie.Conventions;

namespace IntegrationTests
{
    public class IntegrationTestConvention : Convention
    {
        readonly IoCContainer container;

        public IntegrationTestConvention()
        {
            container = InitContainerForIntegrationTests();

            Fixtures
                .Where(type => type.IsInNamespace("IntegrationTests"))
                .NameEndsWith("Tests");

            Cases
                .Where(method => method.Void())
                .ZeroParameters();

            FixtureExecution
                .CreateInstancePerFixture(UsingContainer);
        }

        static IoCContainer InitContainerForIntegrationTests()
        {
            //Prepare the IoC Container for integration testing:
            //real implementations where possible, and fakes
            //where necessary.

            var container = new IoCContainer();

            container.Add(typeof(ISupportTicketService), new RealSupportTicketService());
            container.Add(typeof(ITwitter), new FakeTwitter());
            container.Add(typeof(ISupportTicketRepository), new RealSupportTicketRepository());

            return container;
        }

        ExceptionList UsingContainer(Type fixtureclass, out object instance)
        {
            //Teach Fixie to use the IoC container
            //in order to obtain an instance of a
            //test fixture class.

            instance = container.Get(fixtureclass);

            // Return an empty list of exceptions
            // to indicate success.
            // (This bit is still awkward!)

            return new ExceptionList();
        }
    }
}
```

With this convention class in place, all of our integration tests can accept arguments through their constructors, and they will do so with a container that contains mostly-real things, plus a few fakes.  With this approach, we get to "wear the IoC hat" in our test code as well as our code-under-test.
