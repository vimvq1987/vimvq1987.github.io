---
layout: post
title: "A Curious Case of Free Objects"
date: 2025-04-10
author: Quan Mai
categories: [Diagnostics, Memory Dump, .NET]
tags: [GC, LOH, memory leak, dumpheap, CustomMemoryCache]
excerpt: A memory dump investigation into rising memory usage and the mystery of free objects in the Large Object Heap.
---

It’s been a long time since I posted a memory dump investigation. Not that I haven’t done them, but it’s often hard to tell a good story. Many cases have a gotcha moment—exciting in context—but hard to share meaningfully when details are restricted or too technical.

Memory dumps are a powerful aid, but rarely the only tool. They show what the app was doing at the exact moment of capture, but for ongoing processes, it’s harder to pinpoint the cause. Still, they help narrow the scope of investigation.

---

## The Setup

A colleague reached out because a customer’s memory usage was steadily rising.

Normally, memory should stabilize over time. If it keeps climbing without plateauing, it’s a red flag. Since we were already engaged with this customer on other memory issues, we collected a dump. Game on.

What puzzled me immediately: **a lot of free objects** in the dump. That indicates a GC had occurred. But with so much freed memory, tracking allocations was difficult. We took another dump 30 minutes later—and it was the same.


7c343a686798    201,951     24,234,120 EPiServer.Core.Html.StringParsing.ContentFragment
7c34431d93a8     59,155     36,690,528 System.Collections.Generic.Dictionary<System.String, EPiServer.Core.PropertyData>+Entry[]
7c3438e63230    534,370     42,749,600 EPiServer.Core.PropertyLongString
7c3436362a28    202,652    107,559,201 System.Byte[]
7c34356d7c18    158,223  1,082,118,840 System.String[]
7c34355cd7c8 10,944,050  1,235,497,138 System.String
5b4faa00d1b0  8,039,645 11,560,389,656 Free
Total 28,779,604 objects, 14,685,150,235 bytes
What could potentially create that many free objects and leave them floating around. And memory keeps raising? The free objects indicating that there have been some heavy Garbage collection. But we can’t track where those come from because they were “freed”, rendering !gcroot useless.

I was pulling my hairs for a while (not that I have that many hairs left to pull). But then I realize, could these be big objects allocated in LOH (Larger objects heap). If they are big enough (more than 85.000 bytes in size), they will not be be freed when no longer referenced, but they are not “moved” (e.g. compacted).

0:000> !dumpheap -stat -mt 5b4faa00d1b0 -min 85000
Statistics:
          MT Count   TotalSize Class Name
5b4faa00d1b0    53 237,914,208 Free
Total 53 objects, 237,914,208 bytes
Hmm, we have 53 big object with more than 200MB in size, so averaging around 4MB per object. Let’s see the biggest objects by removing the -stat option

0:000> !dumpheap -mt 5b4faa00d1b0 -min 85000
         Address               MT           Size
    7bf42b10e140     5b4faa00d1b0         97,264 Free
    7bf42bd1bb50     5b4faa00d1b0      2,319,816 Free
    7bf42c8521a0     5b4faa00d1b0        586,096 Free
    7bf42ca8d448     5b4faa00d1b0         91,064 Free
    7bf42d71bdd8     5b4faa00d1b0      4,396,576 Free
    7bf42dc0a5a8     5b4faa00d1b0      2,428,160 Free
    7bf42e041680     5b4faa00d1b0      4,639,520 Free
    7bf42faad6e8     5b4faa00d1b0      2,527,600 Free
    7bf43057f0c8     5b4faa00d1b0      2,857,968 Free
    7bf431c98d60     5b4faa00d1b0      2,609,160 Free
    7bf4321c2a08     5b4faa00d1b0      4,194,392 Free
    7bf4325e2a78     5b4faa00d1b0      3,153,664 Free
    7bf43a747550     5b4faa00d1b0        662,736 Free
    7bf44c90f528     5b4faa00d1b0     13,684,008 Free
    7bf44da5b780     5b4faa00d1b0      3,578,672 Free
    7bf484525ad8     5b4faa00d1b0        386,352 Free
    7bf4845c8710     5b4faa00d1b0     15,650,872 Free
    7bf485522248     5b4faa00d1b0      2,131,104 Free
    7bf489000028     5b4faa00d1b0      5,767,368 Free
    7bf489597aa8     5b4faa00d1b0      2,568,696 Free
    7bf4b9c00028     5b4faa00d1b0      6,029,488 Free
    7bf4bc000060     5b4faa00d1b0     12,785,496 Free
    7bf4bcc517d0     5b4faa00d1b0    117,106,864 Free
    7bf4c7400028     5b4faa00d1b0      2,854,216 Free
    7bf4c7ae2930     5b4faa00d1b0      2,180,536 Free
    7bf4c7d0d260     5b4faa00d1b0      3,539,256 Free
    7bf4d7000028     5b4faa00d1b0     14,066,712 Free
    7bf4d7e0b350     5b4faa00d1b0        871,840 Free
