---
layout: post
title: "Why you should care about what you return in your ScheduledJob.Execute"
date: 2025-10-02 11:09:50 +0200
categories: [performance, LOH, allocations]
tags: [performance, allocations, perfview]
---

# Why You Should Care About What You Return in Your ScheduledJob.Execute

First of all, welcome back to my little blog. I'm still in the process of moving the old posts that I deem read-worthy from my old blog archive to this new platform, but at least we have a platform to talk about Optimizely, Commerce, performance, troubleshooting, and a zillion things that go on in my life. I tend to keep this blog lean and simple this time, so welcome back, and enjoy.

This part is an extension of [Fix your Search & Navigation please!](https://vimvq1987.com/Fix-your-Search-Navigation-please/). While Search & Navigation indexing job is a built-in job, it is not the only thing that can wreck havoc on your website performance. In fact, **any job can do that by returning a long, long value from `ScheduledJobBase.Execute`**.

The `Execute` method is supposed to return the status of the job, but you can, in fact, return whatever you like into it, including anything and everything: every trace, every message, every warning, every error, etc. Because the result will be saved to a column `nvarchar(max)` in the database, which has a limitation of 2GB (roughly 1 billion characters), there will be no problem, right?

But yes, there is.

---

## The Double Whammy of `nvarchar(max)`

In the previous post, we explored how having a long, long string in the job's `LastText` can **hammer your database performance**. But the nightmare does not end there.

Because that long, long string will be loaded into memory at the application layer, and because it is long, long, it will be created in the **Large Object Heap (LOH)**.

Nobody really likes the Large Object Heap. Because it hurts your website performance when there are a lot of allocations in it. And here is a case when a scheduled job with a long, long `LastText` is repeatedly loaded into memory:

![A very unusual amount of LOH allocations]

You might think, "I'm not visiting the scheduled job interface a lot, I'd be fine." No, you won't. **As long as you have a job scheduled, every time it runs, the framework will list all registered scheduled jobs, and this will happen.** If you have a very frequently called job, like every few minutes, it will add up quickly.

> **Note:** Yes, this should be treated as a bug, and I have filed one for the CMS team to fix.

---

## Hindsight on `nvarchar(max)`

Hindsight 20/20, but I think letting any column be `nvarchar(max)` is a questionable design choice. This does not only apply to `tblScheduledItem`, but also any column in any table.

Sure, it is a very convenient way of storing data without worrying about it will someday break because you set a limit on the length, but without **proper validation and caching**, it opens a gate for misuse and abuses.

Want to define your column as `nvarchar(max)` (or the lesser sibling `varchar(max)`)? **Think twice!**
