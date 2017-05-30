---
layout: post
title: SQL Server Filtered Indexes By Example
date: '2017-05-30 08:23:18'
---

## What Are Filtered Indexes ##

This post will make more sense if you already know what non filtered indexes are and how they work, see [SQL Server Clustered & NonClustered Indexes Explained](https://gavindraper.com/2017/05/16/clustered-and-nonclustered-indexes/) for more information.

Filtered indexes are essentially an index with a predicate on them filtering the data that gets stored in the index. For example on a bug tracking system I could creatre a fitlered index that just indexed bugs with a status of open.

## When To Use Filtered Indexes ##

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

![Group by Result Set]({{site.url}}/content/images/2017-filtered-index/data-distribution.JPG)

Let's assume the home screen of our application lists the last 100 raised bugs that have a status of open...

{% highlight sql %}
SELECT TOP 100
	Bugs.Id,
	Bugs.Title,
	Bugs.[Description]
FROM
    dbo.Bugs
WHERE
	Bugs.BugStatus = 1
ORDER BY id DESC
{% endhighlight %}

From this we'll get the following execution plan...

![Execution Plan No Index]({{site.url}}/content/images/2017-filtered-index/execution-plan.JPG)

We can see we're doing a clusted index scan to order by id and filter just the open bugs as expected. Let's look at how many reads are being performed on that index...

At this point we could create NonClustered index status for the whole table and that would work fine. However we will then be maintaining an index for 99.9% of the table that we're never going to use beause we're only interested in open bugs. This is where a filtered index is a good fit...

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_bugs_bugstatus_open 
    ON dbo.Bugs(id)  
    INCLUDE(BugStatus, [Description], Title)
    WHERE BugStatus = 1
{% endhighlight %}

So here we're creating an index on id, including all the fields our query uses to avoid and key lookups and then we're filtering by BugStatus = 1. This means when our query does BugStatus=1 SQL Server will know all the rows in that index satisfy our predicate. 

If we run our select query again...

![Execution Plan filtered Index]({{site.url}}/content/images/2017-filtered-index/filtered-index.JPG)

One thing to note filtered indexes will not be used if your predicate in your query is a parameter as SQL Server wont know up front when it creates the plan if the filtered index will cover any parameter you're going to pass it. 