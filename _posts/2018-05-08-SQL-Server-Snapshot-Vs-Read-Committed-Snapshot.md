---
layout: post
title: SQL Server Snapshot vs Read Committed Snapshot
date: '2018-05-08 20:56:51'
---
For a quick introduction to transaction isolation levels see my [previous post](https://gavindraper.com/2012/02/18/sql-server-isolation-levels-by-example/) on them.

Once snapshot isolation is enabled on a given database you can use it just like any other transaction level by setting it on any given query. Read Committed Snapshot (Also known as RCSI for read committed snapshot isolation which is how I will refer to it here on in.) however doesn't get used per query instead you turn this on for the database and Read Committed then becomes RCSI making this your default isolation level.

A common misunderstanding is that RCSI is just a way to make Snapshot the default isolation level, however this is not the case RCSI and Snapshot do actually behave differently.

The way Snapshot works is whenever the transaction reads data it stores a snapshot of it in TempDB and will only ever read that snapshot for the lifetime of the transaction, so if another transaction commits changes and the snapshot transaction has already taken a snapshot of those records it wont see those changes. Because snapshot isolation keeps the snapshots for the lifetime of the transaction long running transactions that run lots of statements can create a lot of data in TempDB. Another problem with snapshot is any data you modify inside a snapshot transaction will when it commits check it's not been modified since the transaction started and if it has it will error and rollback the whole transaction.

RCSI has a lot of the benefits of snapshot e.g readers don't block writers and writers don't block readers but it also removes some of it's issues. RCSI rather than snapshoting the whole batch only snapshots at statement level this means snapshots are shorter lived. It also removes the restriction that an update somewhere in the batch will cause the whole batch to error and rollback on commit if the data has been modified in another transaction.

Let's demo some of these features, first setup a new database and table to test on...

{% highlight sql %}
CREATE DATABASE SnapshotTest
GO
ALTER DATABASE SnapshotTest SET ALLOW_SNAPSHOT_ISOLATION ON
ALTER DATABASE SnapshotTest SET READ_COMMITTED_SNAPSHOT ON
GO
USE SnapshotTest
GO
CREATE TABLE SnapshotTest(Blah INT)
GO
INSERT INTO SnapshotTest VALUES(1)
INSERT INTO SnapshotTest VALUES(2)
INSERT INTO SnapshotTest VALUES(3)
{% endhighlight %}

Drop the database and rerun the above script for each section below to reset the test environment.

## Readers Don't Block Writers ##

Tab 1
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   SELECT * FROM SnapshotTest
{% endhighlight %}

Tab 2
{% highlight sql %}
UPDATE SnapshotTest SET Blah = 10 WHERE Blah = 1
{% endhighlight %}

You can see the update ran fine even though we'd looked at those records in our still running tab 1 transaction. Now commit the Tab 1 transaction and run the experiment again but this time with tab 1 set to READ COMMITTED which will be RCSI as we've turned that on for this database, you'll see the exact same results.

## Writers Don't Block Readers ##

Tab 1
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   UPDATE SnapshotTest SET Blah = 10 WHERE Blah = 1
{% endhighlight %}

Tab 2
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   SELECT * FROM SnapshotTest
{% endhighlight %}

As with before neither transaction blocked the other. Commit both tabs then change both to READ COMMITTED to see that has the same results.

## Lifetime Of Snapshots ##
This is where things start to behave differently...

Tab 1
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   SELECT * FROM SnapshotTest
{% endhighlight %}

Tab 2
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   UPDATE SnapshotTest SET Blah = 10 WHERE Blah = 1
COMMIT
{% endhighlight %}

Tab 1
{% highlight sql %}
   SELECT * FROM SnapshotTest
{% endhighlight %}

You'll see the second select in Tab 1 isn't seeing the changes that tab 2 committed. This is because tab 1 took a snapshot of that table the first time we ran a select against it and will keep that snapshot until the transaction ends. Commit Tab 1 then drop and recreate the DB with the above script and run the test again as READ COMMITTED. You'll see that RCSI will see committed changes as they happen as it renews it's snapshots on every statement.

## Concurrent Updates Cause Rollbacks ##

Tab 1
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   UPDATE SnapshotTest SET Blah = 10 WHERE Blah = 1
{% endhighlight %}

Tab 2
{% highlight sql %}
SET TRANSACTION ISOLATION LEVEL SNAPSHOT
BEGIN TRAN
   UPDATE SnapshotTest SET Blah = 10 WHERE Blah = 2
COMMIT
{% endhighlight %}

Tab 1
{% highlight sql %}
COMMIT
{% endhighlight %}

You'll see at this point transaction 2 errors because it attempted to change data that has already been changed. If there had have been other operations in Tab 2 before the offending update they would have all have been rolled back.

If you then change this test to use READ COMMITTED there will be no error and the only tab that actually makes any changes will be that last one that commits with no risk of rolling a whole batch back and had there have been other modifications in the batch they would have gone through providing they haven't been changed elsewhere in which case they would also get ignored.

## Conclusion ##
Both Snapshot and RCSI have their pros and cons and you need to do when to use one over the other or even when to avoid both of them.

Reading above is may sound like RCSI is a golden ticket but there are times that snapshot fits better. For example a procedure that runs lots of statements that all need to be consistent with each other, snapshot will use the same snapshots for the life of batch making this a better fit.

Also beware that applications can rely on blocking and that changing an existing database to RCSI can really screw this up. For example if an application depends on a writer blocking a reader then RCSI can cause big problems...

Tab 1
{% highlight sql %}
BEGIN TRAN
  UPDATE AvailableTickets SET TotalTickets = TotalTickets -1
{% endhighlight %}

Tab 2
{% highlight sql %}
BEGIN TRAN
  SELECT TotalTickets FROM AvailableTickets
{% endhighlight %}

In normal read committed tab 2 will wait for tab 1 to complete and return the committed value. In RCSI however we will return the old value without waiting for the update to commit possibly leading to tickets being booked twice. Applications using RCSI need to be aware of this and legacy applications should be switched to RCSI with caution/lots of testing. For legacy apps sprinkling snapshot onto queries that are safe to do so may be a better option than going all in with RCSI.

