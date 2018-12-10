---
layout: post
title: SQL Server Agregation With No Data Loss
date: '2018-12-10 09:34:01'
---
Let's imagine we have the following table of posts for a generic forum...

{% highlight sql %}
CREATE TABLE posts
(
    Id INT IDENTITY PRIMARY KEY,
    Votes INT,
    Body NVARCHAR(MAX)
)
{% endhighlight %}

With that data we now want to to return the Votes, Body along with the sum of all votes and count of all posts for comparisson. This can't be done with a group by as in order to get the body and id we'd need to group by it and as soon as we do that any count or sum will only be on that group and not on the overall dataset.

One thing you can do in order to get aggregations and with no loss of data is use APPLY as I detailed in my APPLYBLOGPOST. For example...

{% highlight sql %}
SELECT TOP 100
   aggr.PostCount,
   aggr.VoteSum,
   Id,
   Votes,
   Body
FROM
   posts
   CROSS APPLY (SELECT COUNT(*) PostCount, SUM(Votes) VoteSum FROM posts) aggr
ORDER BY id
{% endhighlight %}

I've populated my posts table with 5 million records and the above query did the following...

TODO : Query Plan
TODO : Stats