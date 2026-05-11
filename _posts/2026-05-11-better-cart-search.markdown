---
layout: post
title: "Better serializable cart search"
date: 2026-05-11 11:09:50 +0200
categories: [performance, sql-server, optimization]
tags: [performance, optimization, database, index]
---

If you are using Optimizely Commerce (Connect, B2C, or whatever you call it these days), you are likely using its serializable cart mode. That cart mode has been introduced 10 years ago (I still remember a cold winter day when I first introduced it to the OMVPs (then EMVPs) at a summit). It was introduced to help reducing the performance issues - most notably deadlocks with order system. Previously, the carts use the same database schema as with the orders, you have base information stored in `OrderGroup` table and additional, customizable data stored in `OrderGroup_ShoppingCart`, with a foreign key reference. It works great when it comes to functionality, but when your customer places an order, you need to do two things - saving their order to `OrderGroup_PurchaseOrder` and delete from `OrderGroup_ShoppingCart` - as the reference keys are updated in `OrderGroup`, you risk the chance of deadlock which increasing with your number of carts/orders, and your traffic. Serializable cart mode solved that by moving the cart data to a separate table - and you can probably guess it - `SerializableCart`. The static fields have their own columns, but the data - which is extendable & customizable - is serialized and stored in Data column.

At that time it seems like a smart choice because we didn't want to mess with dynamically changing the database schema to accomondate changes in the metaclass. However the problem arises with searching for cart. Searching for basic things, for example, carts that were not updated in the last 30 days, is easy, because `Modified` has its own column and an index will be very efficient for that. Searching for things such as, the data inside the cart, becomes a bottleneck. I've seen customers that run query on finding specific carts which turned the a huge resource consumer

```
SELECT CartId FROM [SerializableCart] WHERE Data Like '%"MerchantRef":{"$type":"System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089","$value":"2937B064-2380-4DF8-AB1C-CC1AB19FC645"%'
```
For each query of this you can be sure a full clustered index scan will be performed, and it can turn ugly real fast if you have a substainable amount of rows in SerializableCart table. This will certainly hurt performance and could cause further issue. We can't do anything here with the index - because Data is a `NVARCHAR(MAX)` column so index is not allowed. And even if we could, the wildcard before and after the search value will simply invalidate the index usage anyway.

Is there a better way to do it? Yes, it turns out you can have computed columns based on value of a JSON column, and then adding an index for that:

```
ALTER TABLE dbo.SerializableCart ADD MerchantRef AS JSON_VALUE(Data, '$.MerchantRef');  
CREATE INDEX idx_soh_json_MerchantReference     ON dbo.SerializableCart(MerchantRef);
```

And the query can be rewritten to a simple

```
SELECT CartId FROM [SerializableCart] WHERE MerchantRef = `abc`
```

And instead of a clustered index scan, you get a nice Index seek (and even without a key lookup)

Note that we still have some major limitations. For example, searching for carts containing specific SKU is still requiring a full row scan. There is a trick for that, but let's save that for another blog post!