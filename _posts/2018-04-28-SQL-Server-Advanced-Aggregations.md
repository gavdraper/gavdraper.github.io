---
layout: post
title: SQL Server Advanced Aggregations
date: '2018-04-28 19:47:24'
---
Following on from my previous post on [GROUPING SETS](https://gavindraper.com/2018/04/26/SQL-Server-Grouping-Sets-Explained/) I thought I'd cover a couple of other more advanced aggregation options that are often missed, namely ROLLUP and CUBE.

Lets take another look at the GROUPING SETS code I showed in my previous post...

{% highlight sql %}
CREATE TABLE Sales
(
   Id INT IDENTITY PRIMARY KEY,
   ProductName NVARCHAR(100),
   SaleDate DATE,
)

INSERT INTO Sales VALUES('Bike','20170101')
INSERT INTO Sales VALUES('TV','20180102')
INSERT INTO Sales VALUES('Skateboard','20180215')
INSERT INTO Sales VALUES('Stereo','20180216')
{% endhighlight %}

{% highlight sql %}
SELECT
   MONTH(SaleDate) Month,
   YEAR(SaleDate) Year, 
   COUNT(*) AS Sales
FROM 
   Sales
GROUP BY GROUPING SETS
(
   (MONTH(SaleDate)),
   (YEAR(SaleDate))
)
{% endhighlight %}

In the above example we defined two separate groups to aggregate on. Now lets imagine our dataset has several other dates we also want to aggregate on. To do this with GROUPING SETS we'd have to define a group for each aggregation level. Lets create a new sample table to test this...

{% highlight sql %}
CREATE TABLE SalesReporting
(
   Id INT IDENTITY PRIMARY KEY,
   ProductName NVARCHAR(100),
   SaleDate DATE,
   ShipDate DATE,
   DeliveryDate DATE
)

INSERT INTO SalesReporting VALUES('Bike','20170101','20170102','20170103')
INSERT INTO SalesReporting VALUES('TV','20170101','20170201','20170302')
INSERT INTO SalesReporting VALUES('Skateboard','20180215','20180215','20180216')
INSERT INTO SalesReporting VALUES('Stereo','20180216','20180221','20180222')
{% endhighlight %}

Imagine we have some reporting requirements to show aggregations in Sale, Ship and Delivery dates by month and year separately. Using GROUPING SETS that would look something like this....

{% highlight sql %}
SELECT
   MONTH(SaleDate) SalesMonth,
   YEAR(SaleDate) SalesYear, 
   MONTH(ShipDate) ShipMonth,
   YEAR(ShipDate) ShipYear, 
   MONTH(DeliveryDate) DeliveryMonth,
   YEAR(DeliveryDate) DeliveryYear,    
   COUNT(*) AS Amount
FROM 
   SalesReporting
GROUP BY GROUPING SETS
(
   (MONTH(SaleDate)),
   (YEAR(SaleDate)),
   (MONTH(ShipDate)),
   (YEAR(ShipDate)),
   (MONTH(DeliveryDate)),
   (YEAR(DeliveryDate)),
)
{% endhighlight %}

This kind of query can be useful for populating reporting tables but as you can probably see things become a bit messy as you add more aggregation points to the data. Although GROUPING SETS simplify multiple group by queries that would have previously been unioned together they start to fall down when you have a large number of groups. Enter ROLLUP....

ROLLUP allows us to aggregate at all levels of a given group, by using this we define a single group with the ROLLUP clause which will then group it at every level. For example...

{% highlight sql %}
SELECT
   MONTH(SaleDate) SalesMonth,
   YEAR(SaleDate) SalesYear, 
   MONTH(ShipDate) ShipMonth,
   YEAR(ShipDate) ShipYear, 
   MONTH(DeliveryDate) DeliveryMonth,
   YEAR(DeliveryDate) DeliveryYear,    
   COUNT(*) AS Amount
FROM 
   SalesReporting
GROUP BY 
   YEAR(SaleDate), MONTH(SaleDate), 
   YEAR(ShipDate),MONTH(ShipDate),
   YEAR(DeliveryDate), MONTH(DeliveryDate)
WITH ROLLUP
{% endhighlight %}

This will aggregate at each level of the group, below is an extract of the results showing the 2017 data...

SalesYear&nbsp;&nbsp; |  SalesMonth&nbsp;&nbsp; |  ShipYear&nbsp;&nbsp; |  ShipMonth&nbsp;&nbsp; |  DeliveryYear&nbsp;&nbsp; |  DeliveryMonth&nbsp;&nbsp; |  Amount
--- | ---| ---| ---| ---| ---| ---
2017 | 1 | 2017 | 1 | 2017 | 1 | 1
2017 | 1 | 2017 | 1 | 2017 | NULL | 1
2017 | 1 | 2017 | 1 | NULL | NULL | 1
2017 | 1 | 2017 | 2 | 2017 | 3 | 1
2017 | 1 | 2017 | 2 | 2017 | NULL | 1
2017 | 1 | 2017 | 2 | NULL | NULL | 1
2017 | 1 | 2017 | NULL | NULL | NULL | 2
2017 | 1 | NULL | NULL | NULL | NULL | 2
2017 | NULL | NULL | NULL | NULL | NULL | 2

Reading from the bottom we can see that aggregating by SalesYear we had 2 sales in 2017, above that we can see we have 2 sales in 2017, January. As we move up the list you'll see different aggregations for every level of group...

SalesYear<br/>
SalesYear, SalesMonth<br/>
SalesYear, SalesMonth, ShipYear<br/>
SalesYear, SalesMonth, ShipYear, Ship Month<br/>
SalesYear, SalesMonth, ShipYear, Ship Month, Delivery Year<br/>
SalesYear, SalesMonth, ShipYear, Ship Month, Delivery Year, Delivery Month<br/>

There is actually a 7th aggregation in the results missing from my grid above and that is the overall total which is null for all fields with a value of 4 showing there were 4 overall sales. 

As you can see ROLLUP takes a given group and aggregates it at every level. We can take this a step further by then saying we want to aggregate on every permutation of our group not just every level. For example for the group defined above we could also want to see how many sales in january shipped in february regardless of the year, the above ROLLUP will not show that as it will include the year because of the order the levels were defined. Enter CUBE, if we swap our ROLLUP clause for a CUBE clause instead of aggregating at every level of our group by it will aggregate at every possible permutation...

{% highlight sql %}
SELECT
   YEAR(SaleDate) SalesYear, 
   MONTH(SaleDate) SalesMonth,
   YEAR(ShipDate) ShipYear, 
   MONTH(ShipDate) ShipMonth,
   YEAR(DeliveryDate) DeliveryYear,    
   MONTH(DeliveryDate) DeliveryMonth,
   COUNT(*) AS Amount
FROM 
   SalesReporting
GROUP BY 
   YEAR(SaleDate), MONTH(SaleDate), 
   YEAR(ShipDate),MONTH(ShipDate),
   YEAR(DeliveryDate), MONTH(DeliveryDate)
WITH CUBE
{% endhighlight %}

If you run this you'll notice we now get significantly more results (174 to be exact) even from our small test sample of 4 rows, this is because we're aggregating at every possible permutation of our group by clause. This allows us to report aggregations at lots of possible points for example "How many sales were made in January (Regardless of year), Shipped In February and Delivered in 2018.

With these 3 techniques (GROUPING SETS, ROLLUP and CUBE) we have some powerful tools for reporting data baked into the SQL standard without having to use things like SSAS. 
