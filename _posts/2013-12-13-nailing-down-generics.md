---
title: Nailing Down Generics
layout: post
---
## Calling Generic Methods

You write a class containing a harmless generic method:

{% gist 7944797 %}

Actually, for the purposes of this discussion, this is **not** one method declaration. Rather, it is an infinite number of method declarations:

  * `void GenericMethod<int>(int input)`
  * `void GenericMethod<string>(string input)`
  * `void GenericMethod<Customer>(Customer input)`
  * &#8230;

Each time you make a call, the compiler decides which of the infinite methods you really meant. It makes this decision each time you call something called &#8220;GenericMethod&#8221;. It makes this decision based on the compile-time types at each call site:

{% gist 7944816 %}

The compiler knows the compile-time type of each argument, and compares this to the generic type definition in order to pick the single winning &#8220;specific&#8221; method.

Easy, right? C# 101 stuff.

## Calling Generic Methods Via Reflection

Let&#8217;s try to call the same method, with the same inputs, via reflection:

{% gist 7944838 %}

MethodInfo.Invoke(&#8230;) wants you to throw an object[] at the method. It wants to take the first item of that array for the first parameter of the method, the second item for the second parameter, etc. 

Easy, right? Wrong. Each call to Invoke above would throw a System.InvalidOperationException with the message:

> Late bound operations cannot be performed on types or methods for which ContainsGenericParameters is true.

This exception message achieves the high honor of being both 100% accurate and 10% helpful. &#8220;Late bound operation&#8221; is compiler-speak for &#8220;something that is figured out dynamically at runtime instead of statically at compile time.&#8221; The figuring out happens _later_ than compilation. The late bound operation, the thing we&#8217;re trying to _do_ dynamically at runtime, is the method invocation itself.

Translation: the exception message is saying, &#8220;You cannot invoke the method because it _still_ has generic parameters.&#8221; In order to invoke it, we&#8217;re going to have to first nail down the generic parameters to specific concrete types so that they won&#8217;t be generic anymore. We asked .NET to do the impossible:

> You: &#8220;Please call this method.&#8221;
> 
> .NET: &#8220;No. That is not a method. It is an infinite number of methods. Try again, and tell me which one you meant.&#8221;

## Once More, With Feeling

Lets narrow things down, each time we want to call the method via reflection. _Just like the compiler did for us in the original compile-time example_:

{% gist 7944850 %}

This time, thankfully, we get the same output as the original plain calls to GenericMethod<T>(T). As with the original example, we know _exactly which 3_ of the _infinite_ methods are being called.

## Generic Test Methods

Let&#8217;s say you&#8217;re using the Fixie test framework and you have [defined an [Input] attribute with an associated Fixie Convention in order to have parameterized tests](https://patrick.lioi.net/2013/09/27/a-swiss-army-katana/). If one of your test methods is generic, Fixie faces the same problem we faced above:

{% gist 7944867 %}

Fixie calls test methods via reflection, using an object[] of inputs. In this case, the object[] has length 1, and the values come from the [Input] attributes. In order to successfully invoke the MethodInfo, Fixie must also call MethodInfo.MakeGenericMethod(&#8230;), passing in the right concrete Type, in order to get a handle on the specific, concrete version of the MethodInfo. Finally, Fixie can invoke _that_ MethodInfo.

Thanks to [Anders Forsgren](https://github.com/andersforsgren), Fixie can handle tests like this one. If the generic type parameter T can be nailed down to something unambiguously specific, it will be. If a generic test is declared with multiple generic type parameters like T1, T2, etc, it&#8217;ll try to nail them all down. When there&#8217;s any ambiguity for a T, though, Fixie has to assume `object` as the most specific type possible.

In short, Fixie will do the most reasonable thing possible without you ever needing to know that any of the above is happening.