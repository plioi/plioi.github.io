---
title: "Public, Private, Super-Private"
layout: post
---


I'll never understand *why* it works, but it turns out that generating pseudorandom numbers takes very little code. Consider the following Random class. Given a seed integer to get started, it will produce an apparently-random list of ints as you repeatedly call Next():

```cs
public class Random
{
    public int x;

    public Random(int seed)
    {
        x = seed;
    }

    public int Next()
    {
        const int k = 16807;
        const int m = Int32.MaxValue;

        x = (k*x) % m;
        return x;
    }
}
```

Easy as pie. Shortest blog post ever.

***Record Scratch***

Wait, why is `x` public? It's so wrong, your gut says "That should be private!" long before your brain catches up and agrees. The downside, of course, is that the coder using this class can mess with `x`, and that could have the effect of the algorithm no longer producing a "fair" sequence.

We mark that field private and call it a day. Later, we revisit the class to start adding convenience methods, written in terms of the existing `Next()` method:

```cs
public int NextInRange(int max)
{
    //Constrain Next() to the range [0..max]
}

public int NextInRange(int min, int max)
{
    //Constrain Next() to the range [min..max]
}
```

We balked at `x` being public because more code *could* mess with it than *should* mess with it. By marking it public, we were *telling* the next coder that altering the value of `x` was something you might want to do *from anywhere*.

However, if the person implementing additional methods on this class doesn't already know that `x` is sacred, not to be altered anywhere but within `Next()`, then marking it private hasn't actually helped. We're stuck with the same problem: by merely marking it private, we are *telling* the next coder that altering the value of `x` is something you might want to do *from anywhere within the class*.

That's a risk we're likely willing to take in practice, but in a perfect world we could further constrain `x` so that it is only visible to the method that deserves to see it. Consider the following, in which `x` is promoted from private to super-private:

```cs
public class Random
{
    public readonly Func<int> Next; 

    public Random(int seed)
    {
        int x = seed;
        const int k = 16807;
        const int m = Int32.MaxValue;

        Next = () =>
        {
            x = (k*x)%m;
            return x;
        };
    }

    public int NextInRange(int max)
    {
        //Constrain Next() to the range [0..max]
    }

    public int NextInRange(int min, int max)
    {
        //Constrain Next() to the range [min..max]
    }
}
```

This produces the same results. Although `x` is declared local to the constructor, it will still live on after construction because the `Next` lambda expression "closes over" or "grabs onto" it. The lambda is even allowed to change it. Think of every lambda expression as being an instance of a class that has fields of its own, and those fields are implied by which local variables we happen to refer to in the lambda's body.

I'm *not* recommending that you start obfuscating all of your classes in this fashion. The main point, though, is that once you get comfortable with the idea that lambdas have their own 'super private' state that lives on for the lambda's lifetime, you may find that you just don't need as many class-level fields as you used to.
