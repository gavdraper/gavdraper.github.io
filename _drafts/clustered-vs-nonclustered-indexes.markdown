---
layout: post
title: SQL ServerClustered vs NonClustered Indexes
date: '2017-05-09 06:05:38'
---
When using row level indexes there are two types Clustered and Non Clustered both of which are there to make data easy to find and sort. Before we look at these index types lets go over a couple of things

* Both these indexes are stored as [B-Trees](https://en.wikipedia.org/wiki/B-tree)
* Root node is the entry point to the index and each index contains exactly one.
* Leaf Nodes are the destination of the index, exactly what is stored here depends on if it's a clustered or non clustered index. To get to a leaf node SQL Server starts at the root node and uses the B-Tree structure to find the relevant Lead Nodes.
* Heap is a table without a clustered index

 To understand the difference we need to look at how each of these work...

## NonClustered Indexes ##
Non clustered indexes work much like an index in a book, The index is stored separate to the actual rows  and contains a pointer back to the data (Just like a page number). The leaf node in a non clustered index contains the fields in the index, any included fields in the index and a the key for either the clustered index on the table if one exists or a RowId key if the table is a heap.

For example 

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndxUsersUsername ON Users(Username)
{% endhighlight %}

Will create an index like this

| IndexFields | RowID |
| --- | --- |
| AliceJones | 3114 |
| CaseyFlow | 56 |
| GavinDraper | 45 |
| Prince201 | 76 |
| TrevorTent | 67 |

If we wrote a query that was looking for a username of GavinDraper SQL Server would potentially use the Username index above to find quick find GavinDraper, if your query selected fields that are not in the index it would then using the pointer in the index go to the actual row and return you the requested data.

### Pros ###
1. You can have an unlimited amount of Non Clustered indexes on any give table.
2. Inserting and Updating data in non clustered indexes is generally faster.

### Cons ###
1. Takes up more space as the index data is stored twice, once in the index and once in the actual table.
1. Have a performance overhead as from the leaf node you need to do a lookup back to the actual table to get any data that is not in the index.

#### Key Lookups ####
One of the cons of a NonClustered index is you have to do a lookup back to the data once you've found your item in the leaf node of the index (Assuming the index doesn't contain all the data required by your query). 

Lets take a look at what this mean and how it can affect performance...

If you want to follow along this query will setup a table with some seed data.

{% highlight sql %}
DROP TABLE [User]
CREATE TABLE [User]
(
	Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
	Username NVARCHAR(100),
	Firstname NVARCHAR(100),
	LastName NVARCHAR(100),
	PlaceOfBirth NVARCHAR(100)
)

INSERT INTO [dbo].[User]
    ( 	
	Username,
	FirstName,
	LastName
	)
VALUES
	('clairetemple','Claire','Temple'),
	('lukecage','Luke','Cage'),
	('jessiejones','Jessie','Jones'),
	('tonystark','Tony','Stark'),
	('mattmurdock','Matt','Murdock')

DECLARE @LoopCount INT = 1
WHILE @LoopCount < 18
	BEGIN
	INSERT INTO [dbo].[User](Username,FirstName,LastName)
	SELECT 
		CAST(@LoopCount AS NVARCHAR(4)) + Username,
		CAST(@LoopCount AS NVARCHAR(4)) + FirstName,
		CAST(@LoopCount AS NVARCHAR(4)) + LastName 
	FROM [dbo].[User]
	SET @LoopCount = @LoopCount +1
	END

{% endhighlight %}

This will create us a User table with 655,000 records in it.

Given a username we want to get their first name

{% highlight sql %}
SELECT Firstname FROM [User] WHERE [Username] = 'lukecage'
{% endhighlight %}

If we look at the query plan for this we can see it's scanning all the rows in the table to look for one with a username of lukecage.

![Table Scan Query Plan]({{site.url}}/content/images/2017-indexes-explained/tablescan.jpg)

For our small table this is fine but imagine a larger table with millions of records where we are interested in more than a single one, in that case it's not optimal to have to look at every record to check if the username matches your predicate. 

To solve this lets  create a Non Clustered index on Username so we can search on it without having to look at every record.

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_user_username ON [dbo].[user](UserName)
{% endhighlight %}

If we now look at the query plan we can see it's using our new index and once it's found the record it's doing a key lookup back to the heap to get the firstname.

![Table Scan Query Plan]({{site.url}}/content/images/2017-indexes-explained/nonclusteredkeylookup.jpg)

In this case the key lookup is very quick as we're only selecting one row. If however we were selecting a lot of rows the key lookup can start to become slow as it's having to constantly jump from the index to the heap. 

Let's imagine we have a slow query, we've checked the execution plan and profiler it and it looks like the key lookup is what's causing the bad performance. What are our options? 

* We could add the field to the end of our index 

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_user_username ON [dbo].[user](UserName, Firstname)
{% endhighlight %}

If we do this then we can see from our plan the key lookup has gone

![Table Scan Query Plan]({{site.url}}/content/images/2017-indexes-explained/nonclusterednokeylookup.jpg)

The problem with this approach is we're not actually wanting to search on firstname and we've added the overhead when maintaining the index that it has to be stored in Username, Firstname order. This is going to slow down inserts and updates on the firstname field when it doesn't really need to.

*  We could use an include field. This is a field that gets stored in the leaf node of the index and doesn't affect the order of the index, perfect when it's not a field we're filtering or sorting on.

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndx_user_username ON [dbo].[user](UserName) INCLUDE (firstname)
{% endhighlight %}

You'll notice the query plan is now the same as above but with the benefit of not having the overhead of maintaining firstname sorting in the index.

Both these techniques are creating what's called a covering index and that is an index that contains all the information required by your query without the need togo back to the actual table/heap.

### Clustered Indexes ###

A clustered index doesn't actually store an index separate to the data instead it defines the order to store the actual data in. The pros of this are once you find your data in the index you are at the data and don't need to do a pointer lookup from the index to the row. The downside is that you can only have one clustered index per table as the tables data is only stored one. So if we take the following table..

| Id | Username | Forename | Location |
| 1 | GavinDraper | Gavin | Brighton |
| 2 | AmyPlumb | Amy | Worthing |
| 3 | Jake200 | Jake | London |

Then add a  clustered index username then the index/table will look like this...

| Id | Username | Forename | Location |
| 2 | AmyPlumb | Amy | Worthing |
| 1 | GavinDraper | Gavin | Brighton |
| 3 | Jake200 | Jake | London |

This means any new records inserted into the table need to be stored in order of the index, this can cause a lot of fragmentation for non sequential indexes. If the priority is for high speed inserts it may be better to use a sequential clustered index so data can be quickly appended.

### Pros ###
* There is no key lookup from the leaf of the index back to the table data as the data is stored there.
* Takes less disk space than a non clustered index as you're not storing the index data twice.

### Cons ###
* You're limited to one per table
* Non sequential clustered indexes an cause a lot of fragmentation when inserting and updating data.

Using the examples from above let's drop the Non Clustered index and create a Clustered index on Username




