---
layout: post
title: Stop Worrying About Index Fragmentation and Manage Your Statistics
date: '2018-02-25 19:05:38'
tags: performance-tuning maintenance
---
I see a lot of posts online about rebuilding indexes in a scheduled maintenance task, this is a pretty intensive task with some potentially large gotchas with things like online/offline, replication etc... I've never seen an index rebuild magically fix an issue that just rebuilding the stats wouldn't have sorted with way less risk and time. The example below demonstrates what can happen when the statistics don't represent the shape of the data that you are querying on.

Let's setup a table and populate it with about 2 million records...

{% highlight sql %}
CREATE TABLE DailyRates
(
   Id INT IDENTITY PRIMARY KEY,
   [Date] DATETIME,
   Rate DECIMAL(10,2)
)

--Insert about 2 million records
INSERT INTO DailyRates ([Date],Rate)
SELECT 
   DATEADD(day,- CAST(((ROW_NUMBER() OVER(ORDER BY s1.type))/100) AS INT),GETDATE()),
   s1.number % 10 + 25/3
FROM 
   master.dbo.spt_values s1
   CROSS JOIN master.dbo.spt_values s2
WHERE 
   s2.number < 650
{% endhighlight %}

If we then run a simple select to get all records for today and turn on statistics IO/Execution plan...

{% highlight sql %}
SET STATISTICS IO ON
GO
SELECT 
   [Date],[Rate] 
FROM 
   DailyRates 
WHERE 
   [Date] BETWEEN DATEADD(DAY,-1,GETDATE()) AND GETDATE()
{% endhighlight %}

We can see this query performs about 12,000 logical reads but only returns about 99 rows. This is because we don't have an index so it's having to perform a full clustered index scan. Let's create an index on Date to avoid the need for a full scan....

{% highlight sql %}
CREATE INDEX ndx_dailyrates_date ON DailyRates([Date])
{% endhighlight %}

At the point the index is created SQL Server will build the statistics for the table. If we now run our select query again we can see we're now doing an index seek on our new index with a key lookup to get the rate, the reads are now only about 300. We could obviously also add rate as an include on the index but for the purpose of this demo let's ignore that. Let's imagine this table gets new rates every day for the following day....

{% highlight sql %}
INSERT INTO DailyRates([Date],[Rate]) VALUES(DATEADD(DAY,1,GETDATE()), 1)
{% endhighlight %}

Keep in mind that by default statistics are only recalculated when about 20% of the records in a table change, so in this case the odds are no statistics will be updated and the current statistics SQL Server holds for our index will not contain a range that covers this date. What do you think happens if we now run our select again but filter it for our new date?

{% highlight sql %}
SELECT 
   [Date], [Rate] 
FROM 
   DailyRates 
WHERE 
   [Date] BETWEEN GETDATE() AND DATEADD(DAY,1,GETDATE())
{% endhighlight %}

We're back to 12,000 logical reads and a full clustered index scan rather than our new non clustered index. Why did SQL Server do this? Because the statistics have not updated there are no existing statistics containing this date range so SQL Server has no idea how many rows to expect, this means when it's building it's plan it decides our new index with a key lookup back to rate might be too costly if there is a large amount of rows so instead it uses a full scan plan with the clustered index to avoid the key lookup.

Let's update the statistics and run the query again to see what happens...

{% highlight sql %}
UPDATE STATISTICS DailyRates WITH FULLSCAN
GO
SELECT 
   [Date], [Rate] 
FROM 
   DailyRates 
WHERE 
   [Date] BETWEEN GETDATE() AND DATEADD(DAY,1,GETDATE())
{% endhighlight %}

We're now getting 6 logical reads and a plan that uses our non clustered index for seeks again.

The point to make here is that you know the shape of your data better than SQL Server does and if there are situations like this that cause the statistics for the majority of the data you query for in a given table to be inaccurate then you should take the time to update them manually. One of the main places this issue shows itself is on tables that have large amounts historic data (causing the statistic rebuild threshold to be quite long) and often query for the current date. In the case above if we know tomorrows rates get loaded at 2am each day then we could schedule a task to update the statistics for this table once this has completed. For more information on viewing your statistics see my [intro to statistics](https://gavindraper.com/2017/05/22/sql-server-into-to-statistics/) post.