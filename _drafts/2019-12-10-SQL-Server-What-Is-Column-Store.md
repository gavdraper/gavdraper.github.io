---
layout: post
title: Indexing In Memory OLTP Tables
date: '2018-12-10 09:34:01'
---
Indexing on In Memory OLTP tables is a little different from your traditional on disk rowstore tables...

In Memory Differences...
- There is no clustered index
- There is a new hash index ideal for single record lookups
- The non clustered index still exists but it;s structure is quite different.

Lets got over each of these points

## No Clustered Index ##

## New Hash Index ##

## Non Clustered Index ##