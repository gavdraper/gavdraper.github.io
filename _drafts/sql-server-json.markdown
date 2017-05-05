---
layout: post
title: Using JSON In SQL Server 2016
date: '2017-05-06 09:01:32'
---

The new JSON bits in SQL Server 2016 give you the ability to pull stuff out of JSON and to convert relational sets to JSON. This is similar to the XML features in SQL Server that have existed for some time now.

Lets create a table that we can run some examples on...

{% highlight sql %}
CREATE TABLE [UserJson]
(
    Id INT IDENTITY PRIMARY KEY,
    Forename NVARCHAR(100),
    Surname NVARCHAR(100),
    Username NVARCHAR(50),
    AdditionalInformation NVARCHAR(3000)
)
ALTER TABLE [UserJson] ADD CONSTRAINT [AdditionalInfoIsJson] CHECK (ISJSON(AdditionalInformation) > 0)
{% endhighlight %}

The only new thing is the constraint to force the AdditionalInformation field to be valid JSON. Unlike XML, JSON has no native type in SQL Server so we use the NVARCHAR type to store it. We can then use the new ISJSON method to test for JSON.

Now lets put an example record into this table.

{% highlight sql %}
INSERT INTO [UserJson] (Forename,Surname, Username,AdditionalInformation)
VALUES(
    'Gavin',
    'Draper',
    'GavinDraper',
    '
    {
        "Language" : "English",
        "FavoriteFoods" : ["Pizza","Chicken","Nachos"],
        "Address" : {
            "Town" : "Brighton",
            "Country" : "England"
        }
    }
    '
)
{% endhighlight %}

### Querying JSON ###

Now lets return a record that pulls out Country and Favorite foods from the JSON.

{% highlight sql %}
SELECT 
	Id,
	Forename,
	Surname,
	Username,
	JSON_VALUE(AdditionalInformation, '$.Address.Country') AS Country,
	JSON_QUERY(AdditionalInformation, '$.FavoriteFoods') AS Foods
FROM [UserJson]
{% endhighlight %}

We have 2 new bits of syntax in the above example 
1. JSON_VALUE - This will pull out the value of a single property
2. JSON_QUERY - This will pull out the contents of an array 

### Indexing JSON ###

### Converting Datasets to JSON ###