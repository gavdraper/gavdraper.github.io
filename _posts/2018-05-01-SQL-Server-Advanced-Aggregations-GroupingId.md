---
layout: post
title: SQL Server Advanced Aggregations Part 3 GROUPING/GROUPING_ID
date: '2018-05-01 20:11:34'
---
This post is part 3 of a 3 part series...

1.  [Part 1 Grouping Sets](https://gavindraper.com/2018/04/26/SQL-Server-Grouping-Sets-Explained/)
1.  [Part 2 ROLLUP and CUBE](https://gavindraper.com/2018/04/28/SQL-Server-Advanced-Aggregations/)
1.  Part 3 GROUPING and GROUPING_ID

For the final post in this series I'm going to cover GROUPING AND GROUPING_ID. If you've not gone through the previous posts in the series then it's worth giving them a look first as this post will follow straight on and examples below will build on previous ones. 

Following on from our CUBE and ROLLUP examples we ended up with a resultset full of aggregations at different levels/permutations of groups with no easy way to see at what level/permutation each row is grouped at. We can add GROUPING/GROUPING_ID to these queries to get group level information for each row describing what the row is grouped by. Lets modify the previous ROLLUP example to use the GROUPING clause....

{% highlight sql %}
SELECT
   GROUPING(MONTH(SaleDate)) GroupSaleDateMonth,
   GROUPING(Year(SaleDate)) GroupSaleDateYear,
   GROUPING(MONTH(ShipDate)) GroupShipDateMonth,
   GROUPING(MONTH(ShipDate)) GroupShipDateYeay,
   GROUPING(MONTH(DeliveryDate)) GroupDeliveryDateMonth,
   GROUPING(MONTH(DeliveryDate)) GroupDeliveryDateYear,
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

The 6 new columns identify which fields are included in the group for each row, with 0 being part of the group and 1 being excluded from the group. So a GROUPING(MONTH(SaleDate)) value of 0 means this field is part of the group for any aggregations in this row.

Lets now look at using GROUPING_ID instead of GROUPING...

{% highlight sql %}
SELECT
   GROUPING_ID(
	   YEAR(SaleDate), MONTH(SaleDate), 
	   YEAR(ShipDate),MONTH(ShipDate),
	   YEAR(DeliveryDate), MONTH(DeliveryDate)
   ) GroupingId,
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

GROUPING_ID takes the name of any fields included in the GROUP BY and will return a bit flag value that can be used to work out what fields are in the group. For example given the above sample the following values can be evaluated to work out what groups are in play...

| Value | YEAR Sale | Month Sale | Year Ship | Month Ship | Year Delivery | Month Delivery |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Yes | Yes | Yes | Yes | Yes | Yes |
| 1 | No | Yes | Yes | Yes | Yes | Yes |
| 2 | Yes | No | Yes | Yes | Yes | Yes |
| 3 | No | No | Yes | Yes | Yes | Yes |
| 4 | Yes | Yes |No | Yes | Yes | Yes |
| 5 | No | Yes | No | Yes | Yes | Yes |

Both the above examples also work for CUBE. Hopefully you can see how by using GROUPING AND GROUPING_ID you can describe your aggregations in your CUBES and ROLLUPS to show what groups are in play for any given aggregations. For more info on how these values translate look up Bit Flags on Google.


