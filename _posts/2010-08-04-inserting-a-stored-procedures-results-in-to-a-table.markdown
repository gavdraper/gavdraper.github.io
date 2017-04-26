---
layout: post
title: Inserting A Stored Procedures Results In To A Table
date: '2010-08-04 19:05:38'
---

<p>
Just a quick post to cover somthing that I've seen cropping up a lost recently. Thats
copying the results of a stored procedure in to a table. 
</p>
<p>
The code below will call the stored procedure spGetRecords and copy its results into
the specified table. 
<br />
<br />
<pre class='brush: sql;toolbar:false'>CREATE TABLE #tmpBus
(
   COL1 INT,
   COL2 INT   
)

INSERT INTO #tmpBus 
Exec SpGetRecords 'Params'
</pre>
