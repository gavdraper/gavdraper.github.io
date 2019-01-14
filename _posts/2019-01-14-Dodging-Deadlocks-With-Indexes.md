---
layout: post
title: Dodging Deadlocks With Indexes
date: '2019-01-14 04:34:01'
tags: performance-tuning indexing
---
A lot of people don't realise that some deadlocks can be removed entirely with the introduction of a new index. The most commonly talked about deadlock solutions that I see are ...

* Switch the order locks are taken out in your queries (If it's possible or makes sense)
* Reduce the chance of the offending queries running at the same time
* Add error handling to the app to auto retry when deadlocks do occur

All of the above are good practices and should be done but I also wanted to cover here how some deadlocks can be solved with the introduction of an index. The ways in which an index can help are...

* Speed up the query, causing locks to be held for less time and reduce the chance of deadlock
* Covering indexes remove the need for the underlying clustered index to be touched on selects. If your deadlock is being caused by shared locks from a select on a small subset of the full rows then you may be able to move these locks to a new index whereby the two offending queries don't need locks on the same data.

Let's look at a demo of how a covering index can help, if you want to follow along you'll need a copy of the [Stack Overflow Database](https://www.brentozar.com/archive/2015/10/how-to-download-the-stack-overflow-database-via-bittorrent/).

To simulate the deadlock run these queries in two separate tabs in the following order...

<table>
<tr><th>Tab 1</th><th>Tab 2</th></tr>
<tr><td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;">
{% highlight sql %}
BEGIN TRAN
/*EXCLUSIVE LOCK ON VOTES*/
UPDATE [Votes] SET BountyAmount = 10 WHERE id = 1
{% endhighlight %}
</td><td style="border-bottom:0px solid;border-left:0px solid;;border-right:0px solid;"></td></tr>
<tr>
<td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;"></td>
<td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;">
{% highlight sql %}
BEGIN TRAN
/*EXCLUSIVE LOCK ON USERS*/
UPDATE [Users] SET Age = 10 WHERE Id = 1	
{% endhighlight %}
</td>
</tr>
<tr><td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;">
{% highlight sql %}
/*Attempt to take a bunch of shared locks on users*/
SELECT TOP 100
 Location,
 COUNT(*)
FROM
 [Users]
GROUP BY [Location]
ORDER BY COUNT(*) DESC
{% endhighlight %}
</td><td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;"></td></tr>
<tr>
<td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;"></td>
<td style="border-bottom:0px solid;border-left:0px solid;border-right:0px solid;">
{% highlight sql %}
/* Attempt to take a bund of shared locks on votes*/
SELECT TOP 100 
 UserId,
 COUNT(*)
FROM
 [Votes]
GROUP BY UserId
ORDER BY COUNT(*) DESC	
{% endhighlight %}
</td>
</tr>
</table>

Boom, Deadlock!

Now let's take a step back and see how a couple of indexes can completely avoid this deadlock. First ROLLBACK both existing transactions.

Both our queries UPDATE statements end up with an Exclusive Key lock on their respective tables clustered indexes. This can be seen by running...

{% highlight sql %}
BEGIN TRAN
UPDATE [Votes] SET BountyAmount = 10 WHERE id = 1

SELECT
   OBJECT_NAME(p.object_id) Object,
   i.name IndexName,
   l.resource_type ObjectType,
   l.request_mode LockLType,
   l.request_status LockStatus
FROM sys.dm_tran_current_transaction t
   INNER JOIN sys.dm_tran_locks l ON l.request_owner_id = t.transaction_id
   INNER JOIN sys.partitions p ON p.hobt_id = l.resource_associated_entity_id
   LEFT JOIN sys.indexes i ON i.object_id = p.object_id AND i.index_id = p.index_id
ROLLBACK
{% endhighlight %}

![Key lock transaction]({{site.url}}/content/images/2019-Indexing-Deadlocks/key-lock.PNG)

Notice that the Exclusive Key lock is on the clustered key (pk_votes_id) of the votes table. 

So to avoid the deadlock we need to make sure the respective select query we ran in Tab2 against the Votes table doesn't block on this key lock. If we refer back to our SELECT query...

{% highlight sql %}
SELECT TOP 100 
 UserId,
 COUNT(*)
FROM
 [Votes]
GROUP BY UserId
ORDER BY COUNT(*) DESC
{% endhighlight %}

We can see the only field our query touches is UserId and that UserId is not changed at all in our Update statement. What this means is that if we create an index on UserId that index will not be updated or locked as part of our UPDATE statement and our SELECT query can use that index to run lock free...

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_votes__user ON Votes(UserId)
{% endhighlight %}

Just to prove that our update won't touch this new index lets run it again and check it still only takes one key lock on the Clustered index with no locks on our new NonClustered index.

{% highlight sql %}
BEGIN TRAN
UPDATE [Votes] SET BountyAmount = 10 WHERE id = 1

SELECT
   OBJECT_NAME(p.object_id) Object,
   i.name IndexName,
   l.resource_type ObjectType,
   l.request_mode LockLType,
   l.request_status LockStatus
FROM sys.dm_tran_current_transaction t
   INNER JOIN sys.dm_tran_locks l ON l.request_owner_id = t.transaction_id
   INNER JOIN sys.partitions p ON p.hobt_id = l.resource_associated_entity_id
   LEFT JOIN sys.indexes i ON i.object_id = p.object_id AND i.index_id = p.index_id
ROLLBACK
{% endhighlight %}

![Key lock transaction]({{site.url}}/content/images/2019-Indexing-Deadlocks/key-lock.PNG)

Great! Our new index is lock free and our SELECT query can now use it without being blocked.

The same can be done with our query against the Users table, again the read query is only touching one field (Location) which is not used in our update...

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_user_location ON Users(Location)
{% endhighlight %}

With these two new indexes if you then run the queries in two tabs again you'll notice there is no longer a deadlock and even no blocking between them.

Winning! As with all things you need to find the tool that works best for your situation. This isn't a one stop deadlock fix but rather just another tool in your toolkit to use when it makes sense. 