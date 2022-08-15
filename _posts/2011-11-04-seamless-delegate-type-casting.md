---
title: "Seamless Delegate Type Casting"
layout: post
---


In <a href="http://patrick.lioi.net/2011/10/28/seamless-generic-type-casting/">Seamless Generic Type Casting</a>, we saw how the `in` and `out` keywords can make our generic interfaces polite with respect to casting. A `List<string>` can be treated as an `IEnumerable<object>` because a) all strings are objects and b) `IEnumerable` is declared with the `out` keyword. Now, we'll see how these keywords can be applied to delegate types in addition to interface types.

Suppose we have a a basic hierarchy of entity classes:

```cs
public abstract class Employee { ... }
public class FieldTechnician : Employee { ... }
public class SalesPerson : Employee { ... }
```

Also, suppose we find ourselves creating/persisting many instances of these in our integration tests, and we create a delegate type with some helper methods for doing so:

```cs
public delegate T EntityCreator<T>();

public static Employee CreateBasicFieldTechnician()
{
    return new FieldTechnician { /* Populate basic properties. */ };
}

public static Employee CreateBasicSalesPerson()
{
    return new SalesPerson { /* Populate basic properties. */ };
}

public static Employee[] PersistEmployees(params EntityCreator<Employee>[] creators)
{
    var employees =  creators.Select(creator => creator()).ToArray();

    //Persist all the Employee instances here.

    return employees;
}
```

In our integration tests, we may use this to quickly get some sample employees into our database:

```cs
var employees = PersistEmployees(CreateBasicFieldTechnician, CreateBasicSalesPerson);
```

As we write some tests, we realize that we need some of our sample employees to be populated differently:

```cs
EntityCreator<SalesPerson> createSpecializedSalesPerson = () => new SalesPerson{ /* Populate some additional properties. */ };

var employees = PersistEmployees(CreateBasicFieldTechnician, CreateBasicSalesPerson, createSpecializedSalesPerson);
```

Surprise! This won't even compile. The compiler can't assign our `EntityCreator<SalesPerson>` to the expected type `EntityCreator<Employee>`. That's strange, since we know that any `SalesPerson` it ever makes will **obviously** be an Employee. As we saw with similar situations around interfaces, we need to use the `out` keyword when we define the delegate type, promising to the compiler that the `T` will only appear in the return value. Armed with this additional promise, the compiler can safely treat all `EntityCreator<FieldTechnician>` as `EntityCreator<Employee>`. With the following change, the example compiles as expected:

```cs
public delegate T EntityCreator<out T>();
```
