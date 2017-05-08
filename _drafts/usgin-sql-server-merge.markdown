---
layout: post
title: Using SQL Server Merge Statement
date: '2017-05-08 13:05:38'
---
The SQL Server merge statement kind of does what it says, given some source data and a destination table it can UPDATE data that already exists but has changed, INSERT data that is not in the destination, remove data from the destination that is not in the source.

I've mainly used this for syncing/importing  data across multiple databases.

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

So Jane's record got updated with her favorite colour from the source table, and Gavin/Joe got inserted as they didnt already exist on the target table.

A merge needs to match on a specific column or set or columns in this case we'll use username, so if the username doesnt exist in the target we'll insert it and if it does we'll update it.

Lets pretend the two tables above are called UserSource and UserTarget we can write the merge like this...

{% highlight sql %}
MERGE
    UserTarget AS Target
    USING (SELECT * FROM UserSource) AS Source
	ON Source.Username = Target.Username
WHEN MATCHED AND Source.FavoriteColour <> Target.FavoriteColour THEN
    -- We have the record in both tables but the colour in the target table is different to source. 
    -- Lets update it...
    UPDATE SET FavoriteColour = Source.FavoriteColour
WHEN NOT MATCHED BY TARGET THEN
    -- Record doesnt exist in target table (Lets Insert It)
    INSERT(Username,Location,FavoriteColour)
    VALUES(Source.Username, Source.Location, Source.FavoriteColour);
{% endhighlight %}

We could have also done things the other way around and used NOT MATCHED BY TARGET to update or delete records in the source. 

The Merge statement also has an OUTPUT clause that we can use to get a summary of what the merge has done. For example given the above example we can 


{% highlight sql %}
MERGE
    UserTarget AS Target
    USING (SELECT * FROM UserSource) AS Source
	ON Source.Username = Target.Username
WHEN MATCHED AND Source.FavoriteColour <> Target.FavoriteColour THEN
    -- We have the record in both tables but the colour in the target table is different to source. 
    -- Lets update it...
    UPDATE SET FavoriteColour = Source.FavoriteColour
WHEN NOT MATCHED BY TARGET THEN
    -- Record doesnt exist in target table (Lets Insert It)
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

