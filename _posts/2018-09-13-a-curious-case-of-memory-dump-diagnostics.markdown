---
layout: post
title: "A Curious Case of Memory Dump Diagnostics: How Stackify Can Cause Troubles to Your Site"
author: vimvq1987
date: 2018-09-13
categories: [commerce, debugging, episerver]
tags: [windbg, memory-dump, stackify, dotnet, dxp]
---

# A Curious Case of Memory Dump Diagnostics: How Stackify Can Cause Troubles to Your Site

It’s been a while since my last blog post, and this time it will be fun time with **Windbg**. One of DXC customers has been having problem with instances of their site hang and restart. We at Episerver takes the availability and performance of DXC sites very seriously, therefore I was asked to help diagnosing the problem. It’s something I’d like to do and to learn anyway, so game is on.

The golden standard for looking at the kind of those problems is still the tried and trusted **Windbg**. With the help from the Managed Services Engineer, I was able to access several memory dumps from the site. Most of them were taken when the site stalled, which is usually the most useful moment because it should contain the most critical piece of information why the stall happened.

As usual, it’s just about firing Windbg, `Ctrl+D` to open the memory dump and start debugging, then load some essential extensions, and everything is ready.

A quick check confirms that there was a high CPU usage situation:

```

\!threadpool
CPU utilization: 81%
Worker Thread: Total: 51 Running: 33 Idle: 0 MaxLimit: 32767 MinLimit: 4
Work Request in Queue: 454

```

But now it’s important part: we don’t know which thread is causing the high CPU situation. Yes we can always go with `!threads` or `runaway` and go through all the stacktraces and guess.

It’s just hard to have a good guess. Especially when there is a **Garbage Collection (GC)** in progress in most of memory dumps, but not all. Well, when there is a GC, GC thread would suspend all other threads to clean & sweep the “dead” objects, and that can slow down the instance greatly, and Azure AutoHeal can see that the instance is not responding and restarts it. But what we really need to explain is why a Gen 2 GC happened, and how to prevent it from happening again.

And what’s about the memory dumps that there was no GC?

It might feel easy to read some guides on finding a memory leak using Windbg. That can be easy, well, on a fairly small application. But we are talking about memory dumps 6-10GB in size, and there is no clear indication of what could possibly wrong. True, I am not a master on Windbg, at least not yet, but if there is a list of things that make me really pulling my hairs, this is at the very top of it. There have been times that I think I am this close to revealing the truth, just to be proven not.

---

## The Suspect Stacktrace

But as they said, “Every cloud has a silver lining”. It comes to my attention that this stacktrace appears **consistently** in some threads:

```

System.Diagnostics.Tracing.EventSource.DispatchToAllListeners(Int32, System.Guid\*, System.Diagnostics.Tracing.EventWrittenEventArgs)+4a
...
Microsoft.ApplicationInsights.DependencyCollector.Implementation.ClientServerDependencyTracker.BeginTracking(Microsoft.ApplicationInsights.TelemetryClient)+1ee
...
Microsoft.Diagnostics.Instrumentation.Extensions.Intercept.Functions+\<\>c\_\_DisplayClass1\_0.b\_\_0(System.Object)+1e

````

But it’s the framework (Application Insights), so it can’t be the problem, right? Or can it? After reviewing several memory dumps, my suspicion grew, this appeared in almost every memory dump I had. There might really be something with it.

If we look at the last line `DispatchToAllListeners`, it sounds possible that we might have a very long list of “Listeners” and that take time. A peek into the [framework source code](https://referencesource.microsoft.com/#mscorlib/system/diagnostics/tracing/eventsource.cs,1760) seems to confirm my suspicion:

```csharp
private unsafe void DispatchToAllListeners(int eventId, Guid* childActivityID, EventWrittenEventArgs eventCallbackArgs)
{
    Exception lastThrownException = null;
    for (EventDispatcher dispatcher = m_Dispatchers; dispatcher != null; dispatcher = dispatcher.m_Next)
    {
        //Do stuffs.
    }

    if (lastThrownException != null)
    {
        throw new EventSourceException(lastThrownException);
    }
}
````

So each `EventSource` has a field of type `EventDispatcher` named `m_Dispatchers`. And as the code above suggests, **EventDispatcher is actually built as a linked list**, and each instance point to their next dispatcher. So that means if the linked list is long enough, then this method, `DispatchToAllListeners` can even take a long time, even causing a bottleneck.

The next question is to find out if our theory is “sustainable”. Let’s check how many `EventDispatcher` instances to we have:

```
0:115> !dumpheap -type EventDispatcher -stat
Statistics:
              MT    Count    TotalSize Class Name
00007ff9d3c28308        1           48 Microsoft.ServiceBus.Tracing.EventDispatcher
00007ff9d4ae8cc0   177825      8535600 System.Diagnostics.Tracing.EventDispatcher
Total 177826 objects
```

