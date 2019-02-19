---
layout: post
title: Wonky LOB/Overflow Read Counts
date: '2019-02-19 09:07:42'
---
I've been playing around with querying the Allocation Unit system views and comparing read page counts in my queries against what's there, in doing this I noticed a couple of things that seem a little odd...

* If I turn on Statistics IO and query a table with no LOB allocations but some Row-Overflow allocations I see LOB logical reads occurring (This may just be how overflow reads are reported, but it seems misleading)
* If I run that same query and look at the actual execution plan under Actual I/O Statistics no LOB reads are reported
* If I create a new table with a column that will be put in LOB storage the query plan still reports zero LOB logical reads.

Let's look at these examples...

## Setup ##
First lets create a sandbox database...
{% highlight sql %}
CREATE DATABASE AllocationUnitSandbox
GO
USE AllocationUnitSandbox
GO
{% endhighlight %}

Then lets add a little stored procedure we can run throughout our examples to see the allocation units...

{% highlight sql %}
CREATE PROCEDURE GetAllocationUnits AS
SELECT  
   o.name ObjectName,
   i.name IndexName,
   i.type_desc IndexType,
   au.type_desc AllocationUnitDesc,
   au.total_pages AllocationUnitTotalPages,
   au.used_pages AllocationUnitUsedPages,
   fg.name AS FileGroupName
FROM 
   sys.allocation_units au
   LEFT JOIN sys.filegroups fg ON fg.data_space_id = au.data_space_id
   LEFT JOIN sys.partitions p ON
      (au.type_desc IN ('IN_ROW_DATA','ROW_OVERFLOW_DATA') AND p.hobt_id = au.container_id)
      OR (au.type_desc IN ('LOB_DATA') AND p.partition_id = au.container_id)
   LEFT JOIN sys.objects o ON o.object_id = p.object_id
   LEFT JOIN sys.indexes i ON 
      i.object_id = o.object_id
      AND i.index_id = ISNULL(p.index_id,0)
WHERE
   o.[type] NOT IN ('S','IT') /*SystemTable, InternalTable*/
{% endhighlight %}

## In-Row Demo ##

Before we look at any Row-Overflow or LOB allocation units lets take this simple example where all the data fits In-Row.

{% highlight sql %}
CREATE TABLE InRow(Field1 VARCHAR(100), Field2 VARCHAR(100))
INSERT INTO InRow(Field1, Field2)
VALUES(REPLICATE('a',100), REPLACE('b',100))
{% endhighlight %}

Then lets check out GetAllocationUnits procedure...

{% highlight sql %}
EXEC GetAllocationUnits
{% endhighlight %}

![In Row]({{site.url}}/content/images/2019-LOB-Reads/in-row.JPG)

We can see we have a single allocation unit for In-Row pages on this table. If we now run a SELECT * with statistics IO turned on we should see a single logical read...

{% highlight sql %}
SET STATISTICS IO ON
SELECT * FROM InRow
{% endhighlight %}

![In Row Single Read]({{site.url}}/content/images/2019-LOB-Reads/single-logical.JPG)

Bingo.

## Row-Overflow Demo ##

Let's now create a table with no fields that qualify for LOB but enough variable length for us to cause overflows...

{% highlight sql %}
CREATE TABLE Overflow
(
    Field1 VARCHAR(100),
    Field2 VARCHAR(8000)
)
INSERT INTO Overflow(Field1,Field2)
VALUES(REPLICATE('a',100),REPLICATE('b',8000))
{% endhighlight %}

If we then run our GetAllocationUnits procedure we should see that we have some data in row and some in overflow due to the fact we can't fit our 100 length field1 and our 8000 length field2 on a single page...

{% highlight sql %}
EXEC GetAllocationUnits
{% endhighlight %}

![Overflow Allocation Unit]({{site.url}}/content/images/2019-LOB-Reads/overflow-allocation.JPG)

If we then run a select * we'd expect to have to read at least 2 pages, one from the In-Row allocation unit and one from our Overflow-Row allocation unit...

{% highlight sql %}
SET STATISTICS IO ON
SELECT * FROM Overflow
{% endhighlight %}

![Overflow LOB Read]({{site.url}}/content/images/2019-LOB-Reads/lob-read.JPG)

I'm still not sure about this reporting as a LOB read when it's really Row-Overflow but I guess it is what it is. What I really find odd is that if we run the above query again but turn on actual query plans we see no LOB reads...

![No LOB Execution Plan]({{site.url}}/content/images/2019-LOB-Reads/no-lob-execution-plan.JPG)

## LOB Data Demo ##

Where this gets even weirder is if we then create another table that has real LOB data...

{% highlight sql %}
CREATE TABLE LOB
(
    Field1 VARCHAR(100),
    Field2 VARCHAR(MAX)
)
INSERT INTO LOB(Field1,Field2)
VALUES(REPLICATE('a',100), REPLICATE('b',10000))

EXEC GetAllocationUnits
{% endhighlight %}

![LOB Allocation Units]({{site.url}}/content/images/2019-LOB-Reads/lob-allocations.JPG)

Now let's do select * again...

{% highlight sql %}
SET STATISTICS IO ON
SELECT * FROM LOB
{% endhighlight %}

![LOB Logic Reads]({{site.url}}/content/images/2019-LOB-Reads/lob-logical-2.JPG)

As expected we can see our LOB reads however if we switch back to our actual execution plan...

![LOB Not On Execution Plan]({{site.url}}/content/images/2019-LOB-Reads/lob-execution-plan.JPG)

Still now LOB reads showing under Actual I/O Statistics.

I've even tried running the insert statement multiple times to increase the page count, STATISTICS IO ON correctly reports the read pages but my actual execution plan stays the same with zero LOB pages read. Weird Right? 
