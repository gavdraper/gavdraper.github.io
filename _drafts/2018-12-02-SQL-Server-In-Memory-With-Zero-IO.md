---
layout: post
title: SQL Server Magic Tables With Zero Disk IO
date: '2018-12-02 09:34:01'
---
Since SQL 2012 some really crucial new technologies have been introduced into the engine that are massively underused. Everyone is familiar with the traditional row store tables that SQL Server uses and people either don't know about or are scared off by ColumnStore and In Memory OLTP. In this post I'm going to demo a particular use case suited to In Memory OLTP and show the benefits versus the traditional RowStore.

The below demo is all done using SQL Server 2019 and the WideImporters sample database but you should be able follow along on older versions and different databases with minor adjustments.

Lets imagine we're loading a throwaway staging tables as an intermediate step in part of our ETL warehousing process. In Memory OLTP tables allow us to set their durability, if we set this to SCHEMA_ONLY then no data is ever persisted to disk, this means whenever you restart your server all data in these tables will be lost. Typically this will be fine for any staging tables that are psart of your ETL process.

Typically your ETL process will be pulling data in from all sorts of places which is hard to demo in a short blog post so in this example I'm just going to take 3 tables in the WideWorldImporters table and seperately insert/merge them into a staging table. The reason I'm not just joining them upfront is because I'm trying to simulate that this data could be coming from different places that cannot be joined  e.g in an SSIS workflo from CSV, SQL Table and an Oracle linked server.

If your database isnt already setup for In Memory OLTP tables then you'll first need to create an in memory filegroup and add a file to it....

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
   StockItem NVARCHAR(300),
   Quantity DECIMAL(18,3),
   Customer NVARCHAR(100),
   PhoneNumber NVARCHAR(100),
   Website NVARCHAR(300)
)

CREATE TABLE StagingTransactionsInMemory
(	
   Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
   Date DATETIME,
   StockItem NVARCHAR(300),
   Quantity DECIMAL(18,3),
   Customer NVARCHAR(100),
   PhoneNumber NVARCHAR(100),
   Website NVARCHAR(300)
)
WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY)
{% endhighlight %}

I've deliberately set both primary keys to non clustered because In Memory tables can't have a clustered index and I wanted to keep them both a close to the same as possible.