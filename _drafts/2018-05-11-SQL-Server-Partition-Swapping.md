---
layout: post
title: SQL Server Partition Swapping
date: '2018-05-08 07:56:51'
---
When working with large amounts of data in ETL jobs it can often take a long time and become unmanageable to remove old data and add new data. 

For example imagine we have a reporting table in our data warehouse that stores the last 12 completed months of sales and each month we remove the oldest month from the table and add the latest complete month to it. the delete operation alone can take hours to run when data gets too large for the engine to handle it also causes a lot of blocking during this process. If we were to partition the data on Month SQL Server will let us remove a whole partition from the table pretty much instantly and swap a new one in again pretty much instantly. Now this does have it's drawbacks and is not appropriate if you're querying across partitions and needing indexes that span partitions but for our example let's pretend we only report on a chosen month and don't need any indexing to cross partition boundaries. 

To hook this up we first need a new database that has 12 partitions one for each month...

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

We now need a partition scheme this is what maps a partition function to one or more file groups, in our case we'll put all our partitions in the same filegroup to keep things simple...

{% highlight sql %}
CREATE PARTITION SCHEME ps_MonthRange
AS
PARTITION pf_DaySalesMonth
ALL TO ([FS_Months])
GO
{% endhighlight %}

I'll keep the sales table simple by just putting a date and quantity on it, I'll also add a computed field for month as that's what we need to pass into the partition function...

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


{% highlight sql %}
--Must be on same filegroup
CREATE TABLE NewData
(	
	[Date] DATETIME NOT NULL,
	Qty INT,
	[Month] AS MONTH([Date]) PERSISTED CHECK([Month] =1  AND [Month] IS NOT NULL) 	
) ON [FS_Months]


--Load In New Data For January
SELECT @Day =  1
WHILE @day <= 31
	BEGIN
	INSERT INTO NewData([Date],qty) 
	SELECT DISTINCT DATEADD(DAY,@Day,'20171231'), RAND()*100
	SELECT @Day = @Day +1
	END

--Remove old partition
--And Swap new one on
SELECT * FROM dbo.DaySales WHERE MONTH([Date]) = 1
--2016+ allows truncation of a partition
--TRUNCATE TABLE DaySales WITH (PARTITIONS(1))
--Previous versions you can swap them out
CREATE TABLE Archive
(	
	[Date] DATETIME NOT NULL,
	Qty INT,
	[Month] AS MONTH([Date]) PERSISTED
) ON [FS_Months]
ALTER TABLE DaySales SWITCH PARTITION 1 TO Archive ; 

SELECT * FROM dbo.DaySales WHERE MONTH([Date]) = 1
ALTER TABLE NewData SWITCH TO DaySales PARTITION 1; 
SELECT * FROM dbo.DaySales WHERE MONTH([Date]) = 1
{% endhighlight %}