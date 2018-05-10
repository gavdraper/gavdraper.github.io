---
layout: post
title: SQL Server Database Snapshots
date: '2018-05-11 08:23:01'
---
## What Are They ##
Database snapshots are a readonly point in time version of a database. You can use them for things like readonly queries that don't need to be on live data or even as an addition to part of your backup process. For example you could create a snapshot once an hour allowing you to recover objects someone might accidentally drop, you can even restore a DB from a snapshot (Under certain conditions) if you really needed to.

## How They Work ##
Database snapshots work in a different way to normal databases. When you read from a snapshot for the most part you're reading from the source database except when the data in the source database has changed. The way it manages this is....

1. A Snapshot is created of MySourceDB called MySnapshot
1. At this point no data has changed since the snapshot so all queries against MySnapshot will query the underlying data in MySourceDB and our snapshot file will be empty.
1. Someone updates a record in MySourceDB and because SQL Server knows this database has a snapshot before the update it will store the record in our MySnapshot database.
1. Our snapshot file now contains that record before the change.
1. If we then query that record in our snapshot it will read it from the snapshot. If we query any other record it will still come from the source database as nothing else has changed.

As you can probably tell the longer a snapshot is around the larger it will get as changes are made to the source database. However short lived snapshots or snapshots on databases with relatively few updates can be very fast to create and delete.

## Using Them ##
You can create a new database snapshot from a source database with the following syntax...

{% highlight sql %}
CREATE DATABASE MySnapshot ON
(
   NAME = MySourceDb,
   FILENAME =   'C:\Dbs\MySnapshot.ss'
) AS SNAPSHOT OF MySourceDb;
{% endhighlight %}

Once created snapshots can be removed in the same way as normal databases...

{% highlight sql %}
DROP DATABASE MySnapshot
{% endhighlight %}

## Reduced Blocking ##
Let's imagine we have some scheduled reports that run throughout the day and as our database is getting larger they are being blocked more and more by other activity on the database. As discussed in previous posts one way to combat this is with things like SNAPSHOT isolation level, however this isn't always an option and if it's not then DATABASE SNAPSHOT may be a good fit.

To setup an example run the following...

{% highlight sql %}
CREATE DATABASE MySourceDB
GO
USE MySourceDB
GO
CREATE TABLE Blah(Id INT)
INSERT INTO Blah VALUES(1)
{% endhighlight %}

Then in tab 1 on this new database simulate users updating data by running the following...

{% highlight sql %}
BEGIN TRAN
   UPDATE Blah SET Id = 40
{% endhighlight %}

Now in tab 2 on the same database run this to simulate a report being blocked...

{% highlight sql %}
SELECT * FROM Blah
{% endhighlight %}

Assuming you've not changed your settings to READ COMMITTED SNAPSHOT you'll see that tab 2 just sits there waiting for the transaction in tab 1 to complete. If you run COMMIT in tab 1 then tab 2 will complete.

Let's create a DATABASE SNAPSHOT and try this again...

{% highlight sql %}
CREATE DATABASE ReportingSnapshot ON
(
   NAME = MySourceDB,
   FILENAME =   'C:\Dbs\ReportingSnapshot.ss' )
AS SNAPSHOT OF MySourceDB;
GO
{% endhighlight %}

Now run the above blocking example again but this time run tab 2 against our new snapshot database, you'll see our reports no longer get blocked by data modifications in our source database.

## Restoring Source Database ##
In most situations restoring after accidental data deletion should be done via database backups and transaction logs to get point in time. However if have a snapshot from right before the deletion the snapshot may be a quicker fix. They can be handy in dev environments where you want to try something but want the option to revert the datbase after, a backup/restore can take a long time on larger databases but a snapshot/restore can be quite quick depending on how much data changes after the snapshot is taken.

You have a couple of options depending on what you are recovering from you could just recreate dropped objects and copy the data from the snapshot back into the source database. Alternatively you can actually restore the source database back to the snapshot (Warning : This will not include any data changed after the snapshot was created).

To restore a database with a snapshot neither database can be corrupt and the source database can only have a single snapshot. If you have multiple snapshots you'll need to remove the others first.

The restore looks like this...

{% highlight sql %}
RESTORE DATABASE MySourceDB
FROM DATABASE_SNAPSHOT = 'ReportingSnapshot'
{% endhighlight %}

At this point if you're in full or bulk recovery you'll want to take a full backup or your log backups are not going to run as now we've run the restore the database has no full backup to base it's log backups on.