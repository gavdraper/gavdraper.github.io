---
layout: post
title: Using JSON In SQL Server 2016
date: '2017-05-06 09:01:32'
tags: tsql json
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
    '{
        "Language" : "English",
        "FavoriteFoods" : ["Pizza","Chicken","Nachos"],
        "Address" : {
            "Town" : "Brighton",
            "Country" : "England"
        }
    }'
)
{% endhighlight %}

### Extracting JSON ###

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

This will return 

| Id | Forename | Surname | Username | Country | Foods |
| --- | ---| ---| ---| ---| ---|
| 1 | Gavin | Draper | GavinDraper | England | ["Pizza", "Chicken", "Nachos"] |

We have 2 new bits of syntax in the above example 
1. JSON_VALUE - This will pull out the value of a single property
2. JSON_QUERY - This will pull out the contents of an array 

As JSON is not bound to a schema you will probably have some records that have JSON properties that other records don't, this is one of the benefits of using a schemaless object, in this case both JSON_VALUE and JSON_QUERY wont error if you specify a field that doesn't exist they will just return null.

### Convert JSON To A SQL Set ###
We can use OPENJSON to convert and manipulate the JSON into a SQL Set. Lets say given the JSON in the table above we want to produce a typed list of fields containing Language and Town. The WITH statement on OPENJSON allows us the specify the properties we want and the fields we want them to go to along with type information.

{% highlight sql %}
DECLARE @Json NVARCHAR(3000)
SELECT @Json = AdditionalInformation FROM UserJson WHERE Username = 'GavinDraper'
SELECT * FROM OPENJSON(@Json) 
WITH 
(
	Language NVARCHAR(50) '$.Language', 
	[Town] NVARCHAR(50) '$.Address.Town'
)
{% endhighlight %}

This will output

| Language | Town |
| --- | --- |
| English | Brighton |

We can also query JSON arrays and produce a row for each item in the array, this can be done by specifying a path to the array in OPENJSON

{% highlight sql %}
SELECT [Value] FROM OPENJSON(@Json, '$.FavoriteFoods')
{% endhighlight %}

Which produces the following output

| Value |
| --- |
| Pizza |
| Chicken |
| Nachos |

### Indexing JSON ###

Indexing doesnt exist for JSON in the same way it does in NOSQL Databases. However what you can do is create a computed field based on one or more properties in the JSON and index on that. For example lets say we want to index on Town from the above example we can do this...

{% highlight sql %}
ALTER TABLE UserJson 
ADD Town AS 
    CAST(JSON_VALUE(AdditionalInformation, '$.Address.Town') AS NVARCHAR(100))

CREATE NONCLUSTERED INDEX ndx_userjson_town ON UserJson(Town)
{% endhighlight %}

I added the cast to NVARCHAR(100) to limit it's length and give some constraint to this data. We can now use the new index on town by querying the new field directly.

### Converting Datasets to JSON ###

Lets imagine we want Forename,Surname and Username as a JSON object. The new FOR JSON clause can do this.

{% highlight sql %}
SELECT
    Id AS [Person.Id],
    Forename AS [Person.Forename],
    Surname AS [Person.Surname]
FROM UserJson
FOR JSON PATH
{% endhighlight %}

The FOR JSON PATH syntax will convert to JSON and use the dot syntax in our field names to create the object hierarchy. The above query produces this JSON...

{% highlight json %}
[{
    "Person":{
        "Id":5,
        "Forename":"Gavin",
        "Surname":"Draper"
    }
}]
{% endhighlight %}

### Update JSON ###
The JSON_MODIFY syntax allows us to modify a JSON string. For example lets say the user in the above table has moved to London and we want to update the town in the JSON (This will also update the computed field).

{% highlight sql %}
UPDATE [UserJson] 
SET AdditionalInformation = 
    JSON_MODIFY(AdditionalInformation,'$.Address.Town','London') 
WHERE Username = 'GavinDraper'
{% endhighlight %}

As you can see JSON_MODIFY takes the JSON to modify, the property you want to change followed by the new value. After running the above SQL the town for the GavinDraper user will be set to London.
