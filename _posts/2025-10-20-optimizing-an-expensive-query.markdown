---
layout: post
title: "Optimize an expensive query, a case study"
date: 2025-10-20 11:09:50 +0200
categories: [performance, database, optimization]
tags: [performance, optimization, sql-server, index]
---

Black Friday is coming, and we are working hard to make sure customers running on Optimizely DXP that anticipating a Black Friday campaign, will be in good shape when the traffic hits. As they say - the optimization season is in full swing. This time, let's take a case study of how we can identify and optimize a (very) expensive query.

A customer has this very heavy custom query (for obvious reasons I renamed the column names )

```lang=SQL
SELECT ISNULL(SUM(t.Units), 0)
FROM tblAllocMapX t
WHERE t.ItemCode = @itemCode
  AND t.FlowState < 3
  AND DATEADD(HOUR, 1, t.LoggedAt) > GETUTCDATE()
```

and it's the most expensive query by a large margin

![](/assets/img/2025-10-20-optimizing-an-expensive-query/2025-10-20-12-00-11.png)

It is so expensive, all other queries are dwarfed completely. For reference, this is something you would normally see

![](2025-10-20-optimizing-an-expensive-query/2025-10-20-12-02-32.png)

Let's take a dive on how can we optimize it.

As this is a query on a production database, I can't make change to it. So what we're going to do is to replicate the table on our local database

```lang=SQL
CREATE TABLE [dbo].[tblAllocMapX](
	[Id] [int] IDENTITY(1,1) NOT NULL,              
	[CustRef]  [varchar](10) NULL,                       
	[CartToken]  [varchar](32) NOT NULL,                 
	[CartRef]  [varchar](32) NOT NULL,                   
	[ItemCode] [varchar](12)  NOT NULL,                  
	[ProdRef] [varchar](21) NOT NULL,                   
	[Units] [smallint] NOT NULL,                        
	[UnitCost] [money] NOT NULL,                        
	[FlowState] [smallint] NOT NULL,                    
	[FinalizedAt] [datetime] NOT NULL,                  
	[LoggedAt] [datetime] NOT NULL                      
)
```

and then insert a random 100k rows into it. The table in reality has around 51k rows, but while we're at it, why not level up our game

```lang=SQL
SET NOCOUNT ON;

DECLARE @i INT = 0;

WHILE @i < 100
BEGIN
    INSERT INTO [dbo].[tblAllocMapX] (
        [CustRef], [CartToken], [CartRef], [ItemCode], [ProdRef],
        [Units], [UnitCost], [FlowState], [FinalizedAt], [LoggedAt]
    )
    SELECT
        RIGHT(NEWID(), 10),                          -- CustRef
        LEFT(NEWID(), 32),                                      -- CartToken
        LEFT(NEWID(), 32),                                     -- CartRef
        LEFT(NEWID(), 12),                           -- ItemCode
        LEFT(NEWID(), 21),                           -- ProdRef
        CAST(RAND(CHECKSUM(NEWID())) * 100 AS SMALLINT), -- Units
        CAST(RAND(CHECKSUM(NEWID())) * 100 AS MONEY),    -- UnitCost
        CAST(RAND(CHECKSUM(NEWID())) * 5 AS SMALLINT),   -- FlowState (0–4)
        DATEADD(DAY, -ABS(CHECKSUM(NEWID()) % 365), GETDATE()), -- FinalizedAt
        GETDATE()                                    -- LoggedAt
    FROM sys.all_objects
    WHERE [object_id] % 2 = 0
    AND [object_id] < 2000; -- ~1000 rows per batch

    SET @i = @i + 1;
END;
```

Now let's run the query with a random value taken from the table. And it does not look good

![](2025-10-20-optimizing-an-expensive-query/2025-10-20-12-36-08.png)

Statistics is very expensive either for such a small query

```
(1 row affected)
Table 'tblAllocMapX'. Scan count 1, logical reads 2329, physical reads 0, page server reads 0, read-ahead reads 0, page server read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob page server reads 0, lob read-ahead reads 0, lob page server read-ahead reads 0.
```

From the execution plan, the obvious next step is to add an index to the `ItemCode` column, but does it help? Nope


