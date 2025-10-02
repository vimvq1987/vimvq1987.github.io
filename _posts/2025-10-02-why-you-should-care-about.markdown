---
layout: post
title: "Why you should care about what you return in your ScheduledJob.Execute"
date: 2025-03-17 11:09:50 +0200
categories: [performance, LOH, allocations]
tags: [performance, allocations, perfview]
---

# Why you should care about what you return in your ScheduledJob.Execute

First of all, welcome back to my little blog. I'm still in the progress of moving the old posts that I deem read-worthy from my old blog archive to this new platform, but at least we have a platform to talk about Optimizely, Commerce, performance, trouble shooting and a zillion things that go on in my life. I tend to keep this blog lean, simple this time, so welcome back, and enjoy.

This part is an extension of https://vimvq1987.com/Fix-your-Search-Navigation-please/ . While Search & Navigation indexing job is a built in job, it is not something that can wreck havok on your website performance. In fact, any job can do that by returning a long, long value from `ScheduledJobBase.Execute`. It is supposed to return the status of the job, but you can, in fact, return whatever you like into it, including anything and everything, every traces, every message, every warning, every error, etc. etc. Because the result will be saved to a column `nvarchar(max)` in the database, which has a limitation of 2GB, which is roughly 1 billion characters, there will be no problem, right?

But yes, there is. In the previous post we explored how having a long, long string in `LastText` can hammer your database performance. But the nightmare does not end there. Because that long, long string will be loaded into memory at application layer, and because it is long, long, it will be created in Large Object Heap.

Nobody really like Large Object Heap. Because it hurts your website performance when there are a lot of allocation in it. And here is a case when a scheduled job with a long, long `LastText` is repeatedly loaded into memory

![A very unusual amount of LOH allocations](/assets/img/Loh.png)

You might think I'm not visiting the scheduled job interface a lot, I'd be fine. No you don't. As long as you have a job scheduled, every time it runs, the framework will list all registered scheduled jobs, and this will happen. If you have a very frequently called job, like every few minutes, it will add up quickly.

D> Yes this should be treated as a bug and I have filed one for the CMS team to fix. 

Hindsight 20/20, but I think letting any column `nvarchar(max)` is a questionable design choice. This does not only apply to tblScheduledItem, but also any column in any table. Sure, it is a very convenient way of storing data without worrying about it will someday break because you set a limit on the length, but without proper validation and caching, it opens a gate for misuse and abuses.

Want to define your column as `nvarchar(max)` (or the lesser sibling `varchar(max)`) ? Think twice!