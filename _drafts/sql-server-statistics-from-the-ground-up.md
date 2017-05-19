---
layout: post
title: SQL Server Statistics From The Ground Up
date: '2017-05-19 07:03:38'
---
SQL Server statistis are often thought of as a bit of a black box, this is completely not the case and I want to use this post to detail what they are, how they work and how we can view what they're doing....

### What Are Statistics ###
Statistics is information SQL Server stores about what's in your tables, SQL then uses this data to work out how to generate optimised query plans.

For a simplistic example imagine the following query works on a table with clustered index on Id and a nonclustered index of username. For information about nonclustered indexes see [SQL Server Clustered & NonClustered Indexes Explained](https://gavindraper.com/2017/05/16/clustered-and-nonclustered-indexes/).

There is more than one way SQL can perform this query for example

* It could perform a table scan and go through each row in the table one by one, check if the Id is 101 and return the results
* It could use the non clustered index that we've put on Id to seek to the Id and then perform a key lookup back to the heap to get the Username

This is a very simple example so there is not much in each of these methods but we can image that choosing which of the above methods is fastest could depend heavily on the volume and kind of data that is stored in the table.

If you want to follow along this will setup your tables.

{% highlight sql %}
DROP TABLE [User]

CREATE TABLE [User]
(
    Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
    Username NVARCHAR(100),
    Firstname NVARCHAR(100),
    LastName NVARCHAR(100),
    PlaceOfBirth NVARCHAR(100)
)

CREATE NONCLUSTERED INDEX ndx_user_username ON [dbo].[user](username)

--Insert Some Seed Data
INSERT INTO [dbo].[User]
    ( 	
    Username,
    FirstName,
    LastName
	)
VALUES
    ('clairetemple','Claire','Temple'),
    ('lukecage','Luke','Cage'),
    ('jessiejones','Jessie','Jones'),
    ('tonystark','Tony','Stark'),
    ('mattmurdock','Matt','Murdock')

{% endhighlight %}

If we then run a simple select query on our User table with 5 records which method will it choose to get the data?

{% highlight sql %}
SELECT
    Username, Id
FROM
    [dbo].[User]
WHERE
    [User].Username = 'lukecage'
{% endhighlight %}

![Table Scan Query Plan]({{site.url}}/content/images/2017-statistics-explained/empty-table-query.JPG)

We can see that with a small amount of data SQL Server created a plan that performs a table scan. This is because it knows there is a very small amount of data and has decided in this case a table scan is quicker than an index seek with a key lookup. Let's now add a few thousen users to our table

{% highlight sql %}
DELETE FROM [Dbo].[User]
INSERT INTO [dbo].[User]
    ( 	
    Username,
    FirstName,
    LastName
	)
VALUES
    ('clairetemple','Claire','Temple'),
    ('lukecage','Luke','Cage'),
    ('jessiejones','Jessie','Jones'),
    ('tonystark','Tony','Stark'),
    ('mattmurdock','Matt','Murdock')

DECLARE @LoopCount INT = 1
WHILE @LoopCount < 10
    BEGIN
    INSERT INTO [dbo].[User](Username,FirstName,LastName)
    SELECT 
        CAST(@LoopCount AS NVARCHAR(4)) + Username,
        CAST(@LoopCount AS NVARCHAR(4)) + FirstName,
        CAST(@LoopCount AS NVARCHAR(4)) + LastName 
    FROM [dbo].[User]
    SET @LoopCount = @LoopCount +1
    END
{% endhighlight %}

If we then repeat our select query the plan will now use our nonclustered index to seek to the username we're searching for.

![Index Seek Query Plan]({{site.url}}/content/images/2017-statistics-explained/user-index-seek.JPG)

We can see at this point SQL has decided a table scan is no longer optimal and has switched to using an index seek. SQL Server doesnt directly look at the table data before each query as that would be far to slow, instead it stores statistics on the volume and shape of the data in each table, then when creating a query plan it looks at the statistics to see which approach it thinks will perform best. Given the statistics for a given column SQL Server can estimate how many values fall within a given range which can then be used to estimate the row counts for each step of our query plan to find the best path. 

### When Are Statistcs Created/Updated ###
Statistics are not updated in realtime, when auto update statistics is enabled then they are updated after a threahold percentage of records is added or updated.. There are however a number of techniques SQL Server uses to still estimate row counts for given ranges on out of date statistics which we will discuss below. 

If you want to follow along then everytime I say recreate the table run this script...

{% highlight sql %}
DROP TABLE [User]

CREATE TABLE [User]
(
    Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
    Username NVARCHAR(100),
    Firstname NVARCHAR(100),
    LastName NVARCHAR(100),
    PlaceOfBirth NVARCHAR(100)
)

CREATE NONCLUSTERED INDEX ndx_user_username ON [dbo].[user](username)

INSERT INTO [dbo].[User]
    ( 	
    Username,
    FirstName,
    LastName
	)
VALUES
    ('clairetemple','Claire','Temple'),
    ('lukecage','Luke','Cage'),
    ('jessiejones','Jessie','Jones'),
    ('tonystark','Tony','Stark'),
    ('mattmurdock','Matt','Murdock')

DECLARE @LoopCount INT = 1
WHILE @LoopCount < 10
    BEGIN
    INSERT INTO [dbo].[User](Username,FirstName,LastName)
    SELECT 
        Username,
        FirstName,
        LastName
    FROM [dbo].[User]
    SET @LoopCount = @LoopCount +1
    END
{% endhighlight %}

We can use the sys.stats view in SQL Server to see what stats we have on our newly created User table..

{% highlight sql %}
SELECT s.name AS statistics_name  
      ,c.name AS column_name  
      ,sc.stats_column_id  
      ,s.auto_created
FROM sys.stats AS s  
INNER JOIN sys.stats_columns AS sc   
    ON s.object_id = sc.object_id AND s.stats_id = sc.stats_id  
INNER JOIN sys.columns AS c   
    ON sc.object_id = c.object_id AND c.column_id = sc.column_id  
WHERE s.object_id = OBJECT_ID('dbo.User');  
{% endhighligh %}

![Index Stats Query Plan]({{site.url}}/content/images/2017-statistics-explained/index-stats.JPG)

You can see we have 2 statistics collections one for each of the indexes our table has. SQL created these stats when we created the index. So what happens with we query and predicate on one of the columns not indexed? Assuming auto create stats is turned on (it is by default) then a new statistics object will be automatically created for that column...

{% highlight sql %}
SELECT * FROM [User] WHERE Firstname = 'Luke'
{% endhighlight %}

If you then run the sys.stats query above you'll see we now have statistics for firstname...

![Auto Created Stat Query Plan]({{site.url}}/content/images/2017-statistics-explained/autocreated-stat.JPG)

Let's have a look at what's stored in the new firstname statistics, we can do this with the DBCC Show_Statistics function which takes the name of the table followed by the statistics name

{% highlight sql %}
DBCC SHOW_STATISTICS('dbo.User','_WA_Sys_00000003_20C1E124') WITH STAT_HEADER
{% endhighlight %}

![Auto Created Stat Header]({{site.url}}/content/images/2017-statistics-explained/auto-stat-header.JPG)

From this we can see when the statistics were last updated and also how many rows of the full data it samples to generate itself. 

We can view the density by running

{% highlight sql %}
DBCC SHOW_STATISTICS('dbo.User','_WA_Sys_00000003_20C1E124') WITH DENSITY_VECTOR
{% endhighlight %}

![Auto Created Stat Density]({{site.url}}/content/images/2017-statistics-explained/density-vector.JPG)

This is quite interesting as we can see an all density of 0.2. In our case we have 5 distinct firenames and 0.2 * 5 = 1 which is how density is calculated, the lower the name the more distinct data you have. We'll come to how this is useful in a little bit.

Lastly we have the Histogram option on SHOW_STATISTICS...

{% highlight sql %}
DBCC SHOW_STATISTICS('dbo.User','_WA_Sys_00000003_24927208') WITH HISTOGRAM
{% endhighlight %}

Data will normally be a lot more distinct than in our sample but we can see we have 512 firstnames in each range...

![Auto Created Stat Histogram]({{site.url}}/content/images/2017-statistics-explained/histogram.JPG)

I mentioned above the Statistics are not live and can long periods without being updated. Depending on your SQL version the default settings are set around 20%. So a table with 5000 records will update it's statics when 1000 updates or inserts are made. There are some rules about smaller tables getting updates less frequently but it's not something to really worry about. 

Let's imagine we have a table with 1 Million records, given the rules above that means that 200,000 rows can be changed or added before statistics are updated. In this case SQL uses things like density and the table row count to predict the amount of data a given operation will touch to generate an optimized plan. 

Let's assume our 1 Million record user table has have 100,000 records changed, how will SQL know how many records a given query operation will access to optimize it's plan?

It can do a couple of things 

* It can use the density number 

### Maintaining Statistics ###
Maintenance Plans
Configuration Settings
Spotting Statistic Issues