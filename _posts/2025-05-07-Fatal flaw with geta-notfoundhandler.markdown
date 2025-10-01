# Fatal Flaw with Geta-NotFoundHandler

First of all, I'd like to make this a **farewell post** to this blog. The VM that is hosting this blog—which is generously sponsored by my employer—is being decommissioned. I have looked at some alternatives to move my blog to, but none is interesting in term of time/effort/cost. With all the things going on in my life, setting up a new blog is not of priority (that'd be, surprise, surprise, **tomatoes** these days). We had a good run, and I hope this blog has been useful to you, one way or another. And I hope to see you again, some days. 👋

Back to business. The topic of today is a flaw that's quite fatal in **geta-notfoundhandler**, which is open source at GitHub: [Geta/geta-notfoundhandler: The popular NotFound handler for ASP.NET Core and Optimizely, enabling better control over your 404 page in addition to allowing redirects for old URLs that no longer works.](https://github.com/Geta/geta-notfoundhandler).

We had cases where a customer’s instance hit **very high CPU**, essentially hanging, and the only course of action was to restart the instance. Memory dumps taken at the time all pointed out to the `notfoundhandler`, specifically, `CustomRedirectCollection`.

This issue was brought to me by a colleague a couple of months ago. As he treated it as low-key, we took a quick look into it, had some good ideas, but we never really got to the bottom of it. I let it slip because it was not critically urgent/important (and you know that is when something will not be addressed).

Another colleague brought it up again recently, and while I was suffering a bad headache from a prolonged flu, I decided to get this over with. Before jumping to the conclusion and the fix, let's start with the symptoms and analysis.

-----

## Symptoms and Analysis

As mentioned above, in the memory dumps taken, we saw one or two threads stuck in this stack trace:

```
00007E69D57E67F0 00007e6c3bded5c3 Geta.NotFoundHandler.Core.Redirects.CustomRedirectCollection.CreateSubSegmentRedirect(System.ReadOnlySpan`1, System.ReadOnlySpan`1, Geta.NotFoundHandler.Core.Redirects.CustomRedirect, System.ReadOnlySpan`1)
00007E69D57E6880 00007e6c3a47ea43 Geta.NotFoundHandler.Core.Redirects.CustomRedirectCollection.FindInternal(System.String)
00007E69D57E6920 00007e6c3c1e2be5 Geta.NotFoundHandler.Core.Redirects.CustomRedirectCollection.Find(System.Uri)
00007E69D57E6950 00007e6c3c1e2a19 Geta.NotFoundHandler.Core.RequestHandler.HandleRequest(System.Uri, System.Uri, Geta.NotFoundHandler.Core.Redirects.CustomRedirect ByRef)
00007E69D57E6990 00007e6c3bd326f7 Geta.NotFoundHandler.Core.RequestHandler.Handle(Microsoft.AspNetCore.Http.HttpContext)
00007E69D57E69E0 00007e6c3bc9f9ba Geta.NotFoundHandler.Infrastructure.Initialization.NotFoundHandlerMiddleware+d__2.MoveNext()
00007E69D57E6A40 00007e6c3bce9418 System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[[Geta.NotFoundHandler.Infrastructure.Initialization.NotFoundHandlerMiddleware+d__2, Geta.NotFoundHandler]](d__2 ByRef) [/_/src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/AsyncMethodBuilderCore.cs @ 38]
00007E69D57E6A90 00007e6c3bce9360 Geta.NotFoundHandler.Infrastructure.Initialization.NotFoundHandlerMiddleware.InvokeAsync(Microsoft.AspNetCore.Http.HttpContext, Geta.NotFoundHandler.Core.RequestHandler)
```

If there are a lot of threads stuck in the same stack trace, you'd suspect **lock contention** (i.e., many threads waiting for a lock to be released). But if there are only a few threads in a case of high CPU, that suggests an **endless loop**, some `do while` or `while` code that never exits properly.

One quite infamous case of the endless loop is when you mess with a `Dictionary` in a concurrent scenario—adding to it while other threads are reading. That's when your `foreach` will never end.

But where?

Luckily for us, the library is open source, so it's much easier to try some dry code reading and guess where the problem is. And of course, we found a `while` loop:

```csharp
        // Note: Guard against infinite buildup of redirects
        while (appendSegment.UrlPathMatch(oldPath))
        {
            appendSegment = appendSegment[oldPath.Length..];
        }
