---
layout: post
title: "The Search for the Dictionary Key"
date: 2024-12-04 11:45:10 +0200
categories: [debugging, csharp, dotnet, caching]
tags: [dictionary, hashing, equality, imutable, dotnet]
---

# The Search for the Dictionary Key

Recently I helped to chase down a ghost (and you might be surprised to know that I, for the most part, spend hours to be a ghostbuster—it could be fun, sometimes).

A customer reported a weird issue: a visitor would go to their website, have everything correct in the cart, including the **discount**, only to have the discount **disappear** when they checked out.

This would be a fairly easy task to debug and fix if not for the problem being **random** in nature. It might happen once in a while, but on average, daily. It could not be reproduced locally, or consistently on production, so all fixes were based on guesswork.

---

## The Root Cause: Missing Cache

After a lot of dry code reading and then log reading, it turned out that the problem with the missing discount seemed to be a problem with **missing cache**. Once in a while, the cache that contained the promotion list was returned empty, resulting in no discount being applied to the order.

But why?

After a few guesses, it eventually came to me that the problem was with the caching using a **Dictionary**. More specifically, campaigns were loaded and cached using a `Dictionary`, with the **`IMarket`** interface as the key.

This setup would be fine and highly efficient... if not for the fact that the default implementation of `IMarket` is not suitable to be a `Dictionary` key. It does **not** implement `IEquatable<T>` (or properly override `GetHashCode()`), which means that for two `IMarket` instances to be considered equal, they must be the **exact same instance** in memory. Otherwise, even if their properties all equal in value, they will not be equal.

This is a short program that demonstrates the problem. You can expect it to write "**False**" to the output console:

```csharp
public class Program
{
    private static Dictionary<AClass, int> dict = new Dictionary<AClass, int>();
    
    public static void Main()
    {
        dict.Add(new AClass("abc", 1), 1);
        dict.Add(new AClass("xyz", 2), 2);

        // This will print 'False' because the new AClass instance 
        // is not the same reference as the one added.
        Console.WriteLine(dict.ContainsKey(new AClass("abc", 1)));
    }
}

public class AClass
{
    public AClass(string a, int b)
    {
        AString = a;
        AnInt = b;
    }

    public string AString { get; set; }
    public int AnInt { get; set; }
    // Missing: Proper override of Equals(object) and GetHashCode()
}
````

-----

## Why Only Sometimes? The Timing Trick

If the key is not matched and an empty list of campaigns returns, why does this only happen **sometimes**?

The answer is that the **`IMarket` instances themselves are cached**, by default for 5 minutes.

For the problem to occur, the timing needs to be perfect:

1.  A cache for campaigns must be loaded into memory, using an existing `IMarket` instance as the key.
2.  The cache for the **`IMarket` instances** must expire (after 5 minutes), causing new `IMarket` instances to be created on the next load.
3.  The campaign cache must be accessed again **before its own expiration** (default is often 30 seconds).

When step 3 happens, the code uses the **new** `IMarket` instance (with the same values) to look up the cache key. Since the key objects are different instances, the `Dictionary` lookup fails, returning an empty campaigns list.

The precise timing window made this problem elusive and hard to find from normal testing—both automated and manual.

-----

## The Fix (And the Confession)

Time for some blaming and finger-pointing. When I fix something, I usually try to check the history of the code to understand the reason behind the original idea and intention. Was there a reason, or just an overlook? And most importantly:

**Who wrote such code?**

Me, about 7 months ago.

Uh oh.

The fix was simple enough. Instead of `IMarket`, we changed the dictionary key to **`MarketId`**, which is a value type that properly implements `IEquatable<T>`. It does not matter if you have two different instances of `MarketId`; as long as they hold the same value, they will be equal.

A workaround was sent to the customer to test, and after a week or so, they reported back that the problem is gone. The official fix is in Commerce 14.31, which was released recently.

-----

## Lessons Learned

1.  **Pick the Dictionary key carefully.** The key type should implement **`IEquatable<T>`** and properly override **`GetHashCode()`**. In general, a **`struct`** (value type) is a better choice than a **`class`** (reference type), if you can.
2.  **You are human.** No matter how "experienced" you think you are, you are still a human being and can make mistakes. It's important to have someone to check your work from time to time, spotting problems that you couldn't.

What's the most time-consuming "ghost" you've ever had to chase down in production code?

```
```