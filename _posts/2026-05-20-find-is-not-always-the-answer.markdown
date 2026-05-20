---
layout: post
title: "Find is not an answer for everything"
date: 2026-05-20 07:00:00 +0000
categories: [optimizely, episerver, find, performance]
tags: [episerver, optimizely, find, performance]
author: vimvq1987
---

You're often told that Find (Officially Search & Navigation, but most of us just call it Find because it's shorter to speak and to write ;)) is great and you should be using Find as much as possible, off load database access because database access is slower. That is true to some extend, but not always correct. Find is great when you need to search for content, especially in a free text search manner. It's much more efficient when you have a keyword and need to find matching content for it, using Find will be magnitude faster than loading the content from database and do a check on every content, and still even much faster and more efficient than query directly from database. You can leverage it when you are looking for the content. For loading the content, however, it's a different story.

Yesterday I started my day with a support case when a customer website went down. A memory dump is provided by my colleagues, so what would I do except making a latte while waiting for the memory dump to be downloaded? When it's done, I fired our custom built tool to analyze memory dumps, but it choked. So I opened the tried and true Windbg to do some manual steps. It becomes obvious that a lot of 1800 threads in that memory dump was in this similar stacktrace


```
  [HelperMethodFrame_1OBJ] (System.Threading.Monitor.ReliableEnter)
  System.Linq.Expressions.Compiler.DelegateHelpers.MakeDelegateType(System.Type[])
  System.Linq.Expressions.Expression.Lambda(System.Linq.Expressions.Expression, System.String, Boolean, System.Collections.Generic.IEnumerable`1<System.Linq.Expressions.ParameterExpression>)
  EPiServer.Find.FilterExpressionParser.GetFilterFromDelegateFilterBuilderMethod(System.Linq.Expressions.MethodCallExpression, System.String)
  EPiServer.Find.FilterExpressionParser.<GetFilter>b__2_1[[System.__Canon, System.Private.CoreLib]](System.Linq.Expressions.MethodCallExpression)
  EPiServer.Find.Helpers.Linq.ExpressionVisitor.VisitUnary(System.Linq.Expressions.UnaryExpression)
  EPiServer.Find.Helpers.Linq.ExpressionVisitor.VisitBinary(System.Linq.Expressions.BinaryExpression)
  EPiServer.Find.FilterExpressionParser.GetFilter[[System.__Canon, System.Private.CoreLib]](System.Linq.Expressions.Expression`1<System.Func`2<System.__Canon,EPiServer.Find.Api.Querying.Filter>>)
```

If you've worked with LamdbaExpression before, you'd know it's a powerful tool, with a caveat - it's extremely CPU heavy. Find relies heavily on LambdaExpression - and it cached compiled LambdaExpression to boost performance, but if there is too much of it, still a huge bottleneck. I peeked at the code that called Find, and it looks like this

```
  public TPageData GetPage<TPageData>(int id) where TPageData : PageData
  {
    IContentResult<TPageData> contentResult = SearchExtensions.StaticallyCacheFor<TPageData>(SearchExtensions.Take<TPageData>(SearchExtensions.Skip<TPageData>(this.SearchClient.Search<TPageData>().FilterForVisitor<TPageData>(), 0), 1).Filter<TPageData>((Expression<Func<TPageData, Filter>>) (p => p.ContentLink.ID.Match(id))), this.CacheTimespan).GetContentResult<TPageData>();
    IEnumerable<TPageData> items = contentResult.Items;
    return (items != null ? (items.Count<TPageData>() == 1 ? 1 : 0) : 0) == 0 ? default (TPageData) : contentResult.Items.First<TPageData>();
  }
```

There is a few oops with this
- The code loads a page with a known id, which could simply be done by `IContentLoader`
- FilterForVisitor will run through every filter, and they can be very heavy on LambdaExpression, and that is not cached, even if StaticallyCacheFor is used.

When it is confirmed that `FilterForVisitor` is not needed for the specific path, we advised the customer to use `IContentLoader` instead. And to demonstrate how much the impact is, I (asked Gemini to) write this benchmark test, and the difference is staggering

```
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Configs;
using BenchmarkDotNet.Jobs;
using BenchmarkDotNet.Order;
using BenchmarkDotNet.Toolchains.InProcess.Emit;
using EPiServer.Find;
using EPiServer.Find.Cms;
using EPiServer.Find.Framework;

namespace Foundation
{