```

The only remaining question is what value of `oldPath` would trigger that endless loop. Lazy as I am, I turned to CoPilot to have it analyze the code to see what values could potentially be the culprit. Blah blah, CoPilot says something like `/abc/abc/abc/` could. But it does not. (We are still a long way from losing our jobs to AI, folks 😅)

That was the stop of my first engagement.

-----

## The Culprit: An Empty `oldUrl`

So with a new memory dump, I dived again into the problem. After identifying the offending thread, let's run `!clrstack -p` to see if we can figure out the URL that triggers the issue. This really raised my eyebrow:

```
...
0:026> !DumpObj /d 00007e6ad7fff3a0
Name:        System.String
MethodTable: 00007e6c2ffbd2e0
EEClass:     00007e6c2ff97b10
...
String:      
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007e6c2ffa9018  40002ba        8         System.Int32  1 instance                0 _stringLength
00007e6c2ff0e5a8  40002bb        c          System.Char  1 instance                0 _firstChar
00007e6c2ffbd2e0  40002b9       d0        System.String  0   static 00007e6ad7fff3a0 Empty
The **oldUrl is empty** here.
```

And to my surprise, passing an **empty value** to the `while` loop above really tricks it to never exit. A simple program to demonstrate the issue:

```csharp
var appendSegment = ReadOnlySpan<char>.Empty; 
var oldPath = "".AsSpan();
while (appendSegment.UrlPathMatch(oldPath))
{
    appendSegment = appendSegment[oldPath.Length..];
}

internal static class SpanExtensions
{
    public static ReadOnlySpan<char> RemoveTrailingSlash(this ReadOnlySpan<char> chars)
    {
        if (chars.EndsWith("/"))
            return chars[..^1];

        return chars;
    }

    public static bool UrlPathMatch(this ReadOnlySpan<char> path, ReadOnlySpan<char> otherPath)
    {
        otherPath = RemoveTrailingSlash(otherPath); // otherPath is "" (empty span)

        if (path.Length < otherPath.Length) // path.Length (0) < otherPath.Length (0) is FALSE
            return false;

        for (var i = 0; i < otherPath.Length; i++) // The loop for i < 0 is SKIPPED
        {
            var currentChar = char.ToLowerInvariant(path[i]);
            var otherChar = char.ToLowerInvariant(otherPath[i]);

            if (!currentChar.Equals(otherChar))
                return false;
        }

        if (path.Length == otherPath.Length) // 0 == 0 is TRUE
            return true; // Returns TRUE!

        return path[otherPath.Length] == '/';
    }
}
```

The problem is that there was a safeguard for a **`null` value**, but not for **`Empty`**.

To trigger the issue, you will need to have:

1.  A custom redirect rule that has an **empty `oldUrl`**.
2.  A URL that does not match any other rules defined.

Once you reach the empty rule, it's the death sentence. The `while` loop never exits, and it will eat all CPU resources until the instance is restarted. 💀

-----

## The Fix

The fix in this case is to check the `oldUrl` for both `null` and **`Empty`**. If you are using the package in your project, this is the change you need:

[geta-notfoundhandler/src/Geta.NotFoundHandler/Core/Redirects/CustomRedirectCollection.cs at master · quanmaiepi/geta-notfoundhandler](https://www.google.com/search?q=https://github.com/Geta/geta-notfoundhandler/blob/master/src/Geta.NotFoundHandler/Core/Redirects/CustomRedirectCollection.cs)

Maybe someone can take this and contribute to the main repo for everyone, when I’m busy attending my tomatoes. 🍅

![tomatoes](image-2.png)
