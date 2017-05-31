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

## What Can You Do About It? ##
