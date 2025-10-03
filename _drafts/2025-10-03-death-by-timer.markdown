---
layout: post
title: "Death by Timer: A Software Diagnostics Epic"
date: 2025-10-03 13:57:42 +0200
categories: [Software Diagnostics, Memory Leak, .NET, Debugging]
tags: [Windbg, sos.dll, MemoryCache, TimerQueueTimer, diagnostics]
---


# Death by Timer. Hundreds of thousands

Working as a **Software Diagnostics engineer** (a title I’ve given myself) can be fun or frustrating at the same time.

Sometimes, people come to you with a big, critical, urgent problem, and you get ready, adrenaline pumped in your vein, and you want to dive in headfirst to conquer what is troubling your customer. Just 5 minutes later, you already know what is wrong. Everyone is happy, except you — because the problem is solved, but not in the way you wanted it to. You want your battle to be epic, legendary, you want to tell your children and grandchildren, that once upon a time, you have battled the most fearsome beast, and you emerged victorious.

Sometimes, you realize the problem at hand is the most challenging, elusive, ever. But after hours and hours, if not night and night fighting with everything you had to figure out what is the root cause (and how to fix it), you have only yourself to give a tap on the shoulder. You’re satisfied but who else is there to celebrate? Then you have to tell your story, because you’re afraid, that the wisdom and lesson learned will be lost in time.

That’s why we have today's story.

