---
layout: post
title: SQL ServerClustered vs NonClustered Indexes
date: '2017-05-09 06:05:38'
---
When using row level indexes there are two types Clustered and Non Clustered both of which are there to make data easy to find and sort. To understand the difference we need to look at how each of these work...

## NonClustered Indexes ##
Non clustered indexes work much like an index in a book, The index is stored separate to the actual rows  and contains a pointer back to the data (Just like a page number)

For example 

{% highlight sql %}
CREATE NONCLUSTERED INDEX ndxUsersUsername ON Users(Username)
{% endhighlight %}

Will create an index like this

| IndexFields | Pointer |
| --- | --- |
| AliceJones | 3114 |
| CaseyFlow | 56 |
| GavinDraper | 45 |
| Prince201 | 76 |
| TrevorTent | 67 |

If we wrote a query that was looking for a username of GavinDraper SQL Server would potentially use the Username index above to find quick find GavinDraper, if your query selected fields that are not in the index it would then using the pointer in the index go to the actual row and return you the requested data.

### Pros ###
1. You can have an unlimited amount of Non Clustered indexes on any give table.

### Cons ###
1. Creating too many NonClustered indexes can depending on your system impact write performance on the indexed tables as the index is maintained in realtime.

### NonClustered Includes ###
So one of the cons of a NonClustered index is you have to do a lookup back to the data once you've found your item in the index (Assuming the index doesn't contain all the data required by your query). With SQL Server you can add Include Fields to your NonClustered indexes this is extra fields you can store in your index but it wont be sorted, this allows you to make your index cover more queries without having to do lookups for data that doesn't need to be sorted.

Lets say we have the following query...

{% highlight sql %}
SELECT Id,Username, PlaceOfBirth FROM [dbo].[user] WHERE [Username] = 'Gavin'
{% endhighlight %}

We can create a NonClustered index on Username as we want to be able to quickly search on that field

{% highlight sql %}
CREATE NONCLUSTERED INDEX ON [dbo].[user](UserName)
{% endhighlight %}

Lets pretend the user table is a really really big table and this query is run a LOT we can stop the new index having to do lookups back to the row by INCLUDING id and PlaceOfBirth in the index, this then makes it a covering index for our query and the lookup will be gone.

{% highlight sql %}
CREATE NONCLUSTERD INDEX ON [dbo].[User](Username) INCLUDE(id,PlaceOfBirth)
{% endhighlight %}

If we had of added those fields to the index without making them include fields then the index would have been sorted by all 3 fields (2 of which we don't need to search on) which would have slowed down index updates.

### Clustered Indexes ###

A clustered index doesn't actually store an index separate to the data instead it defines the order to store the actual data in. The pros of this are once you find your data in the index you are at the data and don't need to do a pointer lookup from the index to the row. The downside is that you can only have one clustered index per table as the tables data is only stored one.

