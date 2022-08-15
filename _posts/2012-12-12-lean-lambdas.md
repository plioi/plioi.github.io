---
title: "Lean Lambdas"
layout: post
---


**Shirley:** Tell me about the Extract-Variable refactoring.

**Albert:** Let's say I have a long expression that has become hard to read, and that I can identify a noteworthy subexpression within it.  I can 'extract' that subexpression from the original by introducing a new local variable.  The variable is initialized to the subexpression, and then the larger original expression is rewritten in terms of the new variable.  Doing so can make things easier to read: the original expression is shorter, and we've given a useful name to the part we took out.  For example:

```cs
//Before
public decimal Discount(OrderDetail[] items)
{
    return items.Where(x => x.EligibleForDiscount).Sum(x => x.Price) / 10;
}

//After
public decimal Discount(OrderDetail[] items)
{
    var eligibleItems = items.Where(x => x.EligibleForDiscount);

    return eligibleItems.Sum(x => x.Price) / 10;
}
```

**Shirley:** Since we're doing this extra step with an extra variable, will the result run a little slower than the original?

**Albert:** No, silly!  We're doing the same amount of work in both versions.  Even with the original version, the compiler had to break down the expression into many smaller steps just like the change we made.  We're just introducing a useful nickname for one of those intermediate steps.

**S:** Is there any way Extract-Variable *could* make something faster?

**A:** Aren't you listening!?  Oh, wait.  If the expression you are pulling out appears multiple times in the larger expression, and the cost of evaluating it is large enough to notice, then Extract-Variable could actually speed things up, since you'll only evaluate it once.

**S:** If you do that, is there any chance you'll change the runtime behavior?

**A:** Only if the subexpression has side effects.  We used to evaluate it multiple times, and now we evaluate it once.  If the expression *does* something to the world other than just provide a value, we'd be introducing a breaking change.  That's why you want to keep as much of your code as pure as reasonably possible.

**S:** Pure?

**A:** An expression or function is pure if it depends only on its inputs and has no side effects.  It just looks at the inputs and returns a result.  For the same inputs, it always gives the same output.  `x+5` is pure, while `SendAnEmailToGrandma()` isn't.  It's safe to Extract-Variable with pure code, even when the subexpression showed up multiple times beforehand.  Pure stuff is easier to chop up, reorder, and move around.  It's easier to reason about.

**S:** Imagine that we've got a large expression, and that the subexpression we're extracting only appears *once*.  What if I told you we could *still* get a performance boost out of it?

**A:** Shirley, you're joking.

**S:** Nope! I've got two situations in mind.  Let's start with the easy one.  This loop is taking way too long to execute:

```cs
public GridColumn CreateCalculatedColumn(Grid grid, string caption, string userExpression)
{
    List<object> values = new List<object>();
    
    foreach (var row in grid.Rows)
        values.Add(CompileUserExpression(userExpression).Evaluate(row));

    return new GridColumn(caption, values);
}
```

**A:** Oh, I see what you're getting at.  We saw earlier how you can get a performance boost when extracting a subexpression that appeared multiple times in the original expression.  In this loop, we've got the same problem in a different format.  The subexpression *appears* once, but it gets *evalutated* a whole lot, each time with the same result.  We'd fix this with two refactorings: we'd Extract-Variable, and then we'd move that variable declaration *before* the loop:

```cs
public GridColumn CreateCalculatedColumn(Grid grid, string caption, string userExpression)
{
    List<object> values = new List<object>();
    
    var compiledExpression = CompileUserExpression(userExpression);
    foreach (var row in grid.Rows)
        values.Add(compiledExpression.Evaluate(row));

    return new GridColumn(caption, values);
}
```

**S:** Great! Earlier, I said I was thinking about two examples in which code with a singly-occurring subexpression could be sped up with Extract-Variable.  The easy one was the `foreach` you just fixed, but the second one's tough.  Let's improve upon the previous example.  Instead of building up a big list of all the values in the column, we'll make a column object that uses a `Func<Row>` to evaluate each cell only as needed:

```cs
public GridColumn CreateCalculatedColumn(Grid grid, string caption, string userExpression)
{
    return new CalculatedGridColumn(grid, caption, row => CompileUserExpression(userExpression).Evaluate(row));
}
```

**A:** Again, we don't want to keep on performing the costly CompileUserExpression(...) call every time we need to display a grid cell.  The problem is, if I Extract-Variable here, we're left with the same wasteful recompilation:

```cs
public GridColumn CreateCalculatedColumn(Grid grid, string caption, string userExpression)
{
    return new CalculatedGridColumn(row =>
    {
        var compiledExpression = CompileUserExpression(userExpression);
        return compiledExpression.Evaluate(row);
    });
}
```

**S:** If the lambda gets called a bajillion times, how many times will we evaluate the costly part?

**A:** A bajillion.

**S:** Hmmph.

**A:** ...

**S:** How is this any different from the `foreach` example?

**A:** When the costly bit was in a `foreach`, we could just move the variable to the outer scope.  Here, we have a lambda.

**S:** Doesn't a lambda introduce a scope too?

**A:** Well, yeah, but if we move `compiledExpression` outside of the lambda, wouldn't it become available for garbage collection as soon as we returned from `CreateCalculatedColumn`?  Let's see:

```cs
public GridColumn CreateCalculatedColumn(Grid grid, string caption, string userExpression)
{
    var compiledExpression = CompileUserExpression(userExpression);

    return new CalculatedGridColumn(row =>
    {
        return compiledExpression.Evaluate(row);
    });
}
```

**A:** It works!  Our lambda is referring to a variable in the surrounding scope, and we're returning the lambda.  Lambdas are just objects, so this is just like returning any other object that references a locally-constructed thing.  The garbage collector only cleans up unreachable things, and `compiledExpression` will be reachable for as long as the lambda is reachable.  We get to avoid all that rework no matter how many times the lambda is called!

**S:** Slick.

**A:** How does one end a Socratic Dialogue?

**S:** How about with a summary?

**A:** Extract-Variable starts out as a simple refactoring to improve the readability of large expressions.  When working with pure code, the change is both safe and easy to perform.  In addition to aiding readability, Extract-Variable can occasionally improve performance, when the extraction keeps us from doing a lot of excess work.  After extracting a variable, we can consider moving it to an outer scope, so long as doing so preserves behavior.  Moving it outside of a loop can prevent wasteful reevaluation across many iterations.  Likewise, moving it outside of a lambda can prevent wasteful reevaluation across many invocations of the lambda.