The biggest object is more than 100MB. Could there be any big objects that were alive of that size? Let’s go with !dumpheap -min 100000000 to see if we find any

!dumpheap -min 100000000
         Address               MT           Size
    7bf4bcc517d0     5b4faa00d1b0    117,106,864 Free
    7bf724c00048     7c34356d7c18  1,073,741,848 

Statistics:
          MT Count     TotalSize Class Name
5b4faa00d1b0     1   117,106,864 Free
7c34356d7c18     1 1,073,741,848 System.String[]
Total 2 objects, 1,190,848,712 bytes
The result turned out to be better than expected. We found two object – one is the biggest free object, and the one one is an array of string with size of almost 1GB. It has 134217728 elements, which is … a lot. (But also a lot of empty elements, more below)

0:000> !dumpobj /d 7bf724c00048
Name:        System.String[]
MethodTable: 00007c34356d7c18
EEClass:     00007c343551bee0
Tracked Type: false
Size:        1073741848(0x40000018) bytes
Array:       Rank 1, Number of elements 134217728, Type CLASS (Print Array)
Fields:
None
Let’s hope this is not a free object waiting to be collected. !gcroot 7bf724c00048 might give us some clue

          -> 7bf42810a4a8     Microsoft.Extensions.Http.DefaultHttpClientFactory 
          -> 7bf429868210     Microsoft.Extensions.DependencyInjection.ServiceLookup.ServiceProviderEngineScope 
          -> 7bf429882b78     System.Collections.Generic.List<System.Object> 
          -> 7bf429cf3280     System.Object[] 
          -> 7bf4299a5550     SomeNamespace.Services.CustomMemoryCache 
          -> 7bf4299a55a8     System.Collections.Generic.List<System.String> 
          -> 7bf724c00048     System.String[] 
A bit of dotPeek reveals the code

public class CustomMemoryCache : MemoryCache, IMemoryCache, IDisposable
  {
    private List<string> _cacheNames = new List<string>();

    public List<string> GetKeys() => this._cacheNames;

    public new ICacheEntry CreateEntry(object key)
    {
      this._cacheNames.Add(key.ToString());
      return base.CreateEntry(key);
    }
As you can see the _cacheNames will add the cache key to a list. Because List<T> is not unique, whenever a new cache item is added, the key will be added to the list (even if it was a duplicate), so it becomes ever growing. In the previous memory dump (that was taken 30 minutes earlier), there were 81928424 entries in the list. As this is strongly referenced, the list will remain forever and only increases over time.

Now the funny part – because the List<T> has a limit of 2GB on the array, but because string is a reference types, each element in the array is simply a reference of it – so 8 bytes per element. You can see that is a lot. Not only that, there are two things to know about List<T>

List<T> will create a new array, double in size copy the items over once it’s filed the internal array with items. The old array will become unreachable and ready to be GC’ed the next time. Which is also why we might have a lot of empty items in the list.
Once the array grows large enough, they will be created in the Large Object Heap (LOH). The threshold is 85.000 bytes, so around 10k items in the list (because each element is 8 bytes, they do not hold the actual string object, but a reference to it). After that, each time the list is doubled, a new object will be created, and the old list will remain on LOH as a free memory object. They will (almost) never be compacted, unless there was a need to create a big object in which they will be overwritten.
The fix in this case is to simply remove _cacheNames, as it does not serve a purpose. It took a while for the customer to deploy the fix, but when they do, the result was … impressive


Before the fix, the memory was ever increasing until the app needs to be restarted. After that, it remain below 3GB.

Moral of the story

Keep looking.
If there are a lot of big, free objects, most likely, there are live objects of equal size that can give you some ideas.
