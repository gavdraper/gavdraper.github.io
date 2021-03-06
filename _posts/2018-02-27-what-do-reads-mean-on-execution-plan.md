---
layout: post
title: What Do Read Counts On Your SQL Server Execution Plans Even Mean?
date: '2018-02-27 06:05:38'
tags: profiling performance-tuning
---

If you've ever spent any time looking at execution plans you will have almost definitely noticed all the read counts on the different operators, you've probably already realised lower reads is best as that equates to less time reading disks. Have you ever wondered what the number actually means though? What does 10 reads mean? Time? Operations? Volume?

Let's step back and look at how SQL Server stores data.

The smallest unit SQL Server stores/accesses data at is called a page, these are 8kb blocks and are stored inside 64kb extents. If you have a row that takes 1kb and you run a query to get one row it will actually bring the whole page that row is in to memory before sending the single row on to the client, this is what counts as a single read.

Let's look at some examples...

{% highlight sql %}
DROP TABLE PageTest
GO
CREATE TABLE PageTest
(
   OnlyColumn CHAR(3000),
)

INSERT INTO PageTest(OnlyColumn) VALUES('test')
{% endhighlight %}

So we know a page is 8kb, inside the page is a 96 byte header, a 2 byte list of row offsets and the actual row data. Based on that given the example above we can see that a 2 rows will fit in a single page as it's a single CHAR(3000) field.

If you turn on statistics IO and run a select from this table how many reads should that show?...

{% highlight sql %}
SET STATISTICS IO ON
SELECT OnlyColumn FROM PageTest
{% endhighlight %}

> Table 'PageTest'. Scan count 1, logical reads 1, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

1 Logic read, that's one record and one page. A logical read means the page is already in memory and it's being read from there, physical means it had to first be read from the disk. As we can never guarantee what will or will not be in memory never assume logical wont become physical. Let's insert one more record and run that select again...

{% highlight sql %}
INSERT INTO PageTest(OnlyColumn) VALUES('test')
SET STATISTICS IO ON
SELECT OnlyColumn FROM PageTest
{% endhighlight %}

> Table 'PageTest'. Scan count 1, logical reads 1, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

Still 1 read. Let's try that one more time with 3 records...

{% highlight sql %}
INSERT INTO PageTest(OnlyColumn) VALUES('test')
SET STATISTICS IO ON
SELECT OnlyColumn FROM PageTest
{% endhighlight %}

> Table 'PageTest'. Scan count 1, logical reads 2, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.

Now we're up to 2 reads, As you can probably see in this simple example we have 3 records of CHAR(3000) totalling at 9000 which pushes the 3rd record onto a new page. It's worth noting at this point that this works exactly the same for indexes only in that case the index page contains only the fields within the index. 

This example is pretty trivial but it does illustrate that a field that is using a type larger than the underlying type needs is going to cause a lot of extra IO. If we'd made that field a CHAR(10) or even a VARCHAR(10) we could get close to 800 records on a single page.