This has been the most challenging problem I’ve been solving in recent memory. The case even exceeds [*A Curious Case of Memory Dump Diagnostics: How Stackify Can Cause Troubles to Your Site*](https://www.google.com/search?q=https://example.com/a-curious-case-of-memory-dump-diagnostics) that was the pinnacle of my career — pivoting my life from a software engineer to something else.

So, the story starts with a (not so) long ago, in a far, far way server. A customer reports issue with **memory increasing over time**.

That is definitely a sign of **memory leak**. I was not super interested at first because it was likely a simple problem. Boys, how wrong I was.

-----

## Initial Investigation: The Smoking Gun

The normal process of firing **Windbg**, load `sos.dll` and run the popular `!dumpheap -stat` command gave us this:

```
7b4add7ed7c8    278,550    64,519,413 System.Byte[]
7b4adf3d09c0    694,555    66,677,280 System.Threading.TimerQueueTimer
7b4aedd2e2e0    616,736   162,818,304 System.Runtime.Caching.ExpiresBucket[]
7b4adca1d7c8  1,725,481   194,779,498 System.String
7b4ae6b94ed0        326   200,765,144 System.WeakReference<System.Diagnostics.Tracing.CounterGroup>[]
7b4adca189e0 18,875,265   764,326,736 System.Int32[]
614373eb9d30    699,996   969,613,200 Free
7b4aedd2e270 18,502,080 1,776,199,680 System.Runtime.Caching.ExpiresBucket
Total 63,167,736 objects, 5,486,324,146 bytes
```

Look at the most memory consuming type. It’s `ExpiresBucket` that suggests this has something to do with **caching**. This is new to me (i.e. I’ve never touched it in the past) so let’s pick an object and examine it:

```
!dumpobj /d 7b0aeca1f628
Name:        System.Runtime.Caching.ExpiresBucket
MethodTable: 00007b4aedd2e270
EEClass:     00007b4aedda6d88
Tracked Type: false
Size:        96(0x60) bytes
File:        /app/System.Runtime.Caching.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007b4aedd2dca8  400002a        8 ...hing.CacheExpires  0 instance 00007b0aeca1e828 _cacheExpires
00007b4adc9c1a58  400002b       30          System.Byte  1 instance               24 _bucket
00007b4aedd2e518  400002c       10 ...ing.ExpiresPage[]  0 instance 0000000000000000 _pages
00007b4adc9a0980  400002d       20         System.Int32  1 instance                0 _cEntriesInUse
00007b4adc9a0980  400002e       24         System.Int32  1 instance                0 _cPagesInUse
00007b4adc9a0980  400002f       28         System.Int32  1 instance                0 _cEntriesInFlush
00007b4adc9a0980  4000030       2c         System.Int32  1 instance               -1 _minEntriesInUse
00007b4aedd2e200  4000031       34 ...g.ExpiresPageList  1 instance 00007b0aeca1f65c _freePageList
00007b4aedd2e200  4000032       3c ...g.ExpiresPageList  1 instance 00007b0aeca1f664 _freeEntryList
00007b4adc96cb00  4000033       31       System.Boolean  1 instance                0 _blockReduce
00007b4ade2a4e10  4000034       48      System.DateTime  1 instance 00007b0aeca1f670 _utcMinExpires
00007b4adca189e0  4000035       18       System.Int32[]  0 instance 00007b0aeca1f688 _counts
00007b4ade2a4e10  4000036       50      System.DateTime  1 instance 00007b0aeca1f678 _utcLastCountReset
00007b4adcb7eb10  4000029       18      System.TimeSpan  1   static 00007b0acc00ce80 s_COUNT_INTERVAL
```

The most interesting property here is probably `_cacheExpires`, so let's see what it has:

```
!DumpObj /d 00007b0aeca1e828
Name:        System.Runtime.Caching.CacheExpires
MethodTable: 00007b4aedd2dca8
EEClass:     00007b4aedda6be0
Tracked Type: false
Size:        56(0x38) bytes
File:        /app/System.Runtime.Caching.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007b4aedd2cdf8  400003c        8 ....MemoryCacheStore  0 instance 00007b0aeca1e700 _cacheStore
00007b4aedd2e2e0  400003d       10 ...g.ExpiresBucket[]  0 instance 00007b0aeca1e860 _buckets
00007b4aedd2eff0  400003e       18 ...Private.CoreLib]]  0 instance 00007b0aeca75ee8 _timerHandleRef
00007b4ade2a4e10  400003f       28      System.DateTime  1 instance 00007b0aeca1e850 _utcLastFlush
00007b4adc9a0980  4000040       20         System.Int32  1 instance                0 _inFlush
00007b4adcb7eb10  4000037       20      System.TimeSpan  1   static 00007b0acc00ce88 MIN_UPDATE_DELTA
00007b4adcb7eb10  4000038       28      System.TimeSpan  1   static 00007b0acc00ce90 MIN_FLUSH_INTERVAL
00007b4adcb7eb10  4000039       30      System.TimeSpan  1   static 00007b0acc00ce98 _tsPerBucket
00007b4adcb7eb10  400003b       38      System.TimeSpan  1   static 00007b0acc00cea0 s_tsPerCycle
```

So it points to a **MemoryCacheStore**. Interesting.

```
!DumpObj /d 00007b0aeca1e700
Name:        System.Runtime.Caching.MemoryCacheStore
MethodTable: 00007b4aedd2cdf8
EEClass:     00007b4aedda6190
Tracked Type: false
Size:        80(0x50) bytes
File:        /app/System.Runtime.Caching.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007b4add0c2918  40000df        8 ...ections.Hashtable  0 instance 00007b0aeca1e768 _entries
00007b4adc965ad0  40000e0       10        System.Object  0 instance 00007b0aeca1e810 _entriesLock
00007b4aedd2dca8  40000e1       18 ...hing.CacheExpires  0 instance 00007b0aeca1e828 _expires
00007b4aedd2de38  40000e2       20 ...aching.CacheUsage  0 instance 00007b0aeca75d48 _usage
00007b4adc9a0980  40000e3       40         System.Int32  1 instance                0 _disposed
00007b4adffd6718  40000e4       28 ....ManualResetEvent  0 instance 00007b0aeca75de0 _insertBlock
00007b4adc96cb00  40000e5       44       System.Boolean  1 instance                0 _useInsertBlock
00007b4aedd2c5f8  40000e6       30 ...ching.MemoryCache  0 instance 00007b0b06dd5508 _cache
00007b4aedd2d358  40000e7       38 ....Caching.Counters  0 instance 00007b0b06dd55b0 _perfCounters
```

At this point we have a list of `CacheEntries` it contains. So let’s see what is in there:

```
!DumpObj /d 00007b0aeca1e768
Name:        System.Collections.Hashtable
MethodTable: 00007b4add0c2918
EEClass:     00007b4add05c158
Tracked Type: false
Size:        72(0x48) bytes
File:        /usr/share/dotnet/shared/Microsoft.NETCore.App/8.0.20/System.Private.CoreLib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007b4add0c2bd0  4002075        8 ...ashtable+Bucket[]  0 instance 00007b0aeca1e7b0 _buckets
00007b4adc9a0980  4002076       28         System.Int32  1 instance                0 _count
00007b4adc9a0980  4002077       2c         System.Int32  1 instance                0 _occupancy
00007b4adc9a0980  4002078       30         System.Int32  1 instance                2 _loadsize
00007b4adc9cfa20  4002079       34        System.Single  1 instance 0.720000 _loadFactor
00007b4adc9a0980  400207a       38         System.Int32  1 instance                0 _version
00007b4adc96cb00  400207b       3c       System.Boolean  1 instance                0 _isWriterInProgress
00007b4adc96a3c0  400207c       10 ...tions.ICollection  0 instance 0000000000000000 _keys
00007b4adc96a3c0  400207d       18 ...tions.ICollection  0 instance 0000000000000000 _values
00007b4adcab7ec8  400207e       20 ...IEqualityComparer  0 instance 00007b0aeca1e750 _keycomparer
```

To our disappointment, this `MemoryCacheStore` is actually **empty** (as indicated by `_count: 0`). If we go back and pick another `ExpiresBucket` or follow its chain, it’s the same.

The problem just got much more interesting. Let’s see if we can find what is holding those `MemoryCacheStore` objects by the `!gcroot` command. `gcroot` can take a long time to traverse the memory, but I was not prepared for this



The problem was this registration

```
      services.AddScoped<RetailServerContext>((Func<IServiceProvider, RetailServerContext>) (serviceProvider =>
      {
        try
        {
          D365Settings settings = serviceProvider.GetService<IOptions<D365Settings>>().Value;
          IMicrosoftIdentityService microsoftIdentityService = serviceProvider.GetService<IMicrosoftIdentityService>();
          AuthenticationResult authenticationResult = AsyncHelper.ExecuteSync<AuthenticationResult>((Func<CancellationToken, Task<AuthenticationResult>>) (token => microsoftIdentityService.AcquireTokenForClient(settings.ApplicationId, settings.ApplicationSecret, settings.TenantId, settings.ResourceId, token)), new CancellationToken());
          return RetailServerContext.Create(new Uri(settings.CommerceScaleUnitUrl), settings.OUN, authenticationResult.AccessToken);
        }
        catch
        {
          return (RetailServerContext) null;
        }
      }));
```

Which means every time a RetailServerContext is requested, the new instance is created, and that create the chain of RetailServerContext => ETagContextCacheProvider => MemoryCacheProvider => MemoryCache
And while those RetailServerContext ETagContextCacheProvider and even MemoryCacheProvider was collected, MemoryCache linger arounds because they have Timer that keeps them alive. The results? Hundreds of thousands of MemoryCache, MemoryCacheStore, and their referenced objects alive