---
layout: post
title: Why Wont SQL Server Use My Filtered Index?
date: '2018-10-02 20:48:01'
---
As with most of my posts of late all examples here are using the StackOverflow SQL Server database that can be downloaded from [Brent Ozar Unlimited](https://www.brentozar.com/archive/2015/10/how-to-download-the-stack-overflow-database-via-bittorrent/).

Filtered Indexes are exactly that, indexes that have a predicate causing them to only contain a specific part of the overall data. For example..

{% highlight sql %}
CREATE INDEX ndx_users_2018 ON Users(DisplayName)
WHERE CreationDate >= '20180101'
{% endhighlight %}

The above will create an index on display name for any user with a creation date on or after 2018-01-1. This can be useful when a large portion of your data is not needed to satisfy the query/queries we're building the index for. For example if we only run reports on this years data then there is no point in the supporting indexes for these reports covering other years (Unless those indexes have use elsewhere).

SQL Server can only use Filtered Indexes when the underlying query shares a predicate with the filtered index. For example given the above index this query will not be able to make use of it because it's looking at data from 2017 and the index only contains data from 2018 onwards...

{% highlight sql %}
SELECT DisplayName 
FROM [Users]
WHERE CreationDate >= '20170101'
{% endhighlight %}

![Execution Plan]({{site.url}}/content/images/2018-Filtered-Indexes/1.PNG)

However this query will be able to use it...

{% highlight sql %}
SELECT TOP 10 DisplayName 
FROM [Users]
WHERE CreationDate >= '20180101'
{% endhighlight %}

![Execution Plan]({{site.url}}/content/images/2018-Filtered-Indexes/1.PNG)

## Gotchas ##
If you use any kind of BETWEEN, Case Statement, SQL Function or UDF in your predicate then the filtered index will not be used, for example...

{% highlight sql %}
SELECT DisplayName
FROM [Users]
WHERE CreationDate BETWEEN '20180101' AND '20180601'

SELECT DisplayName 
FROM [Users] 
WHERE CreationDate > DATEADD(DAY,1,'20180101')

SELECT DisplayName
FROM [Users]
WHERE
  CASE WHEN LEN(DisplayName) < 10 THEN
    '20100101'
  ELSE '20110101'
  END <= CreationDate
{% endhighlight %}

All the above queries will not use the filtered index.

![Execution Plan]({{site.url}}/content/images/2018-Filtered-Indexes/3.PNG)

### Parameterized Queries ###
Possibly an even bigger gotcha than the above exceptions is parameterization, if your query uses parameters in it's predicate then it will not be a candidate for a filtered index, this is due to parameter sniffing and the fact that plans get reused with different values so SQL Server has no idea if a plan for one set of values will work for a different set as they may not match the filtered index.

There is a trick you can apply to get round this where needed, for example let's imagine the following stored procedure...

{% highlight sql %}
CREATE PROCEDURE GetUsersInYear
(
  @StartYear DATETIME
)
AS
SELECT 
  DisplayName
FROM
  [Users]
WHERE
  CreationDate >= @StartYear
{% endhighlight %}

If we then execute that in a year that matches our filtered index...

{% highlight sql %}
EXEC GetUsersInYear @StartYear = '20180101'
{% endhighlight %}

You'll see it wont use our index because is can't guarantee at the time it builds the plan that it will work for all possible values that the procedure could be called with.

We can get round this by removing the parameter from the underlying query with a bit of dynamic SQL hackery...

{% highlight sql %}
ALTER PROCEDURE GetUsersInYear
(
  @StartYear DATETIME
)
AS
DECLARE @Sql NVARCHAR(500) = '
SELECT 
  DisplayName
FROM
  [Users]
WHERE
  CreationDate >= ''' + CONVERT(VARCHAR(8), @StartYear, 112) + '''
'
EXEC sp_executesql @sql = @sql
{% endhighlight %}

If we then run this again...

{% highlight sql %}
EXEC GetUsersInYear @StartYear = '20180101'
{% endhighlight %}

We can now see it's correctly using our filtered index, by removing the parameter from the query SQL Server will not try to share plans across different parameters and will now generate a plan for each different value you pass in to the stored procedure.

![Execution Plan]({{site.url}}/content/images/2018-Filtered-Indexes/4.PNG)