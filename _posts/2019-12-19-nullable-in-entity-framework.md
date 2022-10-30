---
title: 'Nullable Reference Types in Entity Framework'
layout: post
---

C# 8 adds [Nullable Reference Types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references), which warns us about problematic *null* values. If you're upgrading to C# 8, you may run into a few challenges related to Entity Framework. Let's walk through the upgrade and devise a few new patterns to adopt for using Entity Framework with C# 8.

## Upgrading Entity Framework Core

You'll need the latest version of Entity Framework Core, so start by making sure your Nuget package references are up to date. I upgraded `Microsoft.EntityFrameworkCore.SqlServer` to version 3.0.0, as it has first-class support for working with Nullable Reference Types.

## Enable Nullable Reference Types Across All Projects

I turned on the feature for all projects in my solution by placing a `Directory.build.props` file beside my `*.sln` file. I could have turned the feature on within each `*.csproj` file individually, but I don't want to have to remember to do this each time I add another project to the solution. I want to solve this once so that it stays solved:

```xml
<Project>
        <PropertyGroup>
            <Nullable>enable</Nullable>
        </PropertyGroup>
</Project>
```

Once you turn on the feature, you get compiler warnings based on two strong opinions:

1. A reference type declaration like a `string` is intended to never be null.
2. If a reference type declaration does deserve to be null, you have to say so: `string?`

You'll get a *lot* of warnings at first: "You *said* this wasn't going to be null, but I think you're wrong!" In the middle of a method, it can be clear what to do in response to each warning, but when it comes to Entity Framework `DbContext` and entity classes, the warnings can be a little confusing.

## Deal with DbSets

My first challenge was with my `DbContext` class's own `DbSet` properties. Anyone comfortable with EF just knows that these `DbSet` properties will be non-null by the time we need them. EF ensures that they are initialized for us, but the C# compiler doesn't know that. All the compiler sees is a non-null reference type (`DbSet<Contact>`) that is *apparently* never initialized. Rather than "chase the warning message" by adding meaningless constructor assignments, we'll instead declare that in this case, we know better. We add `= default!` to each `DbSet` property:

```cs
public class Database : DbContext
{
   public Database(DbContextOptions<Database> options)
      : base(options)
   {
   }

   public DbSet<Contact> Contact { get; set; } = default!;
}
```

Technically this means, "Initialize it to the default (null), but *act* like it's not null." In practice, we'll read this as, "Trust me! This will be defaulted for me by EF!"

## Deal with entities

Next, I got a warning about my Contact entity class. We started with a typical bag of mutable properties:

```cs
public abstract class Entity
{
   public Guid Id { get; set; }
}

public class Contact : Entity
{
   public string Email { get; set; }
   public string Name { get; set; }
   public string PhoneNumber { get; set; }
}
```

Similar to the `DbSet` warning, the compiler complained that I had declared non-null string properties, but that they would in fact be null upon construction, as they are never initialized.

This time, it would be a serious mistake to initialize them with `default!`. It was safe before because, in that case, we *knew with great certainty* that any *usage* of the properties would work without null checks. In the case of entity properties, we *know* that there is no such safety net. This time, we need to really follow the suggestion by initializing the non-null columns' properties in a constructor. In my case, only the PhoneNumber column was truly nullable:

```cs
public class Contact : Entity
{
    public Contact(string email, string name, string? phoneNumber = null)
    {
        Email = email;
        Name = name;
        PhoneNumber = phoneNumber;
    }

    public string Email { get; set; }
    public string Name { get; set; }
    public string? PhoneNumber { get; set; }
}
```

Wait, what about `Id`?

You may be surprised that we're not accepting the never-null `Id` property via the constructor as well. We leave it out for good reason.

First, as a value type, we have no warning about it; it will simply default to the empty value.

Second, we *know* that EF would prefer to own our `Id` properties. If we want to perform an `INSERT`, for instance, EF wants us to construct the entity without bothering with the `Id`, `Add(…)` it, and then save those changes. EF will know what to do and will populate our `Id` when it's good and ready. Likewise, on a `SELECT`, EF will fill it in before we ever look at it.

Our entity class is a little more verbose now, but it's also doing a better job of telling the next developer how it should be used. We're saying, "You really need at least these values to be valid, and you really should keep your hands off that `Id` property."

## Conclusion: A Mixed Bag

Enabling Nullable Reference Types is going to challenge some long-held bad habits around "good enough" coding patterns, like leaving entities poorly initialized. That can hurt at first, but it's a clear step towards correctness.

It's also going to hand us some ugliness in the form of syntax like `default!`. It's a shame that we'll have to put that on every `DbSet`, but at least it's a pattern you can learn quickly and that is contained in a single file.

Most importantly, as you explore the feature in your own projects, stop at each warning, consider what it's really telling you about what can go wrong in your system, and only then decide how to react to resolve the warning.