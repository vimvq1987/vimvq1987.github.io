---
layout: post
title: "Solving the Mystery of High Memory Usage"
date: 2025-10-02 12:33:32 +0200
categories: [debugging, memory, caching]
tags: [dotnet, windbg, optimizely, find, memory-dump]
---

# Solving the Mystery of High Memory Usage

Sometimes, my work is easy, the problem could be resolved with one look (when I’m lucky enough to look at where it needs to be looked, just like this one [Varchar can be harmful to your performance – Quan Mai’s blog](https://vimvq1987.com/)). Sometimes, it is hard. Can’t count number of times that I stared blankly at the screen, and decided I’d better take a nap, roast a batch of coffee, or take a walk (that is lying, however, I don’t walk), because I’m out of idea and this is going nowhere. The life of a software diagnostic engineer is like that, sometimes you are solving the mystery of “what do I need to solve this mystery”. There are usually more dots scattered around in all places, your job is to figure out which dots make senses, which dots do not, and how to connect those that are relevant to solve the problem, and to tell a story.

The story today is about a customer complaining about their scheduled instance on **DXP** keeps having **high memory** after running the **Find indexing job**. They have a custom job that was built to optimize performance for their language settings, but the idea is the same – load content, serialize it and send it to the server endpoint for indexing. It is, indeed a memory heavy job, especially when you have a lot of content that needs to be indexed (basically, number of content x number of languages x the complexity of the content). It is normal to have an increase in memory usage during such job – the application (or rather, the runtime, depending on which way you look at it) is doing it job – content needs to be loaded in memory, and if there is available memory it will be a huge waste if it is not used for something useful. And the application will not immediately release that memory, as the content is cached. The memory will only be reclaimed only if the cache expired, or the application has memory pressure (i.e. it asks the operating system for more memory and the OS refuses “there is nothing left”). Even if the cache is expired, the application will not always compact and release the memory back to the OS (LOH etc.)

Now what is problematic is that the customer application retains **25GB of memory for indefinitely**. They waited for 24h but the memory usage is still high. The application appears to be fine, it does not crash because of memory issues (like Out of Memory), but it causes confusions and worries to our customer. Game’s on.

---

## The Misleading Cache Expiration

One thing that does not make senses in this case is that even thought they have a custom index job, it is still a scheduled job. And for scheduled jobs, the contents are supposed to have a very short **sliding expiration time** (**default to 1 minute**). However, the cache entries in the memory dumps tell a different story. A majority of the cache entries have **12h sliding expiration time**. Which does explain – in part at least – why the memory remains high. When you have a longer sliding time, chance is higher that the cache is hit at least once before it expires, which reset the expiration. If you have sufficient hit, the cache will effectively remain in memory forever, until you actively evict it (by editing the content for example)

```

0000753878028910                        0.77kb          0                           12:00:00                    2/16/2024 5:58:43 AM +00:00    EPPageData:601596:en\_\_CatalogContent
0000753878029DC0                        0.78kb          0                           12:00:00                    2/16/2024 2:59:39 PM +00:00    EPPageData:1345603:es-pr\_\_CatalogContent
00007538781C7F48                        0.78kb          0                           12:00:00                    2/16/2024 2:59:39 PM +00:00    EPPageData:1351986:es-pr\_\_CatalogContent
...
00007538787B2E68                        0.77kb          0                           12:00:00                    2/16/2024 2:17:07 PM +00:00    EPPageData:1329808:da\_\_CatalogContent
00007538787B31E8                        0.77kb          0                           12:00:00                    2/16/2024 2:17:07 PM +00:00    EPPageData:1329810:da\_\_CatalogContent

```

It is not what it should be, however, as the default value for sliding expiration timeout of a content loaded by a scheduled job is **1 minute** – i.e. it is considered to be load once and be done item. Was it set to 12h by mistake. Nope.

![alt text](/assets/img/windbg.png)

Timeout is set to `600.000.000` ticks which is **60 second**, which is the default value.



I have been pulling my hairs over this for quite a while. What if the cache entries were not added by the scheduled job, but by some other way not affected by the limitation of scheduled job? In short, we were deceived by customer’s statement regarding Find indexing job. It was merely a victim of same issue. It was resetting the last access to the cache entry but that’s about it.

---

## Finding the True Source of the Content Load

Time to dig a bit more. While **Windbg** is extremely powerful, it does not let you know where is the code that load a specific content into cache (not unless you catch it red handed). So the only way to know is to look around and check if there are any suspicious call to `IContentLoader.GetItems` or `IContentLoader.GetChildren`. A colleague of mine worked with the customer to obtain their source code, and another deep dive.

Fortunately for us, the customer has a custom built Find indexer we helped to built in a previous problem, and that was shown in the search for `GetItems`. It struck me that it could be the culprit. The job itself is … fine, however it was given wrong data so it keeps loading content to index.

If my hypothesis is correct, then these things must be true:

* The app’s memory usage will raise to 25GB regardless of the indexing job running or not. And it remains there without much fluctuation.
* There are a lot of row in `tblFindIndexQueue`.

It turned out both of those were correct: there were more than **4 millions of rows in `tblFindIndexQueue`**, and this is the memory consumption of the app over 24 hours.

![alt text](/assets/img/memory-chart.png)

Once we figured out the source of content loading, the fix was pretty straightforward. One thing we could do from our side is to shorten caching time of content loaded by the event-driven indexer. You should upgrade to **Find 16.2.0** which contains the fix for **FIND-12436** which is a nice improvement for memory usage.

---

## Moral of Story

* I’m a workaholic. I definitely should not work on weekends, but sometimes I need to because that’s when my mind is clearest.
* **Keep looking**. But as always, know when to give up and admit defeat.
* **Take breaks**. Long, shorts. Refresh your mind and look at different angles.
* **The sliding cache expiration time can be quite unexpected.** If a content is already in cache with long sliding expiration, then a cache hit (via `ISynchronizedObjectInstanceCache.ReadThrough`) to get that content with short sliding expiration will **not change that value**, only refresh the last access time, and vice versa.