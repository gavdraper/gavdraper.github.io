---
layout: post
title: SQL Server Parameter Sniffing
date: '2017-05-22 07:20:38'
---
If you've ever search for SQL Server parameter sniffing you've probably read all sorts of bad things about it. In truth parameter sniffing is an optimization technique SQL uses when compiling query plans and for the most part it works well and significantly improves overall performance vs not using parameter sniffing. We are going to look at some examples here where parameter sniffing does cause a problem and how to solve it.

Before we get too indepth on how parameter sniffing works let's setup a database to run the examples on, throughout this post we're going to be doing things like clearing the plan cache which can should not be run on a production server so if you're following along make sure it's on a test/dev server. 

## Setup ##
Create a new database and execute the following scripts to create a table for our examples and seed it with data.

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

Let's then execute that procedure with actual query plan enabled and see what it does...

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'UK'
{% endhighlight %}

![Execution Plan filtered Index]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-tablescan.JPG)

We can see here that even though we have a NonClustered index on PlaceOfBirth the query optimizer has choosen to do a full clustered index scan. This is because it will have looked at the statistics for the PlaceOfBirth column and seen that pretty much every row matches our parameter and decided a table scan is quicker than an index seek with 655,000 key lookups.

What do you think will happen if we run the same procedure with a parameter of Mars? 

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'Mars'
{% endhighlight %}

![Execution Plan filtered Index]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-tablescan.JPG)

Not what you expected? Why would the optimizer choose to do a full clustered index scan of 655,000 records rather than seeking straight to the one matching row? If you havent guessed this is down to parameter sniffing.

## What Is Parameter Sniffing ##
The first time a parameterised query is run (Like our stored procedure above) SQL Server will optimize the query plan based on the parameters supplied and cache the plan. The next time you run that same query with different paramters the same plan will be used.

The reason for this is creating optimized query plans is an expensive operation so SQL Server caches plans so they can be reused. In most situations this works great, the reason the example above has an issue is because the parameters passed in generate a massive difference in the amount of data returned. If we have a more even split of PlaceOfBirth in our database there would be no issue.

To proove this is a caching issue lets clear the plan cache by running...

{% highlight sql %}
DBCC FREEPROCCACHE
{% endhighlight %}

No lets run our stored procedure again for Mars...

{% highlight sql %}
EXEC dbo.GetUsersByPlaceOfBirth @PlaceOfBirth = 'Mars'
{% endhighlight %}

![Execution Plan filtered Index]({{site.url}}/content/images/2017-parameter-sniffing/queryplan-index-seek.JPG)

If we look at the estimated query cost for Mars before we cleared the cache vs after we can see we've gone from 9.35973 to 0.0065704.

## Spotting Parameter Sniffing At Work ##
Normally these issues are spotted by a user reporting that a query has suddenly started going very slow, this is typically triggered by the first run of a given query on a clean cache using parameters that are outside the normal range for the shape of the data. 

There are however some ways we can try to spot these issues before they get reported....

## Preventing Problems Caused By Parameter Sniffing ##
There are several different things we can do when we find we have a parameter sniffing issue...

* Use the WITH RECOMPILE hint to force the procedure to always compile on run, generating a new plan each time

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

This obviously has an overhead over constant recompilations which depending on the complexity of the query and the frequency it's run could cause other issues.

* Use the OPTIMIZE FOR UNKNOWN query hint

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

This tells SQL to generate a plan without looking at the parameter. Although this will probably end up generating a non optimal plan. In the case of our example above it actually ends up generating the clustered index scan plan so doesnt change anything.

* Use the OPTIMIZE FOR x query hint

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

This will force SQL to use the plan that works best for the value Mars, in this case it will make our UK query very inefficient, which may be ok if that's a query that is very rarely used and we want to optimize for the value that get executed the most.

* Use seperate querys for the data that falls outside the normal range. For example PlaceOfBirth Mars clearly sits differs from most of the other data. In this case we could have 2 procedures GetUsersByPlaceOfBirthMars and keep our existing GetUsersByPlaceOfBirth. Any queries for Mars go through the new procedure. This way we get all the benefits of a cached optimized plan and take the hit by having to maintain multiple procedures.

## Fixing A Parameter Sniffing Issue In Proction ##
Often these issues a reporting by users as queries timing out, in this scenario we often need to just get the query running again before we start looking at the best plan of attack for preventing the issue. 

In the demos above I used DBCC FREEPROCCACHE to clear the cache, in production this is almost never a good idea as it will clear all plans from the cache causing the server to take a sudden hit recompiling everything. It is however an option that will most likely fix the issue.

You can also pass in a plan handle to FREEPROCCACHE to just clear a single plan from the cache, this is a must more lightweight option and if you have the plan handle this is probably the best option. You can find the plan handle by querying the sys.dm_exec_query_stats view...

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

![Execution Plan filtered Index]({{site.url}}/content/images/2017-parameter-sniffing/plan_handle.JPG)

From that we can run FREEPROCCACHE with our plan handle we want to remove...

{% highlight sql %}
DBCC FREEPROCCACHE(0x05000B00CDC1731290A264790400000001000000000000000000000000000000000000000000000000000000)
{% endhighlight %}

## Summary ##
To summarize for the most part parameter sniffing is an optimization and allows plans to be cached. When we do have issues due to parameter sniffing there are several options described above to fix and prevent the issue, all of which have various trade offs. It's all about choosing the best option for your usage. Good Luck! 