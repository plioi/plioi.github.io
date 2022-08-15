---
title: "ToString, or not ToString"
layout: post
---


One of the first things people learn about .NET is that every Object can be turned into a string by invoking ToString(), and that this method can be overridden.  This method gets overridden for several different *reasons*, which kind of muddies the waters.  Sometimes it's overridden so that the object can be rendered for the user's eyes, and sometimes it's used to output debugging information for the developer's eyes.  By default, it simply gives the namespace-qualified type name.

While stepping through in debug mode, the debugger frequently needs a string representation of your objects so they can be displayed in a grid.  When in doubt, it calls ToString(), which suggests that we Ought To Use ToString() For That, leading to an inconsistent use of the method.

Fortunately, we have an alternative to ToString() for debugging purposes.  When a <a href="http://msdn.microsoft.com/en-us/library/x810d419.aspx">[DebuggerDisplay]</a> attribute is included on your class, you can tell the debugger to use something *other* than ToString().  The documentation for this attribute lists many caveats and includes a suggested pattern to follow in order to avoid trouble:

```cs
[DebuggerDisplay("{DebuggerDisplay,nq}")]
    public class Example
    {
        public Example(string a, string b)
        {
            PropertyA = a;
            PropertyB = b;
        }
    
        public string PropertyA { get; private set; }
        public string PropertyB { get; private set; }
        
        private string DebuggerDisplay
        {
            get { return string.Format("Example: {0} {1}", PropertyA, PropertyB); }
        }
    }
```

If you have a class that deserves a special debugger-only representation, you can follow this pattern.  First, place [DebuggerDisplay("{DebuggerDisplay,nq}")] on the class.  Then, provide a private read-only string property, also called DebuggerDisplay.  This property can return whatever you want, but to avoid surprises at debug time you probably want to define it in terms of read-only state.

DebuggerDisplay's single string argument can support a wide variety of expressions, but "{DebuggerDisplay,nq}" is all you really need to ever use.  It simply means, "Display the result of the DebuggerDisplay property, with **N**o **Q**uotes"  Anything more complex, and you're asking for either performance problems, or language incompatability problems.  We mark the property as private to avoid polluting our public interface, but that's OK since the Debugger Knows All and Sees All.
