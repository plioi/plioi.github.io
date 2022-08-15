---
title: "The Debugger Must Be Broken!"
layout: post
---


Recently, a coworker and I ran into one of those bugs that seem to behave so strangely that you start to wonder, "Is it me, or is the debugger itself buggy?"  It appeared that the simplest of operations, the assignment operator, was misbehaving.  In the debugger, we seemed to witness a value being properly set to a non-null object and then secretly nulled out as soon as we tried to look at it again.

Of course, whenever a program's behavior is so profoundly counter to all that we know to be true, we have to stop, take a deep breath, and proudly exclaim, "I am making an incorrect assumption!  This code does not say what I think it says."

I've boiled the problem down to a simpler form than the original code, and changed names to protect the guilty (me).

We had a collection of DTO instances, of a type we did not have control over (`StudentDTO`).  In the rest of our system, we wanted to use one of our own classes instead (`Student`), as *our* class implemented an interface that was consumed elsewhere (`Person`):

```cs
public interface Person
{
    string Name { get; }
    int Age { get; }
}

public class Student : Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

public class StudentDto
{
    public string Name { get; set; }
    public int Age { get; set; }
}

public static class MappingExtensions
{
    public static IEnumerable<Person> ToPersons(this IEnumerable<StudentDto> students)
    {
        return students.Select(x =>
        {
            var student = new Student
            {
                Name = x.Name,
                Age = x.Age
            };

            //This lambda could have been shorter, but
            //this makes for a convenient breakpoint.
            return student;
        });
    }
}
```

This first pass was bug-free, and the `ToPersons` method passed our unit test:

```cs
public class MappingExtensionTests
{        
    [Fact]
    public void ShouldConvertStudentDtosToPersons()
    {
        var dtos = new[]
        {
            new StudentDto { Name = "John Doe", Age = 24 },
            new StudentDto { Name = "Jane Doe", Age = 30 }
        };

        var persons = dtos.ToPersons().ToArray();

        persons[0].Name.ShouldEqual("John Doe");
        persons[0].Age.ShouldEqual(24);

        persons[1].Name.ShouldEqual("Jane Doe");
        persons[1].Age.ShouldEqual(30);
    }
}
```

Easy.  C# 101 stuff.  Of course it passed.

Some hours later, I had reason to convert `Person` from an interface to a concrete class:

```cs
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

Suddenly, this and many other tests started failing.  `persons[0].Name` was null!  We set up a breakpoint on the return statement within `ToPersons`, and saw that each student being returned did in fact get populated correctly.  **Surely, the debugger must be broken!  We clearly populate an object, and when we next look at it it is unpopulated!  Those punks at Microsoft really grind my gears.**

**Ok, Patrick, take a step back.  What assumption is leading me to believe the debugger is wrong?**  I'm claiming that a clearly-populated property is null the next time I look at it.  Therefore, I should start to doubt whether I am actually inspecting the same property both times.

In the debugger, we had already seen the `Student.Name` property set as expected.  We then went to the definition of the Name property that appears in our unit test assertion, which took us to `Person.Name`.  Here, we also saw a ReSharper warning on each property within `Student`.  We hadn't even *changed* `Student`, but that is where the warning appeared: "The keyword 'new' is required on 'Name' because it hides property 'Person.Name'."

Oh, right.

This is one of those details you learn about C# early on and then immediately forget, as it so rarely shows up in practice.  Usually, when a subclass defines members with the same name and signature as its parent class, we are in a abstract/override or virtual/override situation, in which you are knowingly and explicitly stating that the subclass provides its own definition of the parent class's behavior.  When a subclass instead defines the same members as its parent *without* overriding, you get *two* different implementations at runtime.  It's basically just a *coincidence* that they have the same name.

In this situation, when you look at an instance through a variable declared as the parent type, you see the parent type's properties.  When you look at an instance through a variable declared as the subtype, you see the subtype's properties.  As we looked at the student variable being returned from the lambda, the debugger showed us the populated `Student.Name`, and when our test assertion looked at the person returned from the lambda, the test experienced the unpopulated `Parent.Name`.

When up is down, black is white, and the debugger stops working, put the apparently impossible behavior into plain English, and cast some healthy doubt on each word.  You've got a faulty assumption in there somewhere.
