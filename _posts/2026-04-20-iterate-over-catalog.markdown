---
layout: post
title: "How to iterate over catalog"
date: 2026-04-20 07:00:00 +0000
categories: [optimizely, episerver, commerce]
tags: [episerver, csharp, commerce, scheduled-jobs]
author: vimvq1987
---

This is a snippet of code that I've used more times than I care to admit, but every time I have to look at my book at https://leanpub.com/epicommercerecipes - the chapter is free and can be read without buying the book. However that is quite inconvenient, so I think it is better if I put it to my blog for later usages - and maybe it helps you too.

Recently we got a question from a customer how to update catalog assets that were imported from a DAM which has some incorrect information. That can, in theory, be done, by a SQL script, but I strongly advise against it. Iterating through the catalog using content API is much more safe, proven method that guaranteed to work. We can easily turn that snippet into the scheduled job. The main part of this post is the implementation of that job which is fully stoppable and uses batch loading to handle large catalogs efficiently.

## The Implementation

This job iterates through the commerce catalog, identifies PDF assets, and moves them into a specific "Download" group. It implements the `Stop()` method and checks the `_stopPressed` flag within every loop and recursive call to ensure the job shuts down gracefully when requested.

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using EPiServer.Commerce.Catalog.ContentTypes;
using EPiServer.Core;
using EPiServer.DataAccess;
using EPiServer.PlugIn;
using EPiServer.Scheduler;
using EPiServer.Security;
using Mediachase.Commerce.Catalog;

namespace YourProject.Infrastructure.Indexing
{
    [ScheduledPlugIn(
        DisplayName = "Catalog Asset Updater",
        Description = "Updates PDF media assets to the 'Download' group.",
        SortIndex = 100)]
    public class CatalogAssetUpdater : ScheduledJobBase
    {
        private readonly IContentRepository _contentRepository;
        private readonly ReferenceConverter _converter;
        private bool _stopPressed;

        public CatalogAssetUpdater(IContentRepository contentRepository, ReferenceConverter converter)
        {
            IsStoppable = true;
            _contentRepository = contentRepository;
            _converter = converter;
        }

        public override string Execute()
        {
            var processed = 0;
            var rootLink = _converter.GetRootLink();
            var catalogs = _contentRepository.GetChildren<CatalogContent>(rootLink);

            foreach (var catalog in catalogs)
            {
                // Check for cancellation
                if (_stopPressed) return ShutdownMessage(processed);

                OnStatusChanged($"Processing catalog: {catalog.Name}");

                var entries = GetEntriesRecursive<EntryContentBase>(catalog.ContentLink, CultureInfo.GetCultureInfo(catalog.DefaultLanguage));

                foreach (var entry in entries)
                {
                    // Check for cancellation
                    if (_stopPressed) return ShutdownMessage(processed);

                    if (entry != null && entry.CommerceMediaCollection.Any())
                    {
                        var writeableClone = entry.CreateWritableClone<EntryContentBase>();
                        bool wasUpdated = false;

                        foreach (var item in writeableClone.CommerceMediaCollection)
                        {
                            var asset = _contentRepository.Get<MediaData>(item.AssetLink);
                            if (asset.Name.EndsWith(".pdf", StringComparison.OrdinalIgnoreCase))
                            {
                                item.GroupName = "Download";
                                wasUpdated = true;
                            }
                        }

                        if (wasUpdated)
                        {
                            _contentRepository.Save(writeableClone, SaveAction.Publish, AccessLevel.NoAccess);
                            processed++;
                        }
                    }

                    if (processed % 10 == 0)
                    {
                        OnStatusChanged($"Processed {processed} entries...");
                    }
                }
            }

            return $"Successfully processed {processed} entries";
        }

        public override void Stop()
        {
            _stopPressed = true;
        }

        private string ShutdownMessage(int count)
        {
            return $"Job stopped by user. Processed {count} entries before stopping.";
        }

        public virtual IEnumerable<T> GetEntriesRecursive<T>(ContentReference parentLink, CultureInfo defaultCulture) where T : EntryContentBase
        {
            foreach (var nodeContent in LoadChildrenBatched<NodeContent>(parentLink, defaultCulture))
            {
                if (_stopPressed) yield break;

                foreach (var entry in GetEntriesRecursive<T>(nodeContent.ContentLink, defaultCulture))
                {
                    if (_stopPressed) yield break;
                    yield return entry;
                }
            }

            foreach (var entry in LoadChildrenBatched<T>(parentLink, defaultCulture))
            {
                if (_stopPressed) yield break;
                yield return entry;
            }
        }

        private IEnumerable<T> LoadChildrenBatched<T>(ContentReference parentLink, CultureInfo defaultCulture) where T : IContent
        {
            var start = 0;
            while (true)
            {
                if (_stopPressed) yield break;

                var batch = _contentRepository.GetChildren<T>(parentLink, defaultCulture, start, 50);
                if (!batch.Any()) yield break;

                foreach (var content in batch)
                {
                    if (!parentLink.CompareToIgnoreWorkID(content.ParentLink)) continue;
                    yield return content;
                }
                start += 50;
            }
        }
    }
}