---
layout: post
title: "Don't Insert an IEnumerable to Cache"
author: vimvq1987
date: 2020-07-05 10:00:00 +0200
categories: [cms, commerce, episerver]
tags: [caching, performance, dotnet, ienumerable, lazy-loading]
---

# Don’t Insert an IEnumerable to Cache

You have been told, cache is great. If used correctly, it can greatly improve your website performance; sometimes, it can even make the difference between life and death.

While that’s true, cache can be tricky to get right. The #1 issue with cache is cache invalidation, which we will get into detail in another blog post. The topic of today is a hidden, easy-to-make mistake, but one that can wreck havoc in production.

Can you spot the problem in this snippet?

```csharp
var someData = GetDataFromDatabase();                
var dataToCache = someData.Concat(someOtherData);
InsertToCache(cacheKey, dataToCache);
````

If you can’t, don’t worry – it is more common than you’d imagine. It’s easy to think that you are inserting your data correctly to cache.

Except you are not.

`dataToCache` is actually just an **enumerator** (`IEnumerable<T>`). It’s not until you get your data back and actually access the elements that the enumerator is called to fetch the data. If `GetDataFromDatabase` does not return a `List<T>`, but a lazy loading collection, that is when unpredictable things happen.

Who likes to have unpredictability on a production website? 💥

A simple, but effective advice is to always make sure you have the **actual data** in the object you are inserting to cache. Calling either **`.ToList()`** or **`.ToArray()`** before inserting the data to cache would solve the problem nicely.

```csharp
var someData = GetDataFromDatabase();                
var dataToCache = someData.Concat(someOtherData).ToList(); // Fix applied here!
InsertToCache(cacheKey, dataToCache);
```

And that’s applied to any other lazy loading type of data as well.

