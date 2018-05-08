---
layout: post
title: SQL Server Partition Swapping
date: '2018-05-11 07:56:51'
---
When working with large amounts of data in ETL jobs it can often can sink a lot of time inserting new data into a table and archiving old data out.

For example imagine we have a reporting table in our data warehouse that stores the last 12 completed months of sales, each month we remove or archive the oldest month from the table and add the latest complete month to it. The delete operation alone can take hours to run when data gets to a large enough size, it often also causes a large level of locks/blocking potentially making the whole table inaccessible to other queries. If we were to partition the data on month SQL Server will let us remove a whole partition from the table pretty much instantly and swap a new one in again pretty much instantly from a source table where we've pre-loaded the table, all this and no blocking!

Now Partitioning does have it's drawbacks and it needs to be a good fit for your scenario before you think about using it. Some potentially blockers are it's Enterprise only until SQL Server 2016 and if you're querying across partitions where SQL Server can't use partition elimination to limit the partitions it reads things can get a lot slower.

Let's take a look at getting a working example that uses partitioning to swap in and out data from our partitioned table. To hook this up we first need a new database that has 12 partitions one for each month...

{% highlight sql %}
CREATE DATABASE PartitionSwapTest
GO
USE PartitionSwapTest
GO

ALTER DATABASE PartitionSwapTest ADD FILEGROUP [FS_Months]

ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Jan',FILENAME = N'e:\PartitionTest\PartitionSwapJan.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Feb',FILENAME = N'e:\PartitionTest\PartitionSwapFeb.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Mar',FILENAME = N'e:\PartitionTest\PartitionSwapMar.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Apr',FILENAME = N'e:\PartitionTest\PartitionSwapApr.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_May',FILENAME = N'e:\PartitionTest\PartitionSwapMay.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Jun',FILENAME = N'e:\PartitionTest\PartitionSwapJun.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Jul',FILENAME = N'e:\PartitionTest\PartitionSwapJul.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Aug',FILENAME = N'e:\PartitionTest\PartitionSwapAug.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Sep',FILENAME = N'e:\PartitionTest\PartitionSwapSep.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Oct',FILENAME = N'e:\PartitionTest\PartitionSwapOct.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Nov',FILENAME = N'e:\PartitionTest\PartitionSwapNov.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
ALTER DATABASE PartitionSwapTest ADD FILE (NAME = N'PartitionSwap_Dec',FILENAME = N'e:\PartitionTest\PartitionSwapDec.ndf', SIZE = 30MB, MAXSIZE = 10000MB, FILEGROWTH = 30MB) TO FILEGROUP [FS_MONTHS]  
{% endhighlight %}

We then need a partition function that will put our data into the correct partition depending on it's month...

{% highlight sql %}
CREATE PARTITION FUNCTION pf_DaySalesMonth(INT)
AS RANGE LEFT FOR VALUES(1,2,3,4,5,6,7,8,9,10,11,12)
GO
{% endhighlight sql %}

We now need a partition scheme to map a partition function to one or more file groups, in our case we'll put all our partitions in the same filegroup to keep things simple...

{% highlight sql %}
CREATE PARTITION SCHEME ps_MonthRange
AS
PARTITION pf_DaySalesMonth
ALL TO ([FS_Months])
GO
{% endhighlight %}

I'll keep the sales table small by just putting a date and quantity on it, I'll also add a computed field for month as that's what we need to pass into the partition function...

{% highlight sql %}
CREATE TABLE DaySales
(
   [Date] DATETIME NOT NULL,
   Qty INT,
   [Month] AS MONTH([Date]) PERSISTED  CHECK([Month] >= 1 AND [Month] <= 12 AND [Month] IS NOT NULL)
) ON ps_MonthRange([Month])
{% endhighlight %}

For examples sake let's just insert a record per day in 2018 with a random quantity...

{% highlight sql %}
DECLARE @day INT = 1
WHILE @day <= 365
   BEGIN
   INSERT INTO DaySales([Date],qty) 
   SELECT DISTINCT DATEADD(DAY,@Day,'20171231'), RAND()*100
   SELECT @Day = @Day +1
   END
{% endhighlight %}

You can then view the partitions we've created and how many rows they have in them by querying sys.partitions...

{% highlight sql %}
SELECT * FROM sys.partitions
WHERE object_id = OBJECT_ID('dbo.DaySales');
{% endhighlight %}

Let's pretend we're loading in data for January 2019 and as part of this process want to first remove or archive the data from January 2018. As I mentioned before large deletes can take time and cause a lot of blocking but because were using partition tables we can either truncate a partition or swap one out almost instantly. On SQL Server 2016+ we have support for truncating a partition like this...

{% highlight sql %}
TRUNCATE TABLE DaySales WITH (PARTITIONS(1))
{% endhighlight %}

With 1 being the ID of the partition we want to truncate. If you're running an earlier version of SQL Server or want to archive rather than delete then instead you can move the partition out into a new table...

{% highlight sql %}
CREATE TABLE Archive
()
   [Date] DATETIME NOT NULL,
   [Qty] INT,
   [Month] AS MONTH([Date]) PERSISTED
) ON [FS_Months]

ALTER TABLE DaySales SWITCH PARTITION 1 TO Archive
{% endhighlight %}

At this point the data for January has been removed or archived so we're ready to now swap in January 2019. To do that we need our source table to be on the same filegroup and have constraints that prevent any data being in it that is not appropriate for the partition we're swapping in to. Without the constraint the swap will error as SQL Sever Will not allow the possibility of swapping in data that does not fit the partition based on the partition function.

{% highlight sql %}
CREATE TABLE NewData
(
   [Date] DATETIME NOT NULL,
   [Qty] INT,
   [Month] AS MONTH([Date]) PERSISTED CHECK([Month] =1  AND [Month] IS NOT NULL)
) ON [FS_Months]

/*Load In New Data For January*/
SELECT @Day =  1
WHILE @day <= 31
   BEGIN
   INSERT INTO NewData([Date],qty)
   SELECT DISTINCT DATEADD(DAY,@Day,'20181231'), RAND()*100
   SELECT @Day = @Day +1
   END

/*swap in the new data*/
ALTER TABLE NewData SWITCH TO DaySales PARTITION 1;
{% endhighlight %}

All the data from NewData is now in the first partition for our DaySales table. For large warehouse tables this process can MASSIVELY speed up load and archiving processes and prevent excessive blocking/locking.