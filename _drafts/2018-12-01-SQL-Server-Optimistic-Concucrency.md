---
layout: post
title: SQL Server, What? Wait... Thing Can Do Optimistic Concurrency?
date: '2018-11-30 09:34:01'
---
SQL Server uses the Pessimistic Concurrency, this means transactions acquire locks upfront to stop future transactions changing data under them, for example an update statement will lock that rows in question to prevent another query also changing them. Locks in SQL Server are not free and at certain thresholds SQL will escalate lock levels as taking thousands of row locks gets expensive quick so at some point it will switch to page locks and then table locks. This is why queries are often blocked in SQL server waiting for another query to finish.

If you have a particular table in your system where blocking is a real problem there is a different engine inside the SQL Database Engine that does use the Optimistic Concurrency model. This concurrency model works differently in that it doesn't acquire locks for different transactions but instead it versions rows and if 2 transactions try to change the same row the second transaction will fail. This concurrency model is only available on In Memory OLTP tables which was introduced in SQL Server 2014. There are all sorted of caveats and limitations with in memory tables and you should read up on it more before committing, in this post I'm just going to demo the optimistic concurrency side of things.

Lets first create a test normal table to see how block behaves in pessimistic mode....

{% highlight sql %}
CREATE TABLE Pessimist
(
   Id INT IDENTITY,
   Title NVARCHAR(100)
)
INSERT INTO Pessimist (Title) VALUES ('Gavin')
{% endhighlight %}

Now lets start a transaction but not finish it, open a new tab and run this...

{% highlight sql %}
BEGIN TRAN
UPDATE Pessimist SET Title = 'Gavin Draper'
{% endhighlight %}

Now open a new tab and run this

{% highlight sql %}
BEGIN TRAN
UPDATE Pessimist SET Title = 'Gavin D'
{% endhighlight %}

If you then run a script like sp_whoisactive or look at sys.dm_exec_requests you'll see our second query is blocked behind our first query. Now run COMMIT in tab 1 and then tab 2. If you then check the data in our Pessimist table you'll see it's now 'Gavin D' as that was the last query to commit so it overwrote the first query. 

Lets now setup the same test with in memory tables.

To create an in memory table you need to have an in memory filegroup on the database you are working in with a file in it...

{% highlight sql %}

ALTER DATABASE DevDemos 
ADD FILEGROUP DevDemosMemoryOptimized CONTAINS MEMORY_OPTIMIZED_DATA

ALTER DATABASE DevDemos
ADD FILE (name = DevDemoMemoryOptimized, filename = 'c:\temp\')
TO FILEGROUP DevDemosMemoryOptimized
{% endhighlight %}

At this point we can create our in memory table and add a row...

{% highlight sql %}
CREATE TABLE Optimist
(
   Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
   Title NVARCHAR(100)
)
WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA)

INSERT INTO Pessimist (Title) VALUES ('Gavin')
{% highlight sql %}

Lets now try our blocking test from earlier, a new tab run this...

{% highlight sql %}
BEGIN TRAN
UPDATE Optimist WITH(SNAPSHOT) SET Title = 'Gavin Draper' 
{% endhighlight %}