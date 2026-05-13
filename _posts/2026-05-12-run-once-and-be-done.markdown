---
layout: post
title: "Run once and be done"
date: 2026-04-20 07:00:00 +0000
categories: [optimizely, episerver, commerce]
tags: [episerver, csharp, commerce, scheduled-jobs]
author: vimvq1987
---

Sometimes your website need some data migration. There should be a way to run it after deployment but before the site performs any other tasks. There are several ways to do it, but IMO it could best be handled with `IMigrationStep`. It is an hidden/undocumented feature that allows data migration. Each step is supposed to run once at the first start up to ensure data migration. 

## The Implementation

This is an example of implementing an `IMigrationStep`. It has a few properties that are quite self-explanatory. Only note that Order value determines how early or late your migration step will be - step with lower Order value will be executed first. So if your step depends on some other steps, make sure they have the right Order. The center part of a migration step is the `Execute` method. It takes an `IProgressMessenger` which allows you to send messages to update the progress to the Migration UI. Also `[ServiceConfiguration(typeof(IMigrationStep))]` attribute is important so the Migration manager knows this is a step to execute in the next run. Here is an example of adding a new field to the `Contact` metaclass:

```csharp

    [ServiceConfiguration(typeof(IMigrationStep))]
    public class AddContactBirthdayStep : IMigrationStep
    {
        public int Order => 2000;

        public string Name => "Add birthday to contact";

        public string Description => "Add birthday to contact";

        public bool Execute(IProgressMessenger progressMessenger)
        {
            using (MetaClassManagerEditScope scope = DataContext.Current.MetaModel.BeginEdit(MetaClassManagerEditScope.SystemOwner, AccessLevel.System))
            {
                MetaClass mc = DataContext.Current.MetaModel.MetaClasses[ContactEntity.ClassName];
                using (MetaFieldBuilder builder = new MetaFieldBuilder(mc))
                {
                    builder.CreateDate("Birthday", "Birthday", true, true);
                    builder.SaveChanges();
                }

                scope.SaveChanges();
            }
            return true;
        }
    }

```

Migration steps are supposed to run only once, if a step has completely successfully it will not be run again. However, it's always a good idea to make sure your step can be run multiple times without problem (e.g. making it idempotent) when applicable. In this specific example, a simple check is that you can see if the Contact class already have the `Birthday` field or not, and stop further processing if that's the case. For example here is how to do it 

```
    MetaClass mc = DataContext.Current.MetaModel.MetaClasses[ContactEntity.ClassName];
    if (mc.Fields.Contains("Birthday"))
    {
        return true;
    }
```

While `IMigrationStep` is a great (again, hidden) tool to run your data migration once, and importantly, in one threading manner, it does not guarantee that only one instance will run it. There is a possibility when your set up has multiple instances and during start up, many of them might try to do the same thing. You might want to ensure your data migration is run on exactly one instance only. There is no built in mechanism for such, but fortunately there are 3rd party library to achive that: DistributedLock.SqlServer. You can download it from https://www.nuget.org/packages/DistributedLock.SqlServer
The idea is straightforward - the library attempts to create a special value in database, if successfully, it is the first to do so and will hold that lock, with periodically pings to ensure that it is still "alive". If the key exists meaning some other instances already are doing that, so it will just wait or exit. 

```
using EPiServer.Commerce.Internal.Migration.Steps;
using EPiServer.ServiceLocation;
using Medallion.Threading.SqlServer;
using Mediachase.BusinessFoundation.Data;
using Mediachase.BusinessFoundation.Data.Meta.Management;
using Mediachase.Commerce.Customers;
using Mediachase.Commerce.Shared;
using Mediachase.Data.Provider;


namespace EPiServer.Reference.Commerce.Site
{
    [ServiceConfiguration(typeof(IMigrationStep))]
    public class AddContactBirthdayStep : IMigrationStep
    {
        private readonly IConnectionStringHandler _connectionStringHandler;

        public AddContactBirthdayStep(IConnectionStringHandler connectionStringHandler) => _connectionStringHandler = connectionStringHandler;

        public int Order => 2000;

        public string Name => "Add birthday to contact";

        public string Description => "Add birthday to contact";

        public bool Execute(IProgressMessenger progressMessenger)
        {
            var @lock = new SqlDistributedLock("MigrateContactBirthday", _connectionStringHandler.Commerce.ConnectionString);

            // 2. Use a standard 'using' block and the synchronous 'TryAcquire' method
            using (var handle = @lock.TryAcquire())
            {
                if (handle != null)
                {
                    // Current thread successfully acquired the lock across all instances.
                    // The lock is held until the 'handle' is disposed at the end of this block.
                    using (MetaClassManagerEditScope scope = DataContext.Current.MetaModel.BeginEdit(MetaClassManagerEditScope.SystemOwner, AccessLevel.System))
                    {
                        MetaClass mc = DataContext.Current.MetaModel.MetaClasses[ContactEntity.ClassName];
                        if (mc.Fields.Contains("Birthday"))
                        {
                            return true;
                        }
                        using (MetaFieldBuilder builder = new MetaFieldBuilder(mc))
                        {
                            builder.CreateDate("Birthday", "Birthday", true, true);
                            builder.SaveChanges();
                        }

                        scope.SaveChanges();
                    }
                    return true;
                }
                else
                {
                    // Could not get the lock (another instance is already running it)
                    // We return false to not override the other instance's result.
                    return false;
                }
            }

            return true;

        }
    }
}
```





This is of course an example but you can take the pattern and adapt to your needs.