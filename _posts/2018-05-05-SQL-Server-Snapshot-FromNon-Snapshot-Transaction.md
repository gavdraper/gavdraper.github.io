---
layout: post
title: Avoid Errors With using SNAPSHOT Isolation From Transactions Under Different Isolation Levels
date: '2018-05-05 08:17:12'
tags: isolation-levels
---
For older systems where the move to READ COMMITTED SNAPSHOT (RCSI) might not be an option or you're running a version of SQL Server that doesn't support this then you can fall back to an adhoc SNAPSHOT isolation option. This is a great way to dip your toe in to the RCSI world without going all in.

Using this method you can set specific procedures to SNAPSHOT without having everything in the DB use SNAPSHOT and having to deal with problems that may arise with it. A great way to slowly move your DB to SNAPSHOT.

One problem with this solution is that if you set a stored procedure to SNAPSHOT then call it from a transaction that is not running under snapshot then it will fail with the following error.

> Transaction failed in database 'Test' because the statement was run under snapshot isolation but the transaction did not start in snapshot isolation. You cannot change the isolation level of the transaction to snapshot after the transaction has started unless the transaction was originally started under snapshot isolation level.

This makes life a bit more difficult when you have one stored procedure that you want to use snapshot but it's called from lots of different places that may be hard to identify (A common problem on legacy systems).

We can actually check for this and only apply SNAPSHOT if it's safe to do so. The easiest way to do this is to check he transaction count and if it's 0, If it is then you know you're safe to set SNAPSHOT as then we know it's not being called from another transaction that could be on a different isolation level.

For Example...

{% highlight sql %}
CREATE PROCEDURE MySnapShotProc
AS
IF @@TRANCOUNT = 0
   SET TRANSACTION ISOLATION LEVEL SNAPSHOT

SELECT Blah FROM Blah
{% endhighlight %}

This approach makes it safer to opt into SNAPSHOT on a procedure by procedure basis without having to worry about where it's called from.
