---
title: "Avoid Mutation by Default"
layout: post
---


Several weeks back, I was <a href="http://patrick.lioi.net/2012/09/13/low-ceremony-xunit/">customizing xUnit</a> to cut down on ceremony in my tests.  When I ran into some trouble, I started looking through the source code for xUnit, and saw something a little surprising: tests within a test class are executed in a random order.  Sure, tests need to be isolated so that test execution order doesn't affect their success or failure, but randomizing within the test runner seemed like overkill.

If it ever caught an issue, it still wouldn't tell me where the problem was, right?  It seemed like it would be better to execute in a deterministic order so that when I *do* get myself into trouble, it will be reproducible trouble.  Today, though, I was grateful for this behavior.  If you're running tests frequently during development, the very fact that your newly-coded tests start failing at random **is** informative: *"What test is wrong?  The one that I just wrote."*

## A Trivial Yet Broken Class

I was working on a routine whose job was to produce a date range after taking into account several business rules.  If none of the rules applied, we wanted to return the range [DateTime.MinValue, DateTime.MaxValue] to indicate "all the dates we could ever possibly care about".  Each business rule could possibly shorten this range, affecting the start, the end, or both.  The overall result was, basically, the range where several other ranges overlapped.  I started by creating a helper class, DateRange, as a glorified pair of DateTimes:

```cs
public class DateRange
{
    public DateRange(DateTime start, DateTime end)
    {
        Start = start;
        End = end;
    }

    public DateTime Start { get; set; }
    public DateTime End { get; set; }
}
```

To make it clear to the reader that we were starting with "all the dates we could ever possibly care about" and subsequently narrowing that down to the overlapping range, I added a static field:

```cs
public class DateRange
{
    public static readonly DateRange AllOfTime = new DateRange(DateTime.MinValue, DateTime.MaxValue);

    public DateRange(DateTime start, DateTime end)
    {
        Start = start;
        End = end;
    }

    public DateTime Start { get; set; }
    public DateTime End { get; set; }
}
```

This field allowed the main algorithm to look something like this:

```cs
public DateRange GetOverallDateRange()
{
    var overlappingRange = DateRange.AllOfTime;

    //Alter overlappingRange to account for rule #1...
            
    //Alter overlappingRange to account for rule #2...

    //Alter overlappingRange to account for rule #3...

    return overlappingRange;
}
```

Do you see the bug?  **The terrible, horrible, no good, very bad bug?**  xUnit did.

The 3 rules I was applying were involved yet test-friendly, so it was easy to implement several tests to make sure my bases were covered.  All was well, until I got to the seventh test.  It failed with an impossible result.  The dates in the resulting range didn't even show up anywhere in my test setup.  Mystified, I ran that test in the debugger.  It passed!  I ran the whole test class a few times, and this one test would pass or fail at random.  This was extremely odd, as the code under test seemed to be clearly deterministic.

## Excessive Mutation

Realizing that xUnit was running my tests in a different order each time helped me to see my mistake.  This implementation of DateRange is mutable, and AllOfTime is static and therefore shared.  AllOfTime is a shared mutable thing whose name suggests an immutable thing.  AllOfTime is lying.

I had broken my own rule: I make classes immutable by default, making them mutable as soon as being dogmatic starts to hurt.  Instead, I had made this mutable from the start and immediately tripped on my own intellectual shoelaces.  Marking the properties as read-only forced me to rephrase the main logic, but that actually resulted in simplifying everything and ensured that AllOfTime is in fact *all* of it:

```cs
public class DateRange
{
    public static readonly DateRange AllOfTime = new DateRange(DateTime.MinValue, DateTime.MaxValue);

    public DateRange(DateTime start, DateTime end)
    {
        Start = start;
        End = end;
    }

    public DateTime Start { get; private set; }
    public DateTime End { get; private set; }
}
```

DateRange deserves to be a value just like DateTimes deserve to be values.  You can't *change* 10/24/2012 any more than you can change the number 5. 5 will always be 5, 10/24/2012 will always be 10/24/2012, and the range [10/24/2012, 10/31/2012] will always be the range [10/24/2012, 10/31/2012].

As a general rule, I avoid mutation.  If I did that everywhere, though, my code would likely become too hard to follow.  As soon as keeping things immutable becomes a larger pain than the pain it solves, I back off and allow mutable state to come into the picture.  Even then, I try to limit the "footprint" of that state change.

Unnecessary mutation makes it too easy to make trivial mistakes.
