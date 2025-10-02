---
layout: post
title: "AsyncHelper Can Be Considered Harmful"
date: 2024-12-05 11:42:44 +0200
categories: [dotnet, async, threading]
tags: [csharp, async-await, threading, deadlock, concurrency]
---

# AsyncHelper Can Be Considered Harmful

.NET developers have been in the transition to move from synchronous APIs to asynchronous APIs. That was boosted a lot by the **`await`/`async`** keywords introduced in C# 5.0. However, we are now in a dangerous middle ground: there are as many synchronous APIs as there are async ones.

The mix of them requires the ability to call async APIs from a synchronous context, and vice versa. Calling synchronous APIs from an async context is simple—you can fire up a task and let it do the work. Calling async APIs from a sync context is much more complicated, and that is where **`AsyncHelper`** comes to the play.

---

## What is AsyncHelper?

`AsyncHelper` is a common class used to run async code in a synchronous context. It's a simple helper class with two methods to run async APIs.

Here is a typical implementation of the core logic:

```csharp
public static TResult RunSync<TResult>(Func<Task<TResult>> func)
{
    var cultureUi = CultureInfo.CurrentUICulture;
    var culture = CultureInfo.CurrentCulture;
    
    return _myTaskFactory.StartNew(() =>
    {
        Thread.CurrentThread.CurrentCulture = culture;
        Thread.CurrentThread.CurrentUICulture = cultureUi;
        return func();
    }).Unwrap().GetAwaiter().GetResult();
}

public static void RunSync(Func<Task> func)
{
    var cultureUi = CultureInfo.CurrentUICulture;
    var culture = CultureInfo.CurrentCulture;
    
    _myTaskFactory.StartNew(() =>
    {
        Thread.CurrentThread.CurrentCulture = culture;
        Thread.CurrentThread.CurrentUICulture = cultureUi;
        return func();
    }).Unwrap().GetAwaiter().GetResult();
}
````

There are slight variants, with and without setting the `CurrentCulture` and `CurrentUICulture`, but the main part is still:

1.  Spawning a new **`Task`** to run the async task.
2.  Blocking and getting the result using **`.Unwrap().GetAwaiter().GetResult()`**.

-----

## The Misconception and the Danger

One of the reasons this helper became popular was the misconception that it was written by Microsoft as a general recommendation and must therefore be safe. This is not true. The class was introduced as an **internal class** within **AspNetIdentity** (see [GitHub source](https://www.google.com/search?q=https://github.com/aspnet/AspNetIdentity/blob/main/src/Microsoft.AspNet.Identity.Core/AsyncHelper.cs)). This means Microsoft teams can use it when they think it's the right choice, but it is **not the default recommendation** for running async tasks in a synchronous context.

Unfortunately, I've seen a fair share of threads stuck in the **`AsyncHelper.RunSync`** stack trace, likely having fallen victim to a **deadlock situation**.

A typical stuck stack trace might look something like this:

```
    756A477F9790	    75ABD117CF16	[HelperMethodFrame_1OBJ] (System.Threading.Monitor.ObjWait)
    756A477F98C0	    75AB62F11BF9	System.Threading.ManualResetEventSlim.Wait(Int32, System.Threading.CancellationToken)
    756A477F9970	    75AB671E0529	System.Threading.Tasks.Task.SpinThenBlockingWait(Int32, System.Threading.CancellationToken)
    756A477F99D0	    75AB671E0060	System.Threading.Tasks.Task.InternalWaitCore(Int32, System.Threading.CancellationToken)
    756A477F9A40	    75AB676068B8	System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(System.Threading.Tasks.Task, System.Threading.Tasks.ConfigureAwaitOptions)
    756A477F9A60	    75AB661E4FE7	System.Runtime.CompilerServices.TaskAwaiter`1[[System.__Canon, System.Private.CoreLib]].GetResult()
```

An excellent further explanation of why this blocking pattern is problematic can be read in this [Stack Overflow thread on `Task.Result` vs `.GetAwaiter.GetResult()`](https://www.google.com/search?q=https://stackoverflow.com/questions/27532386/is-task-result-the-same-as-getawaiter-getresult).

-----

## Conclusion

Async/sync is a complex topic, and even experienced developers make mistakes. There is no simple, guaranteed-safe way to just run async code in a synchronous context. **`AsyncHelper` is absolutely not that simple solution.**

It is a simple, convenient way to introduce a risk of deadlocks into your application. I see it as a shortcut to solve an immediate problem, but one that creates bigger ones down the path.

> Just because you can, doesn’t mean you should. That applies to AsyncHelper perfectly.

```
```