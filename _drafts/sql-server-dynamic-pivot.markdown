---
layout: post
title: SQL Server Dynamic Pivot
date: '2017-05-07 13:15:32'
---
Often when trying to pivot data you wont know what the possible values that you need to pivot on, in this case you can use dynamic SQL.

Lets say we have a table with employee sale counts

| Employee | Sales |
| --- | --- |
| Gavin | 10 |
| Troy | 4 | 
| Joe | 3 |

We can pivot this data using the standard Pivot syntax like this

{% highlight sql %}
SELECT 
    [Gavin],
    [Troy],
    [Joe]
FROM
    (SELECT Employee, Sales FROM EmployeeSales) AS SourceTable
    PIVOT
    (
        SUM(Sales)
        FOR Employee IN
        (
            [Gavin], [Troy], [Joe]
        )
    ) AS PivotTable
{% endhighlight %}

The problem is as new employees come into our table this query wont pick them up as we've not explicitly named them. We can get round this by using dynamic SQL with the Pivot statement...

{% highlight sql %}
DECLARE @Columns NVARCHAR(MAX) = ''
SELECT @Columns  = 
    @Columns + QUOTENAME(Employee) + ',' 
        FROM (SELECT Employee FROM EmployeeSales ) t
SELECT @Columns = SUBSTRING(@Columns, 0, LEN(@Columns))        
    
DECLARE @Query NVARCHAR(MAX) =  
    'SELECT ' + @Columns + ' FROM
        (SELECT Employee, Sales FROM EmployeeSales) SourceTable
        PIVOT 
        (
            SUM(Sales)
            FOR Employee IN (' + @Columns + ')
        ) AS PivotTable ' 

EXECUTE(@query)
{% endhighlight %}

The above query will build a dynamic Pivot SQL statement which will get all possible employee values from the EmployeeSales table and pivot on that. With this you can easily customize the query to limit which employees get pivoted on, for example only the top 2 performers...

{% highlight sql %}
DECLARE @Columns NVARCHAR(MAX) = ''
SELECT @Columns  = @Columns + 
    QUOTENAME(Employee) + ',' 
    FROM 
    (
        SELECT TOP 2 Employee 
        FROM EmployeeSales ORDER BY Sales DESC
    ) t
SELECT @Columns = SUBSTRING(@Columns, 0, LEN(@Columns))        
    
DECLARE @Query NVARCHAR(MAX) =  
    'SELECT ' + @Columns + ' FROM
        (SELECT Employee, Sales FROM EmployeeSales) SourceTable
        PIVOT 
        (
            SUM(Sales)
            FOR Employee IN (' + @Columns + ')
        ) AS PivotTable ' 

EXECUTE(@query)
{% endhighlight %}
