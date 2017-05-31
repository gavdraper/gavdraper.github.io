---
layout: post
title: SQL Server Parameter Sniffing In Depth
date: '2017-05-31 08:38'
---
If you've ever searched for SQL Server parameter sniffing you've probably read all sorts of bad things about it. In truth parameter sniffing is an optimization technique SQL Server uses to allow plans to be cache and reused with different parameter values  and for the most part it works well and significantly improves overall performance vs not using parameter sniffing. We are going to look at some examples here where parameter sniffing does cause a problem and how to solve it.

Before we get too in depth on how parameter sniffing works let's setup a database to run the examples on.

## Setup ##
Create a new database and execute the following script to create a table  and fill it with seed data.

{% highlight sql %}
DROP TABLE [User]
CREATE TABLE [User]
(
    Id INT IDENTITY PRIMARY KEY,
    Username NVARCHAR(100),
    Firstname NVARCHAR(100),
    LastName NVARCHAR(100),
    PlaceOfBirth NVARCHAR(100)
)

INSERT INTO [dbo].[User]
    ( 	
    Username,
    FirstName,
    LastName, 
    PlaceOfBirth
    )
VALUES
    ('clairetemple','Claire','Temple','UK'),
    ('lukecage','Luke','Cage','UK'),
    ('jessiejones','Jessie','Jones','UK'),
    ('tonystark','Tony','Stark','UK'),
    ('mattmurdock','Matt','Murdock','UK')

DECLARE @LoopCount INT = 1
WHILE @LoopCount < 18
    BEGIN
    INSERT INTO [dbo].[User](Username,FirstName,LastName, PlaceOfBirth)
    SELECT 
        CAST(@LoopCount AS NVARCHAR(4)) + Username,
        CAST(@LoopCount AS NVARCHAR(4)) + FirstName,
        CAST(@LoopCount AS NVARCHAR(4)) + LastName,
		PlaceOfBirth
    FROM [dbo].[User]
    SET @LoopCount = @LoopCount +1
    END

INSERT INTO [dbo].[User](Username,FirstName,LastName,PlaceOfBirth)
VALUES('Gavin','Gavin','Draper','Mars')   

CREATE NONCLUSTERED INDEX ndx_User_PlaceOfBirth ON [dbo].[User](PlaceOfBirth)
{% endhighlight %}

At this point with have 655,000 Users in our table with a PlaceOfBirth = UK and 1 User with a PlaceOfBirth = Mars.

## Scenario ##
Let's imagine we have a stored procedure that takes in a PlaceOfBirth as a parameter and returns all users matching that place of birth...

{% highlight sql %}
CREATE PROCEDURE GetUsersByPlaceOfBirth
(
    @PlaceOfBirth NVARCHAR(100)
)
AS
SELECT 
    Username,
    FirstName,
    LastName,
    PlaceOfBirth
FROM
    [dbo].[User]
WHERE
    PlaceOfBirth = @PlaceOfBirth
{% endhighlight %}

Let's then execute that procedure with include actual query plan enabled and see what it does...

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'UK'
{% endhighlight %}

![Tablescan Query Plan]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-tablescan.JPG)

We can see here that even though we have a NonClustered index on PlaceOfBirth the query optimizer has chosen to do a full clustered index scan. This is because it will have looked at the statistics for the PlaceOfBirth column and seen that pretty much every row matches our parameter and decided a table scan is quicker than an index seek with 655,000 key lookups. For more information on SQL Server Statistics see [An Introduction To SQL Server Statistics](https://gavindraper.com/2017/05/22/sql-server-into-to-statistics/)

What do you think will happen if we run the same procedure with a parameter of Mars? 

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'Mars'
{% endhighlight %}

![Tablescan Query Plan]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-tablescan.JPG)

Not what you expected? Why would the optimizer choose to do a full clustered index scan of 655,000 records rather than seeking straight to the one matching row? If you haven't guessed this is down to parameter sniffing.

## What Is Parameter Sniffing ##
The first time a parameterized query is run (Like our stored procedure above) SQL Server will optimize the query plan based on the parameters supplied and cache the plan. The next time you run that same query with different parameters the same plan will be used.

The reason for this is creating optimized query plans is an expensive operation so SQL Server caches plans for reuse. In most situations this works great, the reason the example above has an issue is because the parameters passed in generate a massive difference in the amount of data returned. If we had a more even split of PlaceOfBirth in our database there would be no issue.

To prove this is a caching issue lets clear the plan cache by running...

{% highlight sql %}
DBCC FREEPROCCACHE
{% endhighlight %}

Now lets run our stored procedure again for Mars...

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'Mars'
{% endhighlight %}

![Index Seek Query Plan]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-index-seek.JPG)

Yay! We're now using an index seek. If we look at the estimated query cost for Mars before we cleared the cache vs after we can see we've gone from 9.35973 to 0.0065704.

## Preventing Problems Caused By Parameter Sniffing ##
There are several different things we can do when we find we have a parameter sniffing issue...

1. Use the WITH RECOMPILE hint to force the procedure to always compile on run, generating a new plan each time

