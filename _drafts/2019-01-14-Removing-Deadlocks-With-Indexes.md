---
layout: post
title: Removing Deadlocks With Indexes
date: '2019-01-14 04:34:01'
---
A lot of people don't realise that some deadlocks can be removed entirely with the introduction of a new index. Some common solutions to deadlocks are

* Switch the order locks are taken out in your queries (If it's possible or makes sense)
* Reduce the chance of the offending queries running at the same time
* A error handling to the app to auto retry when deadlocks do occur

All of the above are good practices and should be done but I also wanted to cover here how some deadlocks can be solved with the introduction of an index. The ways in which an index can help are

* Speed up the query causing locks to be held for less time and reduce chance of deadlock
* Covering indexes remove the need for the underlying clustered index to be touched, if you're deadlock is being caused by shared locks from selected on a small number of fields you may be able to move these locks to a new inde whereby the two offending queries don't need locks on the same data.
* They can lower lock escalation, for example a good index may lead to a better plan that locks at row level rather than page level again reducing the chance for conflicting locks on the same data.

Let's look at a demo of how a covering index can help, if you want to follow along you'll need a copy of the [https://www.brentozar.com/archive/2015/10/how-to-download-the-stack-overflow-database-via-bittorrent/](Stack Overflow Database).

Two simulate the deadlock run the following queries in two seperate tables in sequence

| Tab 1 | Tab 2 |
| --- | --- |
| BEGIN TRAN | |
| --EXCLUSIVE LOCK ON VOTES | |
| UPDATE [Votes] SET BountyAmount = 10 WHERE id = 1 | |
| | BEGIN TRAN |
| | --ATTEMPT EXCLUSIVE LOCK ON USERS |
| |UPDATE [Users] SET Age = 10 WHERE Id = 1 |
| --TAKE A BUNCH OF SHARED LOCKS ON USER | |
 SELECT TOP 100 
 	Location, 
 	COUNT(*) 
FROM | |
   [Users] | |
GROUP BY [Location] 
ORDER BY COUNT(*) DESC 