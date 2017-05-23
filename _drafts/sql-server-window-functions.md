---
layout: post
title: SQL Server Window Functions
date: '2017-05-23 07:20:38'
---
SQL Server Window Functions were introduced in SQL Server 2005 with a basic set of operators and massively upgraded in SQL Server 2012 to include a lot more operators.

A window function is a function that takes a window descriptor which describes the subset of rows on the overall dataset it operates on and returns a single value per row. The OVER keyword in SQL Server opens a window function. That probably sounds more complex than it actually is, let's take a simple example...

Imagine we have a table of users and we want to be able to page that table in our application rather than get all the data back in one go. In this case we want the table ordered by Username, for paging to work each row in order of username will need a row number so we can say for page 1 get records 1-10 for page 2 get records 11-20 etc...

The following script will setup a user table and insert some records so you can follow along...

{% highlight sql %}
DROP TABLE [User]
CREATE TABLE [User]
(
    Id INT IDENTITY PRIMARY KEY NONCLUSTERED,
    Username NVARCHAR(100),
    Firstname NVARCHAR(100),
    LastName NVARCHAR(100)
)

INSERT INTO [dbo].[User]
    ( 	
    Username,
    FirstName,
    LastName
	)
VALUES
    ('clairetemple','Claire','Temple'),
    ('lukecage','Luke','Cage'),
    ('jessiejones','Jessie','Jones'),
    ('tonystark','Tony','Stark'),
    ('mattmurdock','Matt','Murdock')

DECLARE @LoopCount INT = 1
WHILE @LoopCount < 4
    BEGIN
    INSERT INTO [dbo].[User](Username,FirstName,LastName)
    SELECT 
        CAST(@LoopCount AS NVARCHAR(4)) + Username,
        FirstName,
        LastName 
    FROM [dbo].[User]
    SET @LoopCount = @LoopCount +1
    END
{% endhighlight %}

Lets say we want 10 records per page and we want to get page 2, we can use a window function to assign each row a number in username order, we can then filter the query to row number > 11 and row number < 21...

{% highlight sql %}
SELECT * FROM 
    (
    SELECT
        ROW_NUMBER() OVER (ORDER BY [Username]) RowNumber,
        [Username],
        [Firstname],
        [Lastname]
    FROM 	
        [dbo].[User]
    ) u
WHERE 
    u.RowNumber BETWEEN 1 AND 10
{% endhighlight %}

In this case our window function is ordering the data by Username and applying the ROWNUMBER() function to it which just gives each row an incrementing number. In our outter query we then filter by the row number returned.

We can use the PARTITION BY syntax to apply our Window Function on a set of data, for example if we PARTITION BY and ORDER BY Firstname then as we have 10 users for user name we'll get a row number of 1-8 for each firstname that resets...

{% highlight sql %}
SELECT * FROM 
    (
    SELECT
        ROW_NUMBER() OVER (PARTITION BY Firstname ORDER BY Firstname) RowNumber,
        [Id],
        [Firstname],
        [Lastname]
    FROM 	
        [dbo].[User]
    ) u
{% endhighlight %}

![Row Number Partition By Result Set]({{site.url}}/content/images/2017-window-functions/rownumber-partition.JPG)

There are many different Windows functions now available in SQL Sever and we've just looked a ROW_NUMBER. Let's move on and look at aggregates SUM, AVG, COUNT, MIN and MAX all work with window functions. At first glance it may seem like you can do all that with a join so why use a window function? Let's look at another example.

Let's imagine given the user table we have above we want to get each user along with a count of how many users exist with the same first name. In this case we can't use a group by as we want the user id in the result set and if we group by that everything will have a firstname count of 1, for example...

{% highlight sql %}
SELECT
    [User].Id,
    [User].Username,
    [User].Firstname,
    [User].LastName,
    COUNT(DISTINCT Firstname) UsersWithThisName
FROM
    [dbo].[User]
GROUP BY
    [User].Id,
    [User].Username,
    [User].Firstname,
    [User].LastName
{% endhighlight %}

![Group By ResultSet]({{site.url}}/content/images/2017-window-functions/groupby.JPG)

A groupby works great for aggregates in most cases because we normally actually want to group the data. In this case however we want all the data along with an aggregate on each record. Let's look at how we can use a window function to partition on firstname and do this...

{% highlight sql %}

SELECT
    [User].Id,
    [User].Username,
    [User].Firstname,
    [User].LastName,
    COUNT(*) OVER (PARTITION BY FirstName) UsersWithThisName
FROM
    [dbo].[User]
{% endhighlight %}

We're telling our window function to count each set of data with the same firstname .

![Partition By ResultSet]({{site.url}}/content/images/2017-window-functions/partitionby.JPG)

### Bounding ###
Window functions allow you to specify an upper and lower bounds for the function to run in, in combination with an order by this can be a really handy tool. Let's look at an example to see why...

Imagine we have a sales aggregate table that stores quantity and value against a month.

{% highlight sql %}
CREATE TABLE SalesWarehouse
(
	ID INT IDENTITY PRIMARY KEY,
	YearMonth INT,
	Quantity INT,
)
INSERT INTO dbo.SalesWarehouse( YearMonth, Quantity)
VALUES 
    (201601, 10),
    (201602, 5),
    (201603, 13),
    (201604, 6),
    (201605, 51),
    (201606, 0),
    (201607, 3),
    (201608, 6),
    (201609, 77),
    (201610, 65),
    (201611, 55),
    (201612, 60)
{% endhighlight %}

Let's imagine that we want to see the qunatity for each month along side the quantity for the month before and after it...

{% highlight sql %}
SELECT 
    YearMonth,
    Quantity,
    SUM(Quantity) OVER (ORDER BY YearMonth ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) LastMonth,		
    SUM(Quantity) OVER (ORDER BY YearMonth ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING) NextMonth	
FROM
    [dbo].SalesWarehouse
{% endhighlight %}

![Bounding ResultSet]({{site.url}}/content/images/2017-window-functions/bounding.JPG)

You can see in the above example we bounded last month to start and end on the previous record by doing this...

{% highlight sql %} 
ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
{% endhighlight %}

We could easily have aggregates the current month with the month before and after by bounding 1 PRECEDING and 1 FOLLOWING. This is great for things like rolling statement aggregations. 

Along with ROW_NUMER and the usual aggreagate operations like SUM,COUNT, MIN, MAX etc SQL Server also has a number of other window functions available these are...

| SQL 2005+ | SQL 2012+ |
| --- | --- |
| RANK | FIRST_VALUE |
| DENSE_RANK | LAST_VALUE |
| NTILE | CUME_DIST |
| | PERCENT_RANK |
| | PERCENTILE_DISC |
| | PERCENTILE_CONT |
| | LEAD |
| | LAG |

Descriptions of what each of these do with examples can be found on MSDN. 