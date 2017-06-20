---
layout: post
title: SQL Server 2017 Concatenation With STRING_AGG
date: '2017-06-20 07:54:01'
---
One of the new less publicized features in SQL Server 2017 is STRING_AGG. This new feature can take an expression of string values and concatenate them with a specified separator.

There's not much more to say about it other than that really, so lets look at an example. Imagine we have a user table and we want to provide our application with a comma separated list of all the usernames in the table...

{% highlight sql %}
CREATE TABLE dbo.[User]
(
    Id INT IDENTITY PRIMARY KEY,
    Username NVARCHAR(20)
)

INSERT INTO dbo.[User](Username)
VALUES 
    ('LukeCage'),
    ('JessicaJones'),
    ('BruceWayne')

SELECT STRING_AGG(ISNULL(Username,'Not Specified'),' , ') 
FROM dbo.[User]
{% endhighlight %}

This kind of concatenation has always been a bit of a pain before SQL Server 2017, usually achieved using xml methods or variables and coalesce, now it couldnt be any easier.