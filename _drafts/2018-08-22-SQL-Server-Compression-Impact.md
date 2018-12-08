---
layout: post
title: SQL Server Impacts of Compression
date: '2018-08-17 13:34:01'
---
I recently had an issue where a server's CPU load was running a lot higher than usual, after studying a profile for while it quickly highlighted the offending queries. What wasn't obvious was why some of these simple queries were using so much CPU, the only thing they had in common is that they were temporal queries using the AS OF syntax to go across the main and history table. After a little head scratching I realized all the history tables were compressed at the page level, it turns out history tables by default are compressed. After rebuilding the table to turn of compression the CPU load dropped right back down. Problem solved.

This got me thinking though, I've never worked on a system where I've needed to compress tables so had never really looked at what compression does to query performance and their plans. I'm going to delve into this in this post.

All examples in this post are based on a data dump of Stack Overflow from 2010 mainly because it's a nice size and well distributed test bed.

First let's take a baseline against the non compressed posts table...

{% highlight sql %}
SELECT TOP 5 
  Users.DisplayName,
  COUNT(*)
FROM 
  Posts
  INNER JOIN Users ON Users.Id = Posts.OwnerUserId
GROUP BY 
  Posts.OwnerUserId,Users.DisplayName
ORDER BY COUNT(*) DESC
OPTION (MAXDOP 1)
{% endhighlight %}

It's a straight forward query that gets the top 5 users with the highest post count. I've added MAXDOP 1 here to stop the process going parallel, the reason for this is our before and after test will have different costs which may cause SQL Server to make one parallel which will make it hard to compare like for like.

Before I ran these 2 queries I turned on STATISITCS IO and TIME and got the following results...

> Table 'Worktable'. Scan count 0, logical reads 0, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> Table 'Workfile'. Scan count 5, logical reads 2912, physical reads 339, read-ahead reads 2573, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> Table 'Posts'. Scan count 1, logical reads 801623, physical reads 1, read-ahead reads 801168, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> CPU time = 8515 ms,  elapsed time = 18315 ms.

The query plan looked like this...

As you can see the SELECT had an estimated subtree cost of 652.

Now lets compress the clustered index on the posts table...

{% highlight sql %}
ALTER TABLE Posts REBUILD  WITH (DATA_COMPRESSION = PAGE)
{% endhighlight %}

The table size has gone from..

If we now run the posts query again the query plan now looks like this...

Before running both these queries I turned on STATISTICS IO and STATISTICS TIME to get info on reads and CPU load, the before and after looks like this...

It's quite clear having seen this that...