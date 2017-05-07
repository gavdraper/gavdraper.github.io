---
layout: post
title: Using SQL Server Merge Statement
date: '2017-05-08 13:05:38'
---
The SQL Server merge statement kind of does what it says, given some source data and a destination table it can UPDATE data that already exists but has changed, INSERT data that is not in the destination, remove data from the destination that is not in the source.

I've mainly used this for syncing/importing  data across multiple databases.

Take this table as our Source

| Id | Username | Location | FavoriteColour |
| --- | --- | --- | --- |