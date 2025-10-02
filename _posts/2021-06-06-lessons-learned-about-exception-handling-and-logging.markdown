---
layout: post
title: "Lessons Learned About Exception Handling and Logging"
author: vimvq1987
date: 2021-05-06 10:00:00 +0200
categories: [bestpractices, episerver]
tags: [exceptions, logging, debugging, csharp]
---

# Lessons Learned About Exception Handling and Logging

Exception handling and logging are essential parts of any site. Your site will eventually run into problems, and as a developer it’s your job to make sure that you have enough – or at least helpful – information to look into problems. From my years of diagnosing and root cause analysis, these are the most common problems about exception handling and logging. You (or any developers that come after you) can thank yourself later.

---

## 1. Empty `try-catch` is Almost Always Bad

In my early days of professional programming, I saw a colleague – several years senior to me, fortunately, not at Episerver – write a lot of this code, basically in each and every method:

```csharp
try { 
    //do stuffs
}
catch {}
````

You surely don’t want to show a YSOD (Yellow Screen of Death) to your visitors, but keep this in mind – this is almost always bad. You are trying to hide errors, and even worse, trying to **swallow it**. This should be a red flag for any code review.

There are cases when you truly, really want to swallow an exception. It’s rare, but it’s a reality. In such case, make sure to **explain why you have an empty catch in comments**.

-----

## 2\. Don’t Catch Exception Only to Throw It

Does this look familiar to you? If yes, then your code base have a problem (or actually, two):

```csharp
try 
{
    //do stuffs
}
catch (Exception ex)
{
    throw ex;
}
```

But why is this bad?

1.  It is wasting a `try-catch` doing nothing of value. Either log it, or wrap it in a different type of exception (with probably more information), but don't just rethrow the exception.
2.  `throw ex;` actually **resets the stacktrace**. If you simply want to rethrow the exception (after doing meaningful stuffs with it), use `throw;` instead. That will preserve the precious stacktrace for exception handling at higher level.

-----

## 3\. Logging Only the Message is a Crime

Someday, you will find an entry in your log that looks like this:

> Object reference not set to an instance of an object.

And that’s it. This is even worse than no message – you know something is wrong, but you don’t know why, or how to fix it. It’s important to always log the **full stacktrace** of the exception, instead of just the message, unless you have a very good reason (I can’t think of one, can you?) not to.

-----

## 4\. Verbose is Better Than Concise

This is an extension of the above lesson. You are already logging the entire stacktrace, but is your exception message helpful? One example is this:

```
[InvalidOperationException: There is already a content type registered with name: PdfFile]
   EPiServer.DataAbstraction.Internal.DefaultContentTypeRepository.ValidateUniqueness(ContentType contentType) +222
```

Which other type was registered with the same name? Is it a built-in (i.e. system) type, or a type from a 3rd party library, or a type that you added but forgot yourself?

In this case, a much better exception message would be:

> There are two or more content types registered with name PdfFile:
>
>   * `Assembly1.Namespace1.PdfFile`
>   * `Assembly2.Namespace2.PdfFile`

Exception message is not only showing an error/unwanted situation has happened, but it needs to be **helpful** as well. When you log a message, you should try to add as much information as you can so the ones who will be looking into the issue can make a good guess of what is wrong, and **how to fix it**. There is virtually no limit on how verbose you can be, so feel free to add as much information as you need (I’m not asking you to add the entire “War and Peace” novel here, of course). You can thank me later.