{% highlight sql %}
ALTER PROCEDURE GetUsersByPlaceOfBirth
(
    @PlaceOfBirth NVARCHAR(100)
)
WITH RECOMPILE
AS
SELECT 
    Username,
    FirstName,
    LastName,
    PlaceOfBirth
FROM
    [dbo].[User]
WHERE
    PlaceOfBirth = @PlaceOfBirth
{% endhighlight %}

This obviously has an overhead of constant recompilations which depending on the complexity of the query and the frequency it's run could cause other issues.

2.  Use the OPTIMIZE FOR UNKNOWN query hint

{% highlight sql %}
ALTER PROCEDURE GetUsersByPlaceOfBirth
(
    @PlaceOfBirth NVARCHAR(100)
)
AS
SELECT 
    Username,
    FirstName,
    LastName,
    PlaceOfBirth
FROM
    [dbo].[User]
WHERE
    PlaceOfBirth = @PlaceOfBirth
OPTION (OPTIMIZE FOR (@PlaceOfBirth UNKNOWN))
{% endhighlight %}

This tells SQL to generate a plan without looking at the parameter. This may or may not create a better generic plan, for the most part I find this generates non optimal plans. In the case of our example above it actually ends up generating the clustered index scan plan so doesn't change anything.

3. Use the OPTIMIZE FOR x query hint

{% highlight sql %}
ALTER PROCEDURE GetUsersByPlaceOfBirth
(
    @PlaceOfBirth NVARCHAR(100)
)
AS
SELECT 
    Username,
    FirstName,
    LastName,
    PlaceOfBirth
FROM
    [dbo].[User]
WHERE
    PlaceOfBirth = @PlaceOfBirth
OPTION (OPTIMIZE FOR (@PlaceOfBirth = 'Mars'))
{% endhighlight %}

This will force SQL Server  to use the plan that works best for the value Mars, in this case it will make our UK query very inefficient, which may be ok if that's a query that is very rarely used and we want to optimize for the value that gets executed the most.

4.  Use separate queries for the data that falls outside the normal range. For example PlaceOfBirth Mars clearly differs in cardinality from most of the other data. In this case we could have 2 procedures GetUsersByPlaceOfBirthMars and keep our existing GetUsersByPlaceOfBirth. Any queries for Mars go through the new procedure. This way we get all the benefits of a cached optimized plan but take the hit by having to maintain multiple procedures.

## Fixing A Parameter Sniffing Issue In Production ##
Often these issues get reported by users saying queries are timing out, in this scenario we often need to just get the query running again before we start looking at the best plan of attack for preventing the issue. 

In the demos above I used DBCC FREEPROCCACHE to clear the cache, in production this is almost never a good idea as it will clear all plans from the cache causing the server to take a sudden hit recompiling everything. It is however an option that will most likely fix the issue depending on the values are first called with that query once the cache is clear.

You can also pass in a plan handle to FREEPROCCACHE to clear a single plan from the cache, this is a much more lightweight option and if you have the plan handle this is probably the best option. You can find the plan handle by querying the sys.dm_exec_query_stats view...

{% highlight sql %}
SELECT 
    query_stats.plan_handle, 
    query_stats.last_execution_time, 
    query_text.text
FROM 
    sys.dm_exec_query_stats query_stats
    CROSS APPLY sys.dm_exec_sql_text (query_stats.[sql_handle]) AS query_text
ORDER BY 
    query_stats.last_execution_time DESC
{% endhighlight %}

![Plan Handle Query]({{site.url}}/content/images/2017-parameter-sniffing/plan_handle.JPG)

From that we can run FREEPROCCACHE with the handle of the plan  we want to remove...

{% highlight sql %}
DBCC FREEPROCCACHE(0x05000B00CDC1731290A264790400000001000000000000000000000000000000000000000000000000000000)
{% endhighlight %}

## Using The Query Store To Spot Bad Parameter Sniffing ##
Normally these issues are spotted by a user reporting that a query has suddenly started going very slow, this is typically triggered by the first run of a given query on a clean cache using parameters that are outside the normal range for the shape of the data. 

SQL Server 2016 Introduced the Query Store which gives us much more information into query plan usage. You need to enable the Query Store before you can use it which can be done from Management Studio by going to the database properties then on the Query Store tab set the operation mode to Read/Write.

If we then expand the Query Store node inside our Database in Management Studio we can look at the Regressed Queries report...

![Regressed Query Report]({{site.url}}/content/images/2017-parameter-sniffing/regressed-queries.JPG)

Just at a glance we can see this query has used 2 different plans, one of which seems to perform better. There is a chance this is caused by parameter sniffing. From here we can Shift and click each plan then click compare to see what the differences are and what parameter values they were compiled for. We can also pick a plan and click the Force button to force that plan to be used, this is a good quick fix to get a query running again that is currently suffering from parameter sniffing. 

For versions before 2016 we can use views like sys.dm_exec_query_stats query_stats mentioned above and run them on an interval logging things like elapsed time and look for regressions, the query store just makes this process a lot easier.

## Summary ##
To summarize for the most part parameter sniffing is an optimization and allows plans to be cached. When we do have issues due to parameter sniffing there are several options described above that can help, all of which have various trade offs. It's all about choosing the best option for your use case. 

Good Luck! 