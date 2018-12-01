---
layout: post
title: SQL Server, In Memory OLTP
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
{% endhighlight %}

Lets now try our blocking test from earlier, a new tab run this...

{% highlight sql %}
BEGIN TRAN
UPDATE Optimist WITH(SNAPSHOT) SET Title = 'Gavin Draper' 
{% endhighlight %}

Then in a second tab run this...

{% highlight sql %}
BEGIN TRAN
UPDATE Optimist WITH(SNAPSHOT) SET Title = 'Gavin D'
{% endhighlight %}

The second query will error instantly with...

   The current transaction attempted to update a record that has been updated since this transaction started. The transaction was aborted.

Rather than taking locks to track what transactions are touching what data In Memory OLTP Tables maintain rowversions in the rows, this means transactions can check if a row is currently being changed by looking at it's rowversion and fail instantly. If you're using In Memory OLTP tables then you need to handle retries for these kind of errors in your app.

So far this probably seems like Optimistic Concurrency doesn't have many benefits ie Error vs Waiting for a lock. The big benefits come outside of 2 transactions trying to write the same data, the whole point of optimistic concurrency is that it's optimistic that most of the time 2 or more transactions will not be trying to write to the same data. Optimistic concurrency will happily let you read and write data in separate transactions without taking out any locks.

As I mentioned above I'm just touching on a very small part of In Memory OLTP here and I plan to do more posts around the other features but it's worth also mentioning here that only a limited set of isolation levels are available on these tables Snapshot, Repeatable Read and Serializable.
