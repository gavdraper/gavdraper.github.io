---
layout: post
title: SQL Server Removing Duplicates
date: '2019-01-03 08:13:23'
---
The best solution to this is to not allow duplicates into your data in the first places but let's imagine someone did let the little critters in, how can we then write a query that will remove the duplicates?

We actually have a few options, some good, some not so good. Before we run through them if you want to follow along then run the following SQL to create two tables and populate them with some duplicates data...

{% highlight sql %}
CREATE TABLE DupesWithNoUniqueKey
(
    Superhero NVARCHAR(20)
)
GO
CREATE TABLE DupesWithUniqueKey
(
    Id INT IDENTITY PRIMARY KEY,
    Superhero NVARCHAR(20)
)
GO

INSERT INTO DupesWithNoUniqueKey (SuperHero)
SELECT 'Batman'
UNION ALL SELECT 'Batman'
UNION ALL SELECT 'Batman'
UNION ALL SELECT 'Black Widow'
UNION ALL SELECT 'Black Widow'
UNION ALL SELECT 'Luke Cage'
UNION ALL SELECT 'Jessica Jones'

INSERT INTO DupesWithUniqueKey (SuperHero)
SELECT 'Batman'
UNION ALL SELECT 'Batman'
UNION ALL SELECT 'Batman'
UNION ALL SELECT 'Black Widow'
UNION ALL SELECT 'Black Widow'
UNION ALL SELECT 'Luke Cage'
UNION ALL SELECT 'Jessica Jones'
{% endhighlight %}


## De-Duping Without Windows Functions ##
Let's imagine for whatever reason you're still rocking SQL Server 2000 and don't have access to the hotness that are window functions. What are our options for de-duping data?

We currently have 2 tables, one with a unique key and one without. Let's start with the one with no unique key...

In DupesWithNoUniqeKey table with have absolutely nothing unique to use in our filters to say delete this record but leave at least one copy, we also don't have access to window functions so we can't stick a row number on and delete all with a row number greater than one. This really leaves us with only one option and it's a pretty painful option especially if it's a large table....

{% highlight sql %}
BEGIN TRAN
SELECT DISTINCT SuperHero INTO #NoDupes FROM DupesWithNoUniqueKey
TRUNCATE TABLE DupesWithNoUniqueKey
INSERT INTO DupesWithNoUniqueKey
SELECT SuperHero FROM #NoDupes
COMMIT
{% endhighlight %}

We're having to do a SELECT DISTINCT into a new table to remove the duplicates, then we're having to clear out our original table and re-insert all the data. 

Let's now look at our other table that does have a unique key which allows us to differentiate between the duplicate records...

We can run the following query to see all the duplicate records and filter out the first occurence of them...

{% highlight sql %}
WITH cte_dupes AS
(
SELECT 
    d.*
FROM 
    DupesWithUniqueKey d
    INNER JOIN (
        SELECT MIN(id) minId, SuperHero 
        FROM DupesWithUniqueKey 
        GROUP BY SuperHero
    ) m ON m.SuperHero = d.SuperHero
WHERE   
    id <> m.minId
)
SELECT * FROM cte_dupes
{% endhighlight %}

Once we've run that and looked at the output to confirm it's returning the record we want to delete we can use the same CTE to do our delete...

{% highlight sql %}
;WITH cte_dupes AS
(
SELECT 
    d.*
FROM 
    DupesWithUniqueKey d
    INNER JOIN (
        SELECT MIN(id) minId, SuperHero 
        FROM DupesWithUniqueKey 
        GROUP BY SuperHero
    ) m ON m.SuperHero = d.SuperHero
WHERE   
    id <> m.minId
)
DELETE s
FROM cte_dupes 
    INNER JOIN DupesWithUniqueKey s 
        ON s.id = cte_dupes.id
{% endhighlight %}

This time we're only touching the deleted rows and not having to move the whole table.

## De-Duping With Windows Functions ##
Things get a lot better with the introudction of Window Functions in SQL Server 2005, we no longer need to worry about having a unique key or some way to differentiate duplicates from each other...

We can now use PARTITION BY to define what needs to be unique in order for it to be classed as a duplicate and ROW_NUMBER() to give each duplicate a ascending ID. At which point we can then delete anything with an ID > 1...

{% highlight sql %}
WITH cte_dupes AS
(
    SELECT ROW_NUMBER() OVER (PARTITION BY SuperHero ORDER BY(SELECT NULL)) AS RowNumber
    FROM DupesWithNoUniqueKey
)
DELETE FROM cte_dupes WHERE RowNumber<>1
{% endhighlight %}

Notice we're orders by SELECT NULL this is because we have no real ID and so don't create which version of a duplicate gets deleted. 

Let's imagine on our other table that does have a duplicate ID we only want records after the first one to be deleted, we can do that by ordering by ID in our partition....

{% highlight sql %}
WITH cte_dupes AS
(
    SELECT ROW_NUMBER() OVER (PARTITION BY SuperHero ORDER BY id) AS RowNumber
    FROM DupesWithUniqueKey
)
DELETE FROM cte_dupes WHERE RowNumber<>1
{% endhighlight %}

It's worth noting that everything above can be achieved without the use of CTE's, I use them so I can first run a select against the CTE to check what I'm going to be deleting, for example...

{% highlight sql %}
WITH cte_dupes AS
(
    SELECT *, ROW_NUMBER() OVER (PARTITION BY SuperHero ORDER BY id) AS RowNumber
    FROM DupesWithUniqueKey
)
--DELETE FROM cte_dupes  WHERE RowNumber<>1
SELECT * FROM cte_dupes WHERE RowNumber<>1
{% endhighlight %}

In the above I've changed the CTE to "SELECT *" so we can see all the data and also commented out the delete statement to replace it with a select. With this method you can always run the select first and when happy just comment out and uncomment the delete line. 