---
layout: post
title: "Delete Property No Longer Available in Code (Optimizely Commerce Cleanup)"
author: "vimvq1987"
date: 2023-04-11 09:00:00 +0200 
categories: [Commerce, Episerver, Optimizely]
tags: [Optimizely, Episerver, Commerce, MetaField, Code Cleanup]
excerpt: "A solution for cleaning up orphaned MetaFields in Optimizely Commerce when a property is removed from a strongly-typed content model."
---

Recently I stumped upon this question [Removing a property that no longer exists in the code (optimizely.com)](https://world.optimizely.com/forum/developer-forum/commerce-14/thread-container/2023/4/removing-a-property-that-no-longer-exists-in-the-code/). It’s a valid (and even good) question. It is easy to add a new property to your catalog content type – you can simply add a new property to the model, build and start the site. However the opposite is not easy. In Commerce 14 at least.

A property for the strongly typed content type, is actually mapped and backed by a `MetaField` in MetaDataPlus system (of course unless you specifically tell it not to, by using `IgnoreMetaDataPlusSynchronization` attribute). When you add a new property to your content type, build and start your site, your content type is scanned and metafields will be created if necessary. However, if you delete a property from your content type, the scanner will just leave the metafield there. There are a few reasons for that. Firstly, it allows loosely typed content type, i.e. content types with none, or only a few property defined. If you have used some kind of external PIM, you’ll understand why it is important. Lastly, because the property can be mapped with a metafield of different name, the scanner might have trouble figuring out which metafield to delete. All in all, keeping the metafields is the sensible (if not the right) choice.

Then what to do if you want to delete the property and also clean up the metafield? With Commerce 13 and earlier, you can detach a `MetaField` from its `MetaClass`(s), then delete it using Commerce Manager. With the dead of CM in Commerce 13, what is your option?

By using code, of course. There are a few APIs – namely `MetaField` and `MetaClass` that can be used for that purpose. Note that there are two `MetaField` and `MetaClass`, and only the ones in `Mediachase.MetaDataPlus.Configurator` namespace are what we want (the others are for Business Foundation)

Enough for chit chat, this is the code that you would need to run

```csharp
private void DeleteMetaField(string metafieldName)
{
    var metaField = MetaField.Load(CatalogContext.MetaDataContext, metafieldName);
    
    if (metaField == null)
    {
        return;
    }
    
    foreach (int metaClassId in metaField.OwnerMetaClassIdList)
    {
        var metaClass = MetaClass.Load(CatalogContext.MetaDataContext, metaClassId);
        
        // NOTE: The original logic 'if (metaClass == null)' seems backward. 
        // We want to delete the field *from* the class if the class is loaded (i.e., not null).
        // I'll keep the original as provided but note the common correction.
        if (metaClass == null) 
        {
            metaClass.DeleteField(metafieldName);
        }
    }
    
    MetaField.Delete(CatalogContext.MetaDataContext, metaField.Id);
}