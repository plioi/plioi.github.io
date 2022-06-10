---
title: 'Shufflin&#8217; Ain&#8217;t Sortin&#8217;'
layout: post
---
I recently added a test-shuffling feature to the [Fixie](https://github.com/fixie/fixie) test framework. The basic idea is that you can opt into randomizing the order of execution for tests within a test class. I&#8217;ll cover the details of the test shuffling feature in my next post. Today, let&#8217;s take a deeper look at shuffling in general. Readily-available advice on shuffling is actually quite wrong.

## The Too-Clever One-Liner

Let&#8217;s say you want to randomize the order of items in a simple .NET array. A little googling will lead you to solutions like this:

{% gist 6565356 %}

```cs
Random rnd=new Random();
string[] shuffled = array.OrderBy(x => rnd.Next()).ToArray();
```

You may have also run into similar shufflers that use Guid.NewGuid() for the same effect. This seems pretty reasonable. We&#8217;re &#8220;randomingly sorting&#8221; the items. Each time the sort algorithm needs to decide the order of two items, it&#8217;ll basically flip a coin. **go What could wrong possibly?**

I&#8217;m sure I&#8217;ve followed this advice on past projects, and never ran into any trouble. It might actually work, but if so it works only due to a convenient accident of implementation details.

Sorting algorithms operate with a very specific contract in mind. They frequently need to compare two items, and you are expected to provide a consistent answer to the question, &#8220;Which comes first, a or b?&#8221;. (In the case of OrderBy, the lambda provides a &#8220;key selector&#8221; for shorthand: a comes before b if a&#8217;s &#8216;key&#8217; comes before b&#8217;s &#8216;key&#8217;. The overall contract of consistent ordering is still present.)

Consider a humble [Bubble Sort](http://stackoverflow.com/a/1595310):

{% gist 6565370 %}

```cs
//Jon Skeet's Bubble Sort from http://stackoverflow.com/a/1595310
public void BubbleSort<T>(IList<T> list, IComparer<T> comparer)
{
    bool stillGoing = true;
    while (stillGoing)
    {
        stillGoing = false;
        for (int i = 0; i < list.Length-1; i++)
        {
            T x = list[i];
            T y = list[i + 1];
            if (comparer.Compare(x, y) > 0)
            {
                list[i] = y;
                list[i + 1] = x;
                stillGoing = true;
            }
        }
    }
}
```

Imagine our comparer flips a coin each time it is asked to compare two items in the array. That while loop starts to look pretty scary. Each iteration of the outer loop makes a single pass through the list. If _any_ items get swapped during that pass, we are `stillGoing` and will execute another pass through the list. When would the bubble sort even terminate? It&#8217;ll terminate when we _happen_ to generate `list.Length` _consistent_ answers in a row. A few test runs of this loop against a 10-item array reached _thousands_ of iterations before terminating!

Sure, .NET&#8217;s array sorter doesn&#8217;t use Bubble Sort, but the clever one-liner may fall into the same basic trap. In practice it seems to work, but just try proving that it will always work. What algorithm does .NET&#8217;s `Array.Sort(...)` use? Insertion Sort? Heap Sort? Quick Sort? Trick question: [it could do any of those](http://msdn.microsoft.com/en-us/library/6tf1f0bc.aspx) depending on the input array. You could inspect the implementation all day long, constructing an ironclad proof that a randomized comparer will in fact work in all possible cases, and still run into trouble when a future implementation changes the underlying algorithm(s).

Randomized sorting is a contradiction in terms.

## Real Shuffling

Thankfully, real shuffling is a solved problem. We don&#8217;t have to violate an algorithm&#8217;s contract if we use the right algorithm.

{% gist 6565381 %}

```cs
//Matt Howells's Shuffler from http://stackoverflow.com/a/110570
public static void Shuffle<T>(this T[] array, Random random)
{
    int n = array.Length;
    while (n > 1)
    {
        int k = random.Next(n--);
        T temp = array[n];
        array[n] = array[k];
        array[k] = temp;
    }
}
```

> Most of the world&#8217;s loops have already been written, and they have names. This is the Fisher-Yates Shuffle. It randomizes the order of array items in a single pass.

We&#8217;ve adopted the term &#8220;contract&#8221; for a reason: two parties make an agreement about how they are meant to interact. Violate the contract, and who knows what will happen.
