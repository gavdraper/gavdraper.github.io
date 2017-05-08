---
layout: post
title: SQL ServerClustered vs NonClustered Indexes
date: '2017-05-09 06:05:38'
---
When using row level indexes there are two types Clustered and Non Clustered. To understand the difference we need to look at what an index is...

### NonClustered Indexes ###
Non clustered indexes work much like an index in a book, The index is stored seperate to the actual data and contains a pointer bak to the data (Just like a page number)

For example 

CREATE NONCLUSTERED INDEX ON Users(Username)

Will create an index like this

| IndexFields | Pointer |
| AliceJones | 
| CaseyFlow
| GavinDraper
| Prince201
| TrevorTent

The pros of NonClustered indexes are you can have multiple of them and writes are faster than a clustered index as you are typically storing/reorganizing less data than a clustered index.

### NonClustered Includes ###
So one of the cons of a NonClustered index is you have to do a lookup back to the data once you've found your item in the index (Assuming the index doesn't contain all the data required by your query). With SQL Server you can add Include Fields to your NonClustered indexes this is extra fields you can store in your index but it wont be sorted, this allows you to make your index cover more queries without having to do lookups for data that doesnt need to be sorted.

Lets say we have the following query...

SELECT Id,Username, PlaceOfBirth FROM [dbo].[user] WHERE [Username] = 'Gavin'

We can create a NonClustered index on Username as we want to be able to quickly search on that field

CREATE NONCLUSTERED INDEX ON [dbo].[user](UserName)

Lets pretend the user table is a really really big table and this query is run a LOT we can stop the new index having to do lookups back to the row by INCLUDING id and PlaceOfBirth in the index, this then makes it a covering index for our query and the lookup will be gone.

CREATE NONCLUSTERD INDEX ON [dbo].[iser](Username) INCLUDE(id,PlaceOfBirth)

If we had of added those fields to the index without making them include fields then the index would have been sorted by all 3 fields (2 of which we don't need to search on) which would have slowed down index updates.


### Clustered Indexes ###

A clustered index doesnt actually store an index seperate to the data instead it defines the order to store the actual data in. The pros of this are once you find your data in the index you are at the data and don't need to do a pointer lookup from the index to the row. The downside is that you can only have one clustered index per table as the tables data is only stored one.

