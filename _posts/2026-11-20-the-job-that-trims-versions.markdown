---
layout: post
title: "The job that trims version"
date: 2026-05-29 07:00:00 +0000
categories: [optimizely, episerver, cms]
tags: [episerver, optimizely, cms, performance, scheduledjob]
author: vimvq1987
---

You probably know that a lot of versions is not really a good thing - you won't look back as far as last year, or the year before that. And while it's not really useful, it's costly. If you have many content in many languages, and each content has many version, the number of versions will add up quickly. And any features that rely on versions will be affected - like when some user look up the versions they've changed. To help deal with that, CMS has the Trim Content Version job to remove older versions that are probably not helpful to anyone. It works well and is recommended to run periodically to keep the version number in check. But there is a catch - it only trims previously published versions. So no CheckedIn, no Rejected. A colleague of mine has asked to make the change, because he saw the need of that, which I agree. But making such a change in feature requires Product Manager's signing off, and that takes time. What you can do, is to just write a scheduled job yourself to take care of the versions. And it's surprisingly easy to do so, especially with the help of AI.

And here it is. Because you are filtering by status rather than letting ListObsolete shrink the queue automatically, pay close attention to pagination: if a version fails to delete or is skipped, you must increment startIndex so the while loop doesn't get stuck re-fetching the exact same failing batch infinitely.

```
using EPiServer.Core;
using EPiServer.DataAbstraction;
using EPiServer.PlugIn;
using EPiServer.Scheduler;
using EPiServer.Security;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;

namespace YourProject.ScheduledJobs;

[ScheduledJob(
    GUID = "A1B2C3D4-E5F6-4789-8012-3456789ABCDE", // Generate a new unique GUID
    DisplayName = "Trim content versions by status",
    Description = "Deletes content versions matching configured statuses (excluding Published).",
    DefaultEnabled = false,
    IntervalLength = 1,
    IntervalType = ScheduledIntervalType.Days,
    Restartable = true)]
public class TrimContentVersionsByStatusJob : ScheduledJobBase
{
    private readonly ILogger<TrimContentVersionsByStatusJob> _logger;
    private readonly IContentVersionRepository _contentVersionRepository;
    private readonly CustomTrimContentVersionsOptions _options;
    private bool _stopSignaled;

    public TrimContentVersionsByStatusJob(
        IContentVersionRepository contentVersionRepository,
        IOptions<CustomTrimContentVersionsOptions> options,
        ILogger<TrimContentVersionsByStatusJob> logger)
    {
        _contentVersionRepository = contentVersionRepository;
        _options = options.Value;
        _logger = logger;
        IsStoppable = true;
    }

    public int VersionBatchSize { get; set; } = 100;
    public int MaxNumberOfFailures { get; set; } = 10;

    protected override void OnStatusChanged(string statusMessage)
    {
        _logger.LogDebug("{StatusMessage}", statusMessage);
        base.OnStatusChanged(statusMessage);
    }

    public override void Stop()
    {
        _stopSignaled = true;
    }

    public override string Execute()
    {
        // Enforce exclusion of Published versions regardless of configuration
        var targetStatuses = (_options.StatusesToDelete ?? Enumerable.Empty<VersionStatus>())
            .Where(status => status != VersionStatus.Published && status != VersionStatus.NotCreated)
            .Distinct()
            .ToList();

        if (targetStatuses.Count == 0)
        {
            const string noStatusMsg = "No valid non-published statuses configured to delete.";
            OnStatusChanged(noStatusMsg);
            return noStatusMsg;
        }

        var numberOfFailures = 0;
        var progressNumber = 0;
        var cutoffDate = _options.OlderThanDays.HasValue
            ? DateTime.UtcNow.AddDays(-_options.OlderThanDays.Value)
            : (DateTime?)null;

        foreach (var status in targetStatuses)
        {
            if (_stopSignaled || numberOfFailures > MaxNumberOfFailures)
            {
                break;
            }

            ProcessStatus(status, cutoffDate, ref progressNumber, ref numberOfFailures);
        }

        if (_stopSignaled)
        {
            OnStatusChanged("Stop was called.");
        }

        var msg = numberOfFailures == 0
            ? $"Removed {progressNumber} content versions."
            : $"Removed {progressNumber} content versions. Failed to remove {numberOfFailures} content versions.";

        OnStatusChanged(msg);
        return msg;
    }

    private void ProcessStatus(VersionStatus status, DateTime? cutoffDate, ref int progressNumber, ref int numberOfFailures)
    {
        var startIndex = 0;
        var statusNoticed = false;

        var filter = new VersionFilter
        {
            Status = status
        };

        while (!_stopSignaled && numberOfFailures <= MaxNumberOfFailures)
        {
            IEnumerable<ContentVersion> batch;
            int totalCount;

            try
            {
                batch = _contentVersionRepository.List(filter, startIndex, VersionBatchSize, out totalCount);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to list content versions for status {Status}.", status);
                numberOfFailures++;
                break;
            }

            var versions = batch.ToList();
            if (versions.Count == 0)
            {
                break;
            }

            if (!statusNoticed)
            {
                OnStatusChanged($"Found {totalCount} versions with status '{status}' to evaluate.");
                statusNoticed = true;
            }

            var skippedInBatch = 0;

            foreach (var version in versions)
            {
                if (_stopSignaled || numberOfFailures > MaxNumberOfFailures)
                {
                    break;
                }

                // Double-check safety guards: never delete WorkID 0 (master/unversioned) or Published
                if (version.ContentLink.WorkID == 0 || version.Status == VersionStatus.Published)
                {
                    skippedInBatch++;
                    continue;
                }

                // Optional age check
                if (cutoffDate.HasValue && version.Saved > cutoffDate.Value)
                {
                    skippedInBatch++;
                    continue;
                }

                if (progressNumber % 10 == 0)
                {
                    OnStatusChanged($"Deleted {progressNumber} content versions...");
                }

                try
                {
                    _contentVersionRepository.Delete(version.ContentLink, AccessLevel.NoAccess);
                    progressNumber++;
                }
                catch (Exception ex)
                {
                    numberOfFailures++;
                    skippedInBatch++;
                    _logger.LogError(ex, "Failed to delete content version {ContentLink} with status {Status}.", version.ContentLink, status);
                }
            }

            // If items were deleted, the remaining rows shift forward in the database.
            // Only advance startIndex by the number of items that remained in the table (skipped or failed).
            startIndex += skippedInBatch;

            if (versions.Count < VersionBatchSize)
            {
                break;
            }
        }
    }
}
```

The version statuses that are eligible for trimming can be configured by this class

```
using EPiServer.Core;

namespace YourProject.ScheduledJobs;

[Options(ConfigurationSection = "CustomTrimContentVersions")]
public class CustomTrimContentVersionsOptions
{
    public const string SectionName = "CustomTrimContentVersions";

    /// <summary>
    /// The statuses of content versions to delete. Published is always excluded by the job.
    /// </summary>
    public IEnumerable<VersionStatus> StatusesToDelete { get; set; } = new[]
    {
        VersionStatus.PreviouslyPublished,
        VersionStatus.CheckedOut,
        VersionStatus.Rejected
    };

    /// <summary>
    /// Optional: Only delete versions older than this number of days.
    /// </summary>
    public int? OlderThanDays { get; set; }
}
```

Voila. Now you can decide which statuses you want to trim!
