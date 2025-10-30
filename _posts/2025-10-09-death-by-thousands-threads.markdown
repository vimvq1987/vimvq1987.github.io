---
layout: post
title: Death by a (Few) Thousands of Threads 👻
subtitle: Or how silence can be bad for you.
date: 2025-10-09 12:00:00 +0200
categories: [debugging, memory-leak, .net, application-insights]
tags: [dotnet, memory-dump, application-insights, thread-leak, unmanaged-memory]
---

There are things in life that excites and scares you at the same time. Recently I helped out a case when it's both exciting and scary. The problem was a gigantic **106 GB memory dump** with a very small managed heap compared to the dump's size. That hints **unmanaged** memory leak. While I'm fairly confident with memory dump analysis, I haven't had a lot of good luck with unmanaged memory leaks. It could have turned into a big headache and unknown territory. But if it's not me then who will go to hell? Then game is on.

## The Unmanaged Mystery

As we said, the actual memory dump is 106GB insize. The actual managed heap was only about **750 MB**. When you have a big memory dump that also has a big heap, you likely have some dominant object types. But if you have a (very) small heap, it means a lot of memory is allocated as **unmanaged**, which can be, most of the time, very tricky to figure out. 

The heap statistics looked like this:

```
7644edfba450    87,127   4,895,256 System.String[]
7644f0b1b718    88,507   4,956,392 EPiServer.Core.PropertyBoolean
7644f77fb8d8    81,056   5,187,584 Microsoft.Extensions.Caching.Memory.PostEvictionDelegate
7644fa376d60    73,994   5,327,568 System.Collections.Concurrent.ConcurrentDictionary<EPiServer.Core.Internal.ContentCacheKeyLookup+KeyLookup, System.String>+Node
7644f0b16f30    83,480   5,342,720 EPiServer.Core.PropertyString
7644f88e8020     1,622   6,526,928 Microsoft.ApplicationInsights.Channel.ITelemetry[]
7644fa3b5390    31,727   7,371,552 EPiServer.Core.PropertyData[]
7644f4ffd310    82,950   8,626,800 Microsoft.Extensions.Caching.Memory.CacheEntry
7644eeb43710    88,044   9,156,576 System.Reflection.RuntimeMethodInfo
7644edea89e0   109,744  10,730,392 System.Int32[]
7644fa730e58    30,460  23,308,416 System.Collections.Generic.Dictionary<System.String, EPiServer.Core.PropertyData>+Entry[]
7644f0b12b90   317,446  25,395,680 EPiServer.Core.PropertyLongString
7644eeb8c418    90,688  58,585,762 System.Byte[]
7644edead7c8 1,051,306 148,543,168 System.String
62efedb200b0    90,833 213,897,448 Free
Total 5,623,983 objects, 763,492,663 bytes

```

Then where is the rest?
When I was about to pull the hairs (whatever I have left), I realized I don't have to. Something else caught I attention. Before diving into the scary unmanaged memory part, maybe it's worth checking this first.

## The Thread Bomb

The silver lining was this: multiple threads stuck at the same stack trace. In a smaller dump, there were 1,647 threads with **1,600** of them in the same stack trace. In the big one, there were **13,000**! Yes, you read that right, thirteen thousands. And most of them were in this same stack trace:

```

00007603C3FFE8B0 000076456c8bcf16 [HelperMethodFrame: 00007603c3ffe8b0] System.Threading.WaitHandle.WaitOneCore(IntPtr, Int32)
00007603C3FFE9E0 00007644fd999020 System.Threading.WaitHandle.WaitOneNoCheck(Int32) [/*/src/libraries/System.Private.CoreLib/src/System/Threading/WaitHandle.cs @ 128]
00007603C3FFEA20 00007644f872440a **Microsoft.ApplicationInsights.Channel.InMemoryTransmitter.Runner()**
00007603C3FFEAB0 00007644fb42db75 System.Threading.ExecutionContext.RunInternal(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object) [/*/src/libraries/System.Private.CoreLib/src/System/Threading/ExecutionContext.cs @ 179]
00007603C3FFEB10 00007644fb4810aa System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef, System.Threading.Thread) [/\_/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.cs @ 2345]
00007603C3FFECD0 000076456c542c37 [DebuggerU2MCatchHandlerFrame: 00007603c3ffecd0]

```

The trace clearly suggests this has something to do with **Application Insights** telemetry. I turned to CoPilot to get a better understanding—it is surprisingly good at that. And here is what it suggested:

✅ **What You Should Check**

If you're seeing many such threads or suspect a leak:
- Ensure `TelemetryClient` is reused (singleton pattern)
- Call `Flush()` and allow time for transmission on shutdown
- Avoid creating multiple `InMemoryChannel` instances

Hmm, then maybe I should check how many `TelemetryClient` instances there are.

---

## The Silent Killer: Missing Configuration

There were several places that created an instance of `TelemetryConfiguration` with `.CreateDefault()`. This creates an instance of `TelemetryConfiguration` with the default setting found in `ApplicationInsights.config`. What if such a file does not exist? It will create an **empty configuration**.

An instance that does not have a **connection string** nor **instrumentation key** to send to AI. So all the threads are waiting for data to be sent, just nowhere to send it, causing a massive amount of threads and a huge allocation on unmanaged memory (stack space for each thread).

This was confirmed with this debug output:

```

0:023\> \!do 7604f69db0a0
Name:        Microsoft.ApplicationInsights.Extensibility.TelemetryConfiguration
...
Fields:
...
00007644edead7c8  40000bb       28        System.String  0 instance 00007604e18b3c00 instrumentationKey
00007644edead7c8  40000bc       30        System.String  0 instance **0000000000000000 connectionString**
...
0:023\> \!DumpObj /d 00007604e18b3c00
Name:        System.String
...
String:      325550d4-7bfd-4fbf-a5f8-e253b4120426

````

The `connectionString` was null, despite an instrumentation key (which was likely not valid in the context).

### The Mechanics of the Leak

Now this is the fun part. Whenever the customer sends a telemetry event to AI, it will be sent **async**. A new thread will be spawned, either new or from the threadpool, to send the request, and then return to the threadpool waiting to be awakened again.

**BUT**, and that's a big but, if there is no configuration, the `InMemoryTransmitter` does not know where to send the telemetry to. Ending up **waiting forever** via `WaitOneCore`. Because it does not return, the next time, the CLR will spawn a new thread, just for it to end up with the same fate.

![All of them were waiting for a signal](/assets/img/2025-10-09-death-by-thousands-threads/20251009123403.png)

After a while, you get a few thousands of live threads but doing nothing.

## Resolution

It is bad in both ends:

1.  The `InMemoryTransmitter` should have thrown an exception or timed out if it couldn't send the telemetry.
2.  There should be a proper configuration for AI.

The quick solution in this case is to set the `ConnectionString` properly in code, something like:

```csharp
_configuration.ConnectionString = Configuration["ApplicationInsights:ConnectionString"];
````

Moral of story is that you fail to do something, better to throw an exception for it, than just waiting silently