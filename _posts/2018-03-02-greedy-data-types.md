---
layout: post
title: SQL Server and Evil Greedy Data Types
date: '2018-03-02 07:15:23'
---
In a [recent post](https://gavindraper.com/2018/03/02/greedy-data-types), I talked about SQL Server reads and how the data is stored in pages. One of the points I made in that post was that you should always use the smallest type that makes sense for a given set of data in order to minimize the IO required. I once worked on a system that constantly used data types that were greedier than needed (e.g INT instead of BIT, CHAR(200) where CHAR(80) or even better VARCHAR(80) would have sufficed) and had all sorts of issues with performance. One of the things we did to improve this was to go through the application/database and set everything to use the correct types. One of the things we noticed with this process was that initially page reads went up not down after correcting the data types, let's look at why that was and how to fix it...

Run this to create a demo environment....

{% highlight sql %}
CREATE DATABASE DataTypeCleanup
GO
USE DataTypeCleanup
GO

CREATE TABLE Users
(
	Id INT IDENTITY,
	Username CHAR(100) NOT NULL,
	IsAdmin INT,
	IsReporter INT,
	IsManager INT 
)
ALTER TABLE Users ADD  CONSTRAINT [pk_users] PRIMARY KEY CLUSTERED(Id ASC)
{% endhighlight %}

Hopefully you can already spot some potential issues with this...

* Username wont always be the same length so VARCHAR is a better fit than CHAR, also in this case I know we never use usernames longer than 50 characters.
* The 3 Boolean fields have all been given the type INT when they will only ever contain a 1 or 0, this should have used the BIT data type.

Before we correct these issues let's insert some dummy data....

{% highlight sql %}
INSERT INTO [Users](Username,IsAdmin,IsReporter,IsManager)
SELECT 
   s1.[Name],
   s1.number % 2,
   s2.number % 2,
   s1.number % 2
FROM 
   master.dbo.spt_values s1
   CROSS JOIN master.dbo.spt_values s2
WHERE 
   s2.number < 100
   AND s1.Name IS NOT NULL
{% endhighlight %}

Lets then turn on STATISTICS IO and run a query selecting all the rows and see how many reads that causes...

{% highlight sql %}
SET STATISTICS IO ON
SELECT Id,Username,IsAdmin,IsReporter,IsManager FROM Users
{% endhighlight %}

> (140220 rows affected)
Table 'Users'. Scan count 1, logical reads 2200, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.


So reading 140k records aused 2200 page reads.

Let's now alter our column types the be more suitable for this data...

{% highlight sql %}
ALTER TABLE Users ALTER COLUMN Username VARCHAR(50) NOT NULL
ALTER TABLE Users ALTER COLUMN IsAdmin BIT 
ALTER TABLE Users ALTER COLUMN IsReporter BIT 
ALTER TABLE Users ALTER COLUMN IsManager BIT 
{% endhighlight %}

Now let's run our select again and see if the read counts have dropped...

{% highlight sql %}
SET STATISTICS IO ON
SELECT Id,Username,IsAdmin,IsReporter,IsManager FROM Users
{% endhighlight %}

> (140220 rows affected)
Table 'Users'. Scan count 1, logical reads 4393, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

They almost doubled!! How can this be if we've shrunk the data types? This is because SQL Server wont reclaim the space used in every row for the old column type and is in fact storing our new BIT/VARCHAR data types along side the space already allocated for the old data types. The only way to then reclaim this space is to REBUILD the table if it's a heap or rebuild the clustered index if it is not...

{% highlight sql %}
ALTER INDEX pk_users ON Users REBUILD
{% endhighlight %}

If we then try our select again...

{% highlight sql %}
SET STATISTICS IO ON
SELECT Id,Username,IsAdmin,IsReporter,IsManager FROM Users
{% endhighlight %}

> (140220 rows affected)
Table 'Users'. Scan count 1, logical reads 1191, physical reads 0, read-ahead reads 21, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

In our simple example we now have over 1000 less reads just by changing the types of 3 columns. On the system I was working on at the time IO dropped nearly 50% after correcting greedy data types as our largest most frequently run queries were constantly hitting these fields. One point to note is that unless you are using Enterprise edition REBUILDS must be done offline so make sure this is done in scheduled downtime if you're doing this.
