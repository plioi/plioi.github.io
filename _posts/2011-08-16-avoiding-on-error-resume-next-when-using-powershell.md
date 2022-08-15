---
title: "Avoiding \"ON ERROR RESUME NEXT\" when Using Powershell"
layout: post
---


In <a href="http://patrick.lioi.net/2011/07/29/powershell-and-the-principle-of-most-astonishment/">Powershell and the Principle of Most Astonishment</a>, I covered some of the surprises that new Powershell users encounter when trying to reliably return results from functions. The next area where Powershell suprises new users is in its approach to error handling. Fortunately, there are some useful workarounds for making the surprising default behavior work more like you would expect.

While learning Powershell, I was trying to create a deployment script. The script needed to perform several tasks including, copying the deployment package to the target machine, setting up services, and the like. While testing it out locally, I would deliberately cause certain steps to fail in order to ensure that the user of the script would be clearly alerted to failures. To my surprise, when I caused exceptions to be thrown, the script would happily continue on to the next step, ultimately printing a success message to the user. Consider this example:

```
function divide-by-zero() {
    $x = 0
    $y = 1/$x
}

write-host "A"
divide-by-zero
write-host "B"
divide-by-zero
write-host "Success!"

Output:
    A
    Attempted to divide by zero.
    At C:\dev\blog_post\default.ps1:5 char:12
    +     $y = 1/ &lt;&lt;&lt;&lt; $x
        + CategoryInfo          : NotSpecified: (:) [], RuntimeException
        + FullyQualifiedErrorId : RuntimeException
         
    B
    Attempted to divide by zero.
    At C:\dev\blog_post\default.ps1:5 char:12
    +     $y = 1/ &lt;&lt;&lt;&lt; $x
        + CategoryInfo          : NotSpecified: (:) [], RuntimeException
        + FullyQualifiedErrorId : RuntimeException
       
    Success!
```

What the heck just happened? If you can't rely on uncaught exceptions to stop execution, how can you reliably deal with failures? What if we throw an exception and then blindly move along to a subsequent step that *depends* on the success of previous steps?

**Powershell reintroduces VB's "ON ERROR RESUME NEXT", but goes one step further by making it the default!** Abandon all hope, ye who etc, etc.

To make Powershell error handling work more like error handling in other .NET languages, we can set **$global:ErrorActionPreference = "Stop"** at the start of our script. With this variable set, uncaught exceptions thrown by Powershell code will cause the whole script to stop. Altering our example with this line, we get the output that we originally expected:

```
$global:ErrorActionPreference = "Stop"

function divide-by-zero() {
    $x = 0
    $y = 1/$x
}

write-host "A"
divide-by-zero
write-host "B"
divide-by-zero
write-host "Success!"

Output:
    A
    Attempted to divide by zero.
    At C:\dev\blog_post\default.ps1:5 char:12
    +     $y = 1/ &lt;&lt;&lt;&lt; $x
        + CategoryInfo          : NotSpecified: (:) [], ParentContainsErrorRecordException
        + FullyQualifiedErrorId : RuntimeException
```

This solves most of our problem: the behavior of Powershell code that throws errors. Unfortunately, it doesn't help us when we invoke an *external* executable that fails in the middle of our script. In my next post, I'll show you how to address the failure of external executables.
