---
layout: post
title: SQL Server PIVOT Explained
date: '2017-05-04 08:05:38'
---
The PIVOT/UNPIVOT features in SQL Server is I find quite underused. For a long time I got by with SELECT, JOIN, GROUP in SQL Server and missed out on some really handy features like PIVOT, CROSS APPLY, MERGE, XML and more recently JSON. Each of these features has a time and a place and if you don't know about them then you could be missing out on some really handy tools.

### What Does PIVOT Do? ###
Pivot essentially rotates a set so that unique data  in a given field become a field of their own.

Given the following dataset

| UnitsSold | Month |
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
	('Febuary','5'),
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

To do this our outter query needs to list all fields we want to select e.g January.. December. We then aply the PIVOT to the source query above telling it which field and values we want to PIVOT on...

{% highlight sql %}
SELECT [January],
    [Febuary],
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
        FOR [Month] IN ([January], [Febuary], [March], [April], [May], [June], [July], [August], [September],[October], [November], [December])
    ) AS PivotTable;
{% endhighlight %}

The result set will then look like this...

### What About UNPIVOT? ###
UNPIVOT rotates in the opposite direction of PIVOT so fields become data.

Lets take the reverse of before and have each month as a field. The following script will setup a test table with data

{% highlight sql %}
{% endhighlight %}


