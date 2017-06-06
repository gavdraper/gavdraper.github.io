---
layout: post
title: SQL Server 2016 What's New With TSQL
date: '2017-06-03 16:47:47'
---

## DROP IF EXISTS ##
This allows us to drop objects without first having to check if they exist. It works on pretty much every object you could want to drop and changes this...

{% highlight sql %}
IF OBJECT_ID (N'MyDatabase.MySchema.MyTable', N'U') IS NOT NULL 
    DROP TABLE MyTable
{% endhighlight %}

To this...

{% highlight sql %}
DROP TABLE IF EXISTS MyDatabase.MySchema.MyTable
{% endhighlight %}

## STRING_SPLIT  ##
Allows a string to be split on a character. 

{% highlight sql %}
SELECT value FROM STRING_SPLIT('This,Is,A,Test',',')
{% endhighlight %}

Will produce

| --- |
| This |
| Is |
| A |
| Test |

## STRING_ESCAPE ##
This takes an input string and a type and will escape the characters depending on the type specified. In this release the only supported type is JSON.

{% highlight sql %}
SELECT STRING_ESCAPE('this
is
a
test','json')
{% endhighlight %}

Will produce

> this\r\nis\r\na\r\ntest

## JSON Functions ##
A number of functions for working JSON have been introducded including...

* JSON_VALUE
* JSON_QUERY
* OPENJSON
* FOR JSON PATH
* JSON_MODIFY

I have covered these in more detail in my post [Using JSON In SQL Server 2016](https://gavindraper.com/2017/05/06/sql-server-json/)

## Temporal Table Support ##
Temporal tables allow us to query a table and get back the data as it was at a given point in table. SQL Server will automatically manage the history of a table that we declare as Temporal. We can then work with this data using the new FOR_SYSTEM_TIME syntax.

{% highlight sql %}
SELECT
    *
FROM
    dbo.User FOR SYSTEM_TIME BETWEEN '20170101' AND '20170201'
{% endhighlight %}

For more information see [SQL 2016 Temporal Tables By Example](https://gavindraper.com/2016/04/15/sql-2016-temporal-tables-by-example-2/)

## TRANCATE TABLE WITH PARTITION ##
The TRUNCATE TABLE statement has been improved to allow you to truncate by partition on partitioned tables. If no partitions are specified it will truncate all of them. To specify partitions you can list them one by one or pass in a range or a mixture of both...

{% highlight sql %}
TRUNCATE TABLE dbo.User WITH (PARTITIONS(1,2,3,4 TO 6))
{% endhighlight %}

## FORMATMESSAGE ##
FORMATMESSAGE can now take a string as input rather than just working with existing messages in sys.messages. Previously you had to pass an Id in from sys.messages like this...

{% highlight sql %}
SELECT FORMATMESSAGE(20001, 'Gavin', 'Draper')
{% endhighlight %}

With the new changes we can specify the message string when we call FORMATMESSAGE

{% highlight sql %}
SELECT FORMATMESSAGE('Hello my name is %s %s', 'Gavin', 'Draper')
{% endhighlight %}
