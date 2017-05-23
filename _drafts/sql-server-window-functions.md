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
WHILE @LoopCount < 5
    BEGIN
    INSERT INTO [dbo].[User](Username,FirstName,LastName)
    SELECT 
        CAST(@LoopCount AS NVARCHAR(4)) + Username,
        CAST(@LoopCount AS NVARCHAR(4)) + FirstName,
        CAST(@LoopCount AS NVARCHAR(4)) + LastName 
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
		[User]
	) u
WHERE 
	u.RowNumber BETWEEN 1 AND 10
{% endhighlight %}

In this case our window function is ordering the data by Username and applying the ROWNUMBER() function to it which just gives each row an incrementing number. In our outter query we then filter by the row number returned.


