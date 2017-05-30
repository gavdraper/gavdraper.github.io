---
layout: post
title: SQL Server Filtered Indexes By Example
date: '2017-05-30 08:23:18'
---

## What Are Filtered Indexes ##

This post will make more sense if you already know what non filtered indexes are and how they work, see [SQL Server Clustered & NonClustered Indexes Explained](https://gavindraper.com/2017/05/16/clustered-and-nonclustered-indexes/) for more information.

Filtered indexes are essentially an index with a predicate on them filtering the data that gets stored in the index. For example on a bug tracking system I could creatre a fitlered index that just indexed bugs with a status of open.

## When To Use filtered Indexes ##

Filtered indexes make sense when you're constantly querying a small portion of the overall dataset and are less interested in the rest of it. They allow you to have a smaller index which means you can get to your data faster with less reads. If you get it right based on your data in query needs you'll reduce the writes required to maintain the index and the reads required for the query to complete. 

## Examples ##

Let's setup an example bugs table for our bug tracking software with some example statuses

{% highlight sql %}
CREATE TABLE dbo.BugStatus
(
	Id INT IDENTITY PRIMARY KEY,
	[Status] NVARCHAR(20)
)

INSERT INTO dbo.[BugStatus]([Status])
VALUES
	('Open'),
	('Closed'),
	('OnHold'),
	('InTest'),
	('InProgress')

CREATE TABLE dbo.Bugs
(
	Id INT IDENTITY PRIMARY KEY,
	Title NVARCHAR(100),
	[Description] NVARCHAR(MAX),
	BugStatus INT FOREIGN KEY REFERENCES dbo.BugStatus(Id)
)
{% endhighlight %}

Assuming we're actually fixing bugs then overtime 99% of bugs will be Closed and for the most part we're not interested in querying those bugs, in this situation a filtered index on open bugs may be a good idea as that's the most common data set to query on and it's a very small set of the overall data.

Let's fake some data to test this on...

{% highlight sql %}
DECLARE @InsertCount INT
--Add closed bugs
INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
VALUES('Closed Title Blah','My Description',2)
SELECT @InsertCount = 0
WHILE @InsertCount < 20
	BEGIN
	INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
	SELECT Title,[Description],BugStatus FROM Bugs WHERE BugStatus = 2
	SET @InsertCount = @InsertCount + 1
	END

--Add InTest
INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
VALUES('In TestTitle Blah','My Description',4)
SELECT @InsertCount = 0
WHILE @InsertCount < 3
	BEGIN
	INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
	SELECT Title,[Description],BugStatus FROM Bugs WHERE BugStatus = 4
	SET @InsertCount = @InsertCount + 1
	END

--Add OnHold
INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
VALUES('On Hold Title Blah','My Description',3)
SELECT @InsertCount = 0
WHILE @InsertCount < 5
	BEGIN
	INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
	SELECT Title,[Description],BugStatus FROM Bugs WHERE BugStatus = 3
	SET @InsertCount = @InsertCount + 1
	END

--Add Open
INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
VALUES('Open Title Blah','My Description',1)
SELECT @InsertCount = 0
WHILE @InsertCount < 7
	BEGIN
	INSERT INTO dbo.Bugs(Title,[Description],BugStatus)
	SELECT Title,[Description],BugStatus FROM Bugs WHERE BugStatus = 1
	SET @InsertCount = @InsertCount + 1
	END	
{% endhighlight %}

This gives us a good distribution of Status to work with...

![Group by Result Set]({{site.url}}/content/images/2017-filered-index/data-distribution.JPG)

Lets now create a NonClustered index on BugStatus on the bugs table...

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_bugs_bugstatus ON dbo.bugs(BugStatus)
{% endhighlight %}

Let's assume the home screen of our application lists all open bugs and this is the most used query in the system...

{% highlight sql %}
SELECT
	Bugs.Id,
	Bugs.Title,
	Bugs.[Description]
FROM
	dbo.Bugs
WHERE
	Bugs.BugStatus = 1
{% endhighlight %}

From this we'll get the following execution plan...

![Execution Plan For Normal Index]({{site.url}}/content/images/2017-filered-index/execution-plan.JPG)

We can see we're using our NonClustered index to filter just the open bugs as expected. Let's look at how many reads are being performed on that index...

IMAGE HERE SHOWING IO

Now let's create a filtered index on BugStatus for just open bugs.

{% highlight sql %}

CREATE NONCLUSTERED INDEX ndx_bugs_bugstatus_open ON dbo.Bugs(BugStatus) 
WHERE BugStatus = 1
{% endhighlight %}

If we run our select query again...

EXECUTION PLAN HERE SHOWING NEW FILTERED INDEX

IO COUNT HERE

We can see from this we've reduced the amount of reads required to complete our query. Not only that but we've reduced the amount of writes required to maintain the index as any changes on data that does not have a status of open will not cause an index write. There is a bit of trial and error involved in getting fitlered index right as it really depends on the shape of your data and the queries that are using it.