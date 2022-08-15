---
title: "Powershell and Calling External Executables"
layout: post
---


In my last post on <a href="http://patrick.lioi.net/2011/08/16/avoiding-on-error-resume-next-when-using-powershell/">error handling when using Powershell</a>, we saw how Powershell's default behavior for uncaught exceptions allows the rest of your script to continue running in a likely-invalid state, and how setting **$global:ErrorActionPreference = "Stop"** changes that behavior to stop as soon as an uncaught exception surfaces.

Unfortunately, we can *still* be surprised by Powershell's error handling behavior. There are two main categories of errors your Powershell script might encounter:

1. Uncaught exceptions thrown by your Powershell process.
2. Failure exit codes returned by external programs that your Powershell process invoked.


When we set the ErrorActionPreference to "Stop", we only change the behavior of uncaught exceptions. We have to work a little harder to cover the other category. Consider a Powershell script that calls out to an external program which returns its own failure exit code:

```
schtasks /foo #Let's send a bogus argument to the task scheduler executable
write-host "This should not be output!"

Output:
    ERROR: Invalid argument/option - '/foo'.
    Type "SCHTASKS / QUERY /?" for usage.
    This should not be output!
```

External executables are invoked using similar syntax as Powershell commands like `write-host`, so it isn't obvious that there's anything special about the call to `schtasks`. Since it is an external program, however, it can't exactly throw a .NET exception - it might not even *be* a .NET program.

The lingua franca of command-line failure is exit codes, rather than exceptions. An exit code of 0 means "all is well", and anything else should be treated as a failure.

We have two problems:

1. calls to external programs don't feel special to the reader, even though they behave differently from normal Powershell commands
2. we're tempted to explicitly follow each external call with a corresponding test-and-throw code block, which would be ugly and distracting to the reader


Fortunately, we can take advantage of Powershell's flexible syntax to address both of these concerns. It turns out that a Powershell function can accept a *code block* as an argument, effectively allowing us to add new keywords to the language. Consider the helper function 'exec':

```
function script:exec {
    [CmdletBinding()]

	param(
		[Parameter(Position=0,Mandatory=1)][scriptblock]$cmd,
		[Parameter(Position=1,Mandatory=0)][string]$errorMessage = ("Error executing command: {0}" -f $cmd)
	)
	& $cmd
	if ($lastexitcode -ne 0)
	{
		throw $errorMessage
	}
}
```

When we use exec, we can pass it a { code block surrounded in braces }. This way we can make our external commands stand out as special, using just one new 'keyword', and we get the error handling we wanted to boot:

```
exec { schtasks /foo } #Let's send a bogus argument to the task scheduler executable
write-host "This should not be output!"

Output:
    ERROR: Invalid argument/option - '/foo'.
    Type "SCHTASKS / QUERY /?" for usage.
    Error executing command:  schtasks /foo 
    At C:\dev\blog_post\default.ps1:11 char:8
    +         throw <<<<  $errorMessage
        + CategoryInfo          : OperationStopped: (Error executing command:  sch 
       tasks /foo :String) [], RuntimeException
        + FullyQualifiedErrorId : Error executing command:  schtasks /foo
```

Eureka! We can ensure that both categories of failures will stop execution dead in its tracks.

To sum up, the default error handling behavior in Powershell is dramatically different from that of other .NET languages. **By default, your script will happily and disasterously continue running even when throwing exceptions.** This can leave the user of the script confused as to whether the process has actually succeeded or failed. By setting the global ErrorActionPreference and by wrapping external commands with the 'exec' helper function, Powershell's behavior can become what we expected in the first place.
