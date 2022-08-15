---
title: "From WebForms to MVC"
layout: post
---


Most of my first job out of college was spent doing desktop development, some of which was in VB6.  I didn't love it, but it was easy to learn.  When *this* button is clicked, *that* function runs in response to it.  Easy.

The controls-and-event-handlers model combined with the language also made it exceptionally easy to create a tangled pile of misplaced responsibilities, but I think you've got to experience that kind of pain before you can see that alternatives even *exist*.  Once you've felt the pain, you can better-evaluate which guidance is actually going to be helpful.


## WebForms Was Designed for Desktop Developers

Right around that time, .NET was released.  ASP.NET WebForms shared little with classic ASP.  The primary goal was to let desktop VB developers make a smooth transition to both .NET and web development at the same time.  With that goal in mind, WebForms provided a similar controls-and-event-handlers model.  Making a web app felt like making a desktop app.

In order to support that approach, WebForms had to make the stateless world of HTTP *feel* stateful.  View State, the Page Life Cycle, and event handlers allowed the developer to go through the familiar motions of dropping controls onto a designer and wiring up methods to control events, *all while ignoring everything that was actually happening*.

Abstractions leak, though, and this was a *huge* abstraction.  It helped me get over the initial hurdle to get a web app up and running, but I found the code I wrote was suspiciously like the desktop VB6 mess I'd wanted to grow beyond.  I was frequently surprised by the effects of the Page Life Cycle, and every time that happened I had to learn a little bit more about how the abstraction worked under the hood:

> "Oh! There's a Big Honking `<form>` Tag for the whole page, and it contains an Enormous Encoded `<input>` Tag containing the current state of all the controls, so whenever I need to hit a button click handler I'm really submitting a normal HTML form and then..."

Ugh. Eventually, you just have to know what HTML and HTTP are, even when using WebForms, so the abstraction was no longer helping.

## Many Page Life Cycles Later...

While working for a previous employer, ASP.NET MVC came out.  We wanted to try out the new approach, and the timing coincided with a new project that was starting up.  However, there wasn't much guidance available yet on how to do it *well*.  We were challenged at first by the lack of View State, and in order to keep our old "stateful stateless web" habit going we ended up storing a *lot* of things into the user's session object.  We also kept treating controller actions like they were event handlers, leading to some awkward-looking URLs.  Distracted by these initial pains, our controllers started absorbing responsibilities like a sponge.  Actually, our *controller* started absorbing responsibilities - there was only controller in the whole app!

I was writing a lot of controller actions like this:

```cs
public ActionResult Edit()
{
    var conferenceName = Request["ConferenceName"] as string;
    var repository = new ConferenceRepository();
    var conference = repository.GetByName(conferenceName);

    if (conference == null)
        return View("ConferenceNotFound");

    ViewData["ConferenceId"] = conference.Id,
    ViewData["ConferenceName"] = conference.Name

    return View();
}

public ActionResult Save()
{
    var conferenceName = Request["ConferenceName"] as string
    if (string.IsNullOrEmpty(conferenceName))
    {
        ViewBag.ErrorMessage = "Conference Name is required";
        return View("Edit");
    }

    var repository = new ConferenceRepository();
    var conference = repository.GetById(new Guid(Request["ConferenceId"] as string));
    conference.ChangeName(Request["ConferenceName"] as string);
    _repository.Save(conference);

    return RedirectToRoute("Default", new { action = "Index"});
}
```

What I saw was the same thing I saw in WebForms development.  The Controller was the new Code Behind, and I was underwhelmed.  WebForms's Page Life Cycle and View State hurt, but the real daily pains came from working with classes that were growing out of control, and I was seeing the same thing with my MVC code.  It seemed like the abstractions were lighter-weight, and it felt right to work with HTML more directly, but I was definitely missing some guidance.

## Turning the Corner

Several coworkers got to take an MVC Boot Camp course at <a href="http://www.headspring.com">Headspring</a>, and I got to learn by osmosis from them upon their return.  At last, I had a good picture in my head of what it looks like when you've succeeded with these tools.  Our controllers started to get much more lean:

```cs
[HttpGet]
public ActionResult Edit(Conference conference)
{
    return AutoMapView<ConferenceEditModel>(View(conference));
}

[HttpPost]
public ActionResult Edit(ConferenceEditModel form)
{
    if (!ViewData.ModelState.IsValid)
        return View(form);

    var conference = _repository.GetById(form.Id);
    conference.ChangeName(form.Name);

    return this.RedirectToAction<DefaultController>(c => c.Index());
}
```

I started to see how the <a href="http://jeffreypalermo.com/blog/the-onion-architecture-part-1/">Onion Architecture</a>, dependency injection, model binders, AutoMapper, and the like help you to keep your controllers small.  MVC wasn't a silver bullet; its primary benefit compared to WebForms was that you could make it *get out of the way* so you could actually start laying down a solid architecture.

ASP.NET MVC has been around for a few years now.  Community contributions like <a href="http://mvccontrib.codeplex.com/">MvcContrib</a> helped to motivate change in subsequent releases, and now ASP.NET MVC is legitimately open-sourced.  Better-yet, guidance on how to do this well has had years to mature.

MVC is a sharper tool than WebForms.  It acknowledges the reality of web technologies like HTML and HTTP, while providing some reasonably-sized abstractions on top of that, such as routing and model binding.  Having a better tool in your toolbelt isn't enough, though.  You have to see what it looks like when used correctly.
