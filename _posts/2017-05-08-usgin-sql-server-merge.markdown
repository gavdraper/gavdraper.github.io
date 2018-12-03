---
layout: post
title: Using The SQL Server Merge Statement
date: '2017-05-08 06:05:38'
tags: tsql
---

> Note : Since publishing this I've been notified that there are a number of outstanding issues with the SQL Server merge statement. I currently work with a system the heavily relies on the Merge statement and we've not knowingly hit any of these issues but it is worth knowing about them. The list can be found here [https://www.mssqltips.com/sqlservertip/3074/use-caution-with-sql-servers-merge-statement/](https://www.mssqltips.com/sqlservertip/3074/use-caution-with-sql-servers-merge-statement/) , it's unclear however if any of these issues are resolved in SQL 2016 as most of them have either been closed or commented from Microsoft saying they will be fixed in the next major version which would have been SQL 2016.

The SQL Server merge statement kind of does what it says, given some source data and a table it can UPDATE data that already exists but has changed, INSERT data that is not in the destination and remove data from the destination that is not in the source. Each of these actions is optional and specified in the statement. I've mainly used this for syncing/importing  data across multiple databases.

Take this table as our Source

| Id | Username | Location | FavoriteColour |
| --- | --- | --- | --- |
| 1 | Gavin | UK | Red |
| 2 | Jane | USA | Purple |
| 3 | Joe | France | Pink |

Then lets say we're trying to merge this data into another table with the following contents

| Id | Username | Location | Age |
| --- | --- | --- | --- |
| 1 | Jane | USA | Pink | 32 |

The target table has an extra field (Age) which we will set to null when inserting new records from our source table as at the point we don't have that data. Once we've written and run our merge script the target table should look like this...

| Id | Username | Location | FavoriteColour |Age |
| --- | --- | --- | --- | --- |
| 1 | Jane | USA | Purple | 32 |
| 2 | Gavin | UK | Red | NULL |
| 3 | Joe | France | Pink | NULL |

So Jane's record got updated with her favorite colour from the source table, and Gavin/Joe got inserted as they didn't already exist on the target table.

A merge needs to match on a specific column or set or columns(The ON Clause) in this case we'll use username, so if the username doesn't exist in the target we'll insert it and if it does we'll update it. Just to add an extra bit of complexity to the example we're only going to update the record in the target if the favorite colour is different. 

Lets pretend the two tables above are called UserSource and UserTarget we can write the merge like this...

{% highlight sql %}
MERGE
    UserTarget AS Target
    USING (SELECT * FROM UserSource) AS Source
	ON Source.Username = Target.Username
WHEN MATCHED AND Source.FavoriteColour <> Target.FavoriteColour THEN
    UPDATE SET FavoriteColour = Source.FavoriteColour
WHEN NOT MATCHED BY TARGET THEN
    INSERT(Username,Location,FavoriteColour)
    VALUES(Source.Username, Source.Location, Source.FavoriteColour);
{% endhighlight %}

We could have also done things the other way around and used NOT MATCHED BY TARGET to update or delete records in the source. 

So the key components of the merge statement are

1. The Target

    ``` UserTarget AS Target ```
2. The Source
    
    ``` USING (SELECT * FROM UserSource) AS Source ```
3. The ON clause to perform the match

    ``` ON Source.Username = Target.Username ```
4. The WHERE statements to defined what happens when data does or does not exist. Remember you can have multiple of these and can add extra clauses to take different paths. In our example we only update if their favorite colour has changed
    
    ``` WHEN MATCHED AND Source.FavoriteColour <> Target.FavoriteColour THEN```

The Merge statement also has an OUTPUT clause that we can use to get a summary of what the merge has done. For example given the above example we can select out what actions were taken...

{% highlight sql %}
MERGE
    UserTarget AS Target
    USING (SELECT * FROM UserSource) AS Source
	ON Source.Username = Target.Username
WHEN MATCHED AND Source.FavoriteColour <> Target.FavoriteColour THEN
    UPDATE SET FavoriteColour = Source.FavoriteColour
WHEN NOT MATCHED BY TARGET THEN
    INSERT(Username,Location,FavoriteColour)
    VALUES(Source.Username, Source.Location, Source.FavoriteColour)
 OUTPUT deleted.*, $action, inserted.* ;
{% endhighlight %}

From that we get this...

| Id | Username | Location | FavoriteColor | Age | $Action | Id | Username | Location | FavoriteColour | Age |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| NULL | NULL | NULL | NULL | NULL | INSERT | 2 | Gavin | UK | Red | NULL |
| NULL | NULL | NULL | NULL | NULL | INSERT | 3 | Joe | France | Pink | NULL |
| 1 | Jane | USA | Pink | 32 | UPDATE | 1 | Jane | USA | Purple | 32 |

This shows us on the left how the row in the target table used to look and on the right how it looks after the merge.

If you wanted to play with the above examples then the following script will setup the tables and data...

{% highlight sql %}
DROP TABLE UserSource
DROP TABLE UserTarget

CREATE TABLE UserSource
(
    Id INT IDENTITY PRIMARY KEY,
    Username NVARCHAR(20),
    [Location] NVARCHAR(20),
    FavoriteColour NVARCHAR(20)
)

CREATE TABLE UserTarget
(
    Id INT IDENTITY PRIMARY KEY,
    Username NVARCHAR(20),
    [Location] NVARCHAR(20),
    FavoriteColour NVARCHAR(20),
    Age INT
)

INSERT INTO UserSource(Username,Location,FavoriteColour)
VALUES
    ('Gavin','UK','Red'),
    ('Jane','USA','Purple'),
    ('Joe','France','Pink')

INSERT INTO dbo.UserTarget( Username, Location, FavoriteColour, Age)
VALUES
    ('Jane','USA','Pink',32)
{% endhighlight %}