---
layout: post
title: Turbo Charged Staging\Caching Tables With In Memory OLTP
date: '2018-12-03 01:23:01'
tags: in-memory-oltp performance-tuning etl
---
Since SQL 2012 some really awesome new technologies have been introduced into the engine that are massively underused. Everyone is familiar with the traditional row store tables that SQL Server uses and people either don't know about or are scared off by Column Store and In Memory OLTP. In this post I'm going to demo a particular use case suited to In Memory OLTP and show the benefits versus the traditional on disk Row Store.

The below demo is all done using SQL Server 2019 and the WideImporters sample database but you should be able follow along on older versions and different databases with minor adjustments.

Lets imagine we're loading a throwaway staging table as an intermediate step in part of our ETL warehousing process. In Memory OLTP tables allow us to set their durability, if we set this to SCHEMA_ONLY then no data is ever persisted to disk, this means whenever you restart your server all data in these tables will be lost. This may be fine for any staging tables that are part of your ETL process.

Typically your ETL process will be pulling data in from all sorts of places which is hard to demo in a short blog post so in this example I'm just going to take 3 tables in the WideWorldImporters table and separately insert/update them into a staging table. The reason I'm not just joining them upfront is because I'm trying to simulate that this data could be coming from different places that cannot be joined  e.g in an SSIS workflow from CSV, SQL Table and an Oracle linked server.

If your database isn't already setup for In Memory OLTP tables then you'll first need to create an in memory filegroup and add a file to it....

{% highlight sql %}
ALTER DATABASE MyDatabase 
ADD FILEGROUP FG_MyDatabaseMemoryOptimized CONTAINS MEMORY_OPTIMIZED_DATA

ALTER DATABASE MyDatabase
ADD FILE (name = MyDatabaseDemoMemoryOptimized, filename = 'c:\MyDbs\')
TO FILEGROUP FG_MyDatabaseMemoryOptimized
{% endhighlight %}

Once this is done you're free to create in memory tables. To set us up for this demo let's create 2 staging tables, One traditional and one In Memory SCHEMA_ONLY...

{% highlight sql %}
CREATE TABLE StagingTransactions
(	
   Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
   Date DATETIME,
   StockItemId INT,
   StockItem NVARCHAR(300),
   Quantity DECIMAL(18,3),
   CustomerId INT,
   Customer NVARCHAR(100),
   PhoneNumber NVARCHAR(100),
   Website NVARCHAR(300)
)

CREATE TABLE StagingTransactionsInMemory
(	
   Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
   Date DATETIME,
   StockItemId INT,
   StockItem NVARCHAR(300),
   Quantity DECIMAL(18,3),
   CustomerId INT,
   Customer NVARCHAR(100),
   PhoneNumber NVARCHAR(100),
   Website NVARCHAR(300)
)
WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY)

{% endhighlight %}

I've deliberately set both primary keys to non clustered because In Memory tables can't have a clustered index and I wanted to keep them both a close to the same as possible.

Lets now run the following batch of 3 queries to bring in Transactions then customers and lastly stock item names simulating them all coming from different sources...

{% highlight sql %}
INSERT INTO StagingTransactions(Date,Quantity,StockItemId,CustomerId)
SELECT 
   TransactionOccurredWhen,
   Quantity,
   StockITemId,
   CustomerId
FROM 
   Warehouse.StockItemTransactions

UPDATE t  
SET 
   t.StockItem = s.StockItemName
FROM
   StagingTransactions t
   INNER JOIN Warehouse.StockItems s ON s.StockItemId = t.StockItemId

UPDATE t  
SET 
   t.Customer = c.CustomerName,
   t.PhoneNumber = c.PhoneNumber,
   t.Website = c.WebsiteURL
FROM
   StagingTransactions t
   INNER JOIN Sales.Customers c ON c.CustomerId = t.CustomerId
{% endhighlight %}

Then run it again but swap the table names for your in memory table.

On my machine this runs in about 20 seconds for the disk table and 3 seconds for the in memory table (This can be massively optimized with indexes but that's not what I'm trying to show here). Lets compare the client statistics for the disk table vs in memory...

> Table 'StagingTransactions'. Scan count 0, logical reads 949566, physical reads 0, read-ahead reads 468, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> Table 'StagingTransactions'. Scan count 1, logical reads 1250360, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> Table 'StagingTransactions'. Scan count 1, logical reads 1472818, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

You can see we're doing quite a large amount of IO with all those reads, now lets look at the in memory version...

> Table 'StockItemTransactions'. Scan count 1, logical reads 0, physical reads 0, read-ahead reads 0, lob logical reads 127, lob physical reads 0, lob read-ahead reads 0.

> Table 'StockItemTransactions'. Segment reads 1, segment skipped 0.

That's basically removed all of our IO! Magic right?

This speed difference comes from a few places...

- Memory is WAY faster than disk
- We've sacrificed persistence
- In Memory OLTP uses a completely new storage structure (There be no pages here)
- It's completely lockless (Optimistic Concurrency, even for Write, Write scenarios)

It's not a tool for every job as it comes with a number of limitations but staging tables are a place where it can really shine (providing you have enough memory).

Because we've used SCHEMA_ONLY another benefit is nothing that happens in this table touches the transaction log and as a result of that it wont hold up your HA/DR solution as nothing in this table will be shipped to the DR site. For example when inserting into our on disk staging table lets say we insert 2 million records, transform them then move them to our warehouse and then truncate the staging table the insertion and deletion of these records will be replicated on the DR site and could massively increase your RTO for bringing a DR site online.

I'll try to cover some of the other places where in memory tables can be a good fit in future posts.