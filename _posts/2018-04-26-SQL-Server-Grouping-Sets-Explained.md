---
layout: post
title: SQL Server Advanced Aggregations Part 1 Grouping Sets
date: '2018-04-26 21:21:01'
---

This post is part 1 of a 3 part series...

1)  [](Part 1 Grouping Sets)
1)  [](Part 2 ROLLUP and CUBE)
1)  [](Part 3 GROUPING and GROUPING_ID)

Grouping Sets can be a powerful tool for reporting aggregations. Let's imagine we have a sales table with a SaleDate and want a count of sales by month and also by year....

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

Given this dataset we can achieve the desired results by unioning two separate groups...

{% highlight sql %}
SELECT 
    MONTH(SaleDate) AS Month,
    NULL AS Year,    
    COUNT(*) Sales
FROM    
    Sales
GROUP BY MONTH(SaleDate)

UNION ALL

SELECT 
    NULL AS Month,
    YEAR(SaleDate) AS Year,    
    COUNT(*) Sales
FROM    
    Sales
GROUP BY YEAR(SaleDate)

{% endhighlight %}

Whilst this works it quickly becomes as mass of code as you add more groups you want to aggregate by. Enter Grouping Sets, these allow you for a given query to define multiple group by clauses. For example the above query could be rewritten as....

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

With this solution we just need to add additional fields to the select/grouping sets as we want to aggregate by more groups.