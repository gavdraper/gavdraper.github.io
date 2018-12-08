---
layout: post
title: SQL Server Adventures In Reducing Reads
date: '2018-12-08 09:34:01'
---
After recently spending some time tuning a query where the bottleneck was IO due to the query having a huge amount of reads I thought I'd run some different tests in approached to reducing reads. I'm not suggesting anything here should be blindly followed as with all things there are trade offs. but the results are I think interesting none the less. 

Let's take this fairly simple query on the Stack Overflow data dump database to get the top 5 users by posts and return their name and the amount of posts they have...

{% highlight sql %}
SET STATISTICS IO ON
SET STATISTICS TIME ON

SELECT TOP 5 
  Users.DisplayName,
  COUNT(*)
FROM 
  Posts
  INNER JOIN Users ON Users.Id = Posts.OwnerUserId
GROUP BY 
  Posts.OwnerUserId,Users.DisplayName
ORDER BY COUNT(*) DESC
{% endhighlight %}

On my laptop the statistics output I get is this...

> Table 'Posts'. Scan count 5, logical reads 800230, physical reads 0, read-ahead reads 797892, 
> lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> Table 'Users'. Scan count 0, logical reads 121, physical reads 39, read-ahead reads 0, 
> lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

>  SQL Server Execution Times: CPU time = 7294 ms,  elapsed time = 7178 ms.

So roughly 7 seconds to execute and our biggest hit on reads is 799999 on the Posts table and 121 on Users. Let's start on the small but easy low hanging fruit. 

## Lightweight Covering Index ##
The user table is currently scanning the clustered index to get the DisplayUsername, because the clustered index contains all the fields it's having to read data we don't need, we can create a lightweight covering index with just Id and an include on DisplayName.

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_user_id_include_displayname 
	ON Users(Id) INCLUDE(DisplayName)
{% endhighlight %}

No lets try our query again...

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
{% endhighlight %}

> Table 'Users'. Scan count 0, logical reads 69, physical reads 21, read-ahead reads 0, 
> lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

It's a small gain but we've halved the reads required for our join to the users table.

Now to tackle that posts table, this is a bit more difficult as we have to touch every record in order to group and count. An obvious place to start is to create a smaller index than the clustered index it's currently using so only the field we need in order to satisfy our group gets read...

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_posts_owner_userId
	ON Posts(OwnerUserId)
{% endhighlight %}

Then let's try our query again...

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
{% endhighlight %}

> Table 'Posts'. Scan count 5, logical reads 6539, physical reads 0, read-ahead reads 11, 
> lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> SQL Server Execution Times: CPU time = 1157 ms,  elapsed time = 738 ms.

For me the query now runs in less than a second and our reads on the posts table have gone from 799999 down to 6539. We could happily stop here (and in this case probably should) but in the interest of this post I wanted to see how much further I could take this.

We're now at the point where our query is reading only information it absolutely needs in order to complete, so how can we reduce reads further? Compression! 

## Compressed Indexes ##
We have a couple of options here, we can compress our indexes with Row or Page level compression or we can get fancy and use a Columnstore index. Let's compare these options...

First up lets set our posts index to use page compression...

{% highlight sql %}
ALTER INDEX ndx_posts_owner_userId 
	ON Posts REBUILD WITH (DATA_COMPRESSION = PAGE)
{% endhighlight %}

Then let's try our query again...

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
{% endhighlight %}

> Table 'Posts'. Scan count 5, logical reads 4137, physical reads 0, read-ahead reads 16, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

> SQL Server Execution Times: CPU time = 1457 ms,  elapsed time = 695 ms.

That's just over another 2000 reads knocked off. We've traded off reads for CPU A little here as the compression/decompression process will add overhead on the processor. 

## Columnstore ##
Let's now drop our compressed index and try a Columnstore index...

{% highlight sql %}
DROP INDEX ndx_posts_owner_userId ON Posts
CREATE NONCLUSTERED COLUMNSTORE INDEX ndx_cs_owner_user_id ON Posts(OwnerUserId)
{% endhighlight %}

I'll leave you to read more about Columnstore elsewhere but just know they work great on large reporting tables with lots of duplication, because of the way they store data the duplicates are only stored once.

Now let's see what this does...

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
{% endhighlight %}

> Table 'Posts'. Scan count 2, logical reads 0, physical reads 0, read-ahead reads 0, 
> lob logical reads 6382, lob physical reads 0, lob read-ahead reads 22818.

> Table 'Posts'. Segment reads 5, segment skipped 0.

> SQL Server Execution Times: CPU time = 187 ms,  elapsed time = 276 ms.

Clearly our read counts have shot up here, whilst we only read 6382 pages (Similar to our non compressed index) 22818 were pre-fetched in anticipation that we might need them as can be seen in the "lob read-ahead reads". So in the interest of just trying to reduce reads our columnstore was a failure however I should also add that this query ran in less than 300ms being more than twice as fast as our previous compressed covering index. The compression of a Columnstore index will vary massively depending on how much duplication you have in your data, the more duplication the more compression you will see.