Oops\!

That does not look good. So it kinda reaffirms our suspicion. But we still need to find out which code caused this high number of `EventDispatcher`. Just pick any of those and explore.

-----

## Tracing the Listener Back to Stackify

Using `!DumpObj` on an instance of `EventDispatcher` reveals the linked list structure and, most importantly, the associated **`EventListener`**:

```
0:115> !DumpObj /d 00000244305aa758
Name:        System.Diagnostics.Tracing.EventDispatcher
...
Fields:
...
00007ff9cfc6a118  40016ed        8 ...ing.EventListener  0 instance 00000244305aa670 m_Listener
...
00007ff9d4ae8cc0  40016f0       18 ...g.EventDispatcher  0 instance 000002473197b630 m_Next
```

A check on the root chain (`!gcroot`) for one dispatcher showed a queue of about **7112 dispatchers** in the queue. If any event was ever handled by the `FrameworkEventSource`, this would be a very, very long list to walk through. And for a purpose of logging, that is too much\!

Now let’s look at the `EventListener`:

```
0:115> !DumpObj /d 00000244305aa670
Name:        StackifyHttpTracer.EventListener
MethodTable: 00007ff9d28460f8
...
File:        D:\local\Temporary ASP.NET Files\root\125ea09c\39af438d\assembly\dl3\93224559\0052a77a_8212d201\StackifyHttpTracer.dll
```

Wait, what? **`StackifyHttpTracer.EventListener`**? It is not part of Episerver Framework, nor DXC. It is not even part of Azure. A search with `StackifyHttpTracer` reveals that it is part of **Stackify**, a profiler. Looking at their `web.config`, it is confirmed that `StackifyHttpModule` is present:

```xml
<remove name="StackifyHttpModule_Net40" />
<add name="StackifyHttpModule_Net40" type="StackifyHttpTracer.StackifyHttpModule,StackifyHttpTracer, Version=1.0.0.0, Culture=neutral, PublicKeyToken=93c44ce23f2048dd" preCondition="managedHandler,runtimeVersionv4.0" />
```

But how does that relate to the high number of dispatchers? Using **dotPeek** to look into `StackifyHttpModule`, I get the answer:

```csharp
public class StackifyHttpModule : IHttpModule
{
    private static readonly DateTime _Epoch = new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc);
    private ModuleSettings _moduleSettings = new ModuleSettings();
    private EventListener el = new EventListener(); // <-- THIS IS THE KEY!
```

It is easy to think that you can have only one instance of your `IHttpModule` in your website (I did), but the memory dump is saying otherwise. It seems like Azure creates instances of `IHttpModule` over time (perhaps per AppDomain or during restarts), even though I’m not sure why, or when:

```
0:115> !dumpheap -type StackifyHttpTracer.StackifyHttpModule -stat
Statistics:
              MT    Count    TotalSize Class Name
00007ff9d28465f0      305         9760 StackifyHttpTracer.StackifyHttpModule
Total 305 objects
```

So that’s it. We have **305 instances** of `StackifyHttpTracer.StackifyHttpModule`, and each of them has an **instance** of `StackifyHttpTracer.EventListener`. Worse, many might have been disposed, but their `EventListener` stays attached to the `EventSource`'s global linked list:

```
0:115> !dumpheap -type StackifyHttpTracer.EventListener -stat
Statistics:
              MT    Count    TotalSize Class Name
00007ff9d28460f8     7112       341376 StackifyHttpTracer.EventListener
Total 7112 objects
```

This number of `EventListener` matches with the high number of `EventDispatcher` in our previous check. **Oops\!**

That explains why we have that many instances of `EventDispatcher` in place. Every time a new `StackifyHttpModule` was created, it created an instance `EventListener` which appended itself to the system's tracing dispatch list, forcing all events to iterate over a perpetually growing queue of thousands of listeners.

-----

## Resolution and Impact

The solution was simply to **remove that `StackifyHttpModule` from the `web.config`** to get rid of the memory leak and bottleneck.

I do not daresay it is a bug in Stackify, but it seems to me that the `EventListener` field should have been `static` instead of instance to prevent multiple registrations upon module creation.

The result? The customer deployed a new version which removed `StackifyHttpModule`, and their average server response time **dropped from around 130ms (and increasing)** to around **80ms (and stable)**. For some of their most frequently called methods, there was an improvement of **4 or 5 times faster**. True, there might be some other improvements in their new version, I really think removal of the module had the biggest impact.

My takes from the story:

  * Sometimes it’s hard to find the answer. Just keep looking and **leave no stone untouched**.
  * This is why you should use **Episerver DXC**—you get early alerts on any anomalies and you get the expert help to solve the problem.

