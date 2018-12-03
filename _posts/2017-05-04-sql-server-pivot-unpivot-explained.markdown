---
layout: post
title: SQL Server Pivot and Unpivot Explained
date: '2017-05-04 08:05:38'
tags: tsql pivot
---
The Pivot and Unpivot features in SQL Server are  I find quite underused. For a long time I got by with SELECT, JOIN, GROUP in SQL Server and missed out on some really handy features like PIVOT, CROSS APPLY, MERGE, XML and more recently JSON. Each of these features is another tool in your toolkit that has a time and a place and if you don't know about them then you could be missing out.

### What Does Pivot Do? ###
Pivot essentially rotates a set so that unique data  in a given field become a field of their own.

Given the following dataset

| Count | Month |
| --- | --- |
| 13 | Januray |
| 5 | February |
| 11 | March | 

...

We can use a pivot to make it look like this...

| January | February | March |
| --- | --- | --- |
| 13 | 5 | 11 | 

The following script will create a demo table with data to try out an example on...

{% highlight sql %}
CREATE TABLE Sales
(
	Id INT IDENTITY PRIMARY KEY,
	[Month] NVARCHAR(20),
	[Count] INT
)

INSERT INTO Sales([Month],[Count])
VALUES
	('January','10'), 
	('February','5'),
	('March','10'), 
	('April','10'), 
	('May','10'), 
	('June','10'), 
	('July','10'), 
	('August','3'), 
	('September','10'), 
	('October','10'), 
	('November','100'), 
	('December','103')
{% endhighlight %}

So in this case we want to PIVOT on this 

{% highlight sql %}
SELECT [Month], [COUNT] FROM Sales
{% endhighlight %}

To do this our outer query needs to list all fields we want to select e.g January.. December. We then apply the PIVOT to the source query above telling it which field and values we want to PIVOT on...

{% highlight sql %}
SELECT [January],
    [February],
    [March],
    [April],
    [May],
    [June],
    [July],
    [August],
    [September],
    [October],
    [November],
    [December]
FROM
    (SELECT [Month], [Count] FROM Sales) AS Source
    PIVOT
    (
        SUM([Count])
        FOR [Month] IN 
            (
            [January], [February], [March], [April], [May], [June], [July], 
            [August], [September],[October], [November], [December]
            )
    ) AS PivotTable;
{% endhighlight %}

The result set will then look like this...

| January | February | March |
| --- | --- | --- |
| 10 | 5 | 10 | 

### What About Unpivot? ###
Unpivot rotates in the opposite direction of Pivot so fields become data.

Lets take the reverse of before and have each month as a field. The following script will setup a test table with data

{% highlight sql %}
CREATE TABLE PivotedSales
(
    Id INT IDENTITY PRIMARY KEY,
    January INT,
    February INT,
    March INT,
    April INT,
    May INT,
    June INT,
    July INT,
    August INT,
    September INT,
    October INT,
    November INT,
    December INT
)
INSERT INTO PivotedSales(January,February,March, April,
     May, June, July, August, September, October, November,
     December)
VALUES(1,2,3,4,5,6,7,8,9,10,11,12)
{% endhighlight %}

Much like the Pivot example we create our source query which in this case is...

{% highlight sql %}
SELECT 
    January,February,March, April, May, June, July, 
    August, September, October, November, December 
FROM 
    PivotedSales
{% endhighlight %}

We then wrap the source query in an outer query that specifies the fields we want and add the UNPIVOT to it...

{% highlight sql %}
SELECT 
    [Month], [Sales]
FROM 
    (
    SELECT 
        January,February,March, April, May, June, July, 
        August, September, October, November, December 
    FROM 
        PivotedSales
    ) AS Source
    UNPIVOT
    (
        Sales FOR [Month] IN 
            (
            January,February,March, April, May, June, July, 
            August, September, October, November, December
            )
    ) AS unpivoted
{% endhighlight %}

This gives us a result set the looks similar to our pre Pivot example.

| Sales | Month |
| --- | --- |
| 1 | January |
| 2 | February |
| 3 | March | 

Hopefully this post gives a good overview of what these two statements do and how to use them. 