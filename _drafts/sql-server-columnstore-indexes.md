---
layout: post
title: Supercharge Your Database With Columnstore Indexes
date: '2017-07-07 19:05:38'
---
Columnstore indexes are often either overlooked or missed completely, As of SQL Server 2016 they are pretty well featured. In versions leading up to 2016 there were restrictions with updating them and non clustered only, Now these restrictions are gone a LOT more use cases have opened up for them!

Before we go in to detail on what they are and how they work let's look at a simple example of them in action! 