    namespace MyProject.Benchmarks
    {
        public class CustomBenchmarkConfig : ManualConfig
        {
            public CustomBenchmarkConfig()
            {
                // 1. Disable the optimization validation check for your ODP package
                Options |= ConfigOptions.DisableOptimizationsValidator;

                // 2. Force BenchmarkDotNet to run code inside the SAME process 
                // so it has access to the already booted Optimizely IoC Container
                AddJob(Job.Default.WithToolchain(InProcessEmitToolchain.Instance));

                // Load standard output formatting rules
                Add(DefaultConfig.Instance);
            }
        }

        [Config(typeof(CustomBenchmarkConfig))]
        [MemoryDiagnoser]
        [Orderer(SummaryOrderPolicy.FastestToSlowest)]
        [RankColumn]
        public class ContentLoadBenchmark
        {
            private IContentLoader _contentLoader;
            private IClient _findClient;
            private ContentReference _targetReference;
            private string _targetSearchId;

            [GlobalSetup]
            public void Setup()
            {
                // Resolve services from the Optimizely IoC container
                _contentLoader = ServiceLocator.Current.GetInstance<IContentLoader>();
                _findClient = SearchClient.Instance;

                // Target Content ID (Replace with a valid page/block ID from your DB/Index)
                _targetReference = new ContentReference(105);
                _targetSearchId = _targetReference.ToString();
            }

            [Benchmark(Baseline = true)]
            public PageData GetViaContentLoader()
            {
                // Fetches directly from the L1/L2 Cache (Fastest)
                return _contentLoader.Get<PageData>(_targetReference);
            }

            [Benchmark]
            public PageData GetViaFindHydrated()
            {
                // Queries Find over HTTP, then hooks into CmsHydration to load from Cache/DB
                var result = _findClient.Search<PageData>()
                    .Filter(x => x.ContentLink.ID.Match(_targetReference.ID))
                    .Take(1)
                    .GetContentResult();

                return result.First();
            }

            [Benchmark]
            public PageData GetViaFindWithFilterForVisitors()
            {
                // Queries Find over HTTP, then hooks into CmsHydration to load from Cache/DB
                var result = _findClient.Search<PageData>()
                    .Filter(x => x.ContentLink.ID.Match(_targetReference.ID))
                    .FilterForVisitor()
                    .Take(1)
                    .GetContentResult();

                return result.First();
            }

            [Benchmark]
            public PageData GetViaFindWithFilterForVisitorsAndCache()
            {
                // Queries Find over HTTP, then hooks into CmsHydration to load from Cache/DB
                var result = _findClient.Search<PageData>()
                    .Filter(x => x.ContentLink.ID.Match(_targetReference.ID))
                    .FilterForVisitor()
                    .StaticallyCacheFor(TimeSpan.FromMinutes(10))
                    .Take(1)
                    .GetContentResult();

                return result.First();
            }
        }
    }
}
```

It's not even close. `IContentLoader`, as cached, is super efficient in loading the content, much faster than using Find. The actual culprit is `FilterForVisitor` as we suspected as it's extremely CPU heavy. Adding cache even makes it somewhat worse as it increases tail end while not improving the bottleneck!


| Method                                  | Mean         | Error       | StdDev       | Median       | Ratio  | RatioSD | Rank | Allocated | Alloc Ratio |
|---------------------------------------- |-------------:|------------:|-------------:|-------------:|-------:|--------:|-----:|----------:|------------:|
| GetViaContentLoader                     |     726.8 ns |    14.45 ns |     35.18 ns |     715.4 ns |   1.00 |    0.07 |    1 |     592 B |        1.00 |
| GetViaFindHydrated                      |  43,691.3 ns |   869.98 ns |  2,050.63 ns |  43,084.2 ns |  60.25 |    3.96 |    2 |   72473 B |      122.42 |
| GetViaFindWithFilterForVisitors         | 348,085.7 ns | 6,623.61 ns |  7,627.75 ns | 347,265.6 ns | 479.97 |   24.51 |    3 |  469628 B |      793.29 |
| GetViaFindWithFilterForVisitorsAndCache | 362,478.5 ns | 7,219.97 ns | 16,296.67 ns | 361,481.3 ns | 499.82 |   32.17 |    3 |  470012 B |      793.94 |

Lesson learned - always use the right tool for the right job. Find is a great tool, but it's not the every-problem solution. That can backfire badly!