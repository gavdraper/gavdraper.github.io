---
layout: post
title: Inserting A Stored Procedures Results In To A Table
date: '2017-05-03 08:05:38'
---

CROSS APPLY was introducted as part of TSQL in SQL Server 2005. Origionally it was created as a way to join on table value functions. They are the perfect tool for TVF joins however what I've missed out on recently is how they are also a great tool for table joins in certain situations. 

For example let's say we have a set that defines the second set it wants to join on. Imagine we have a setting for each user of our system that allows them to choose the amount of ads they want to see, we store this setting in a table called AdSettings, we then have a second table called Ads that we want to query to get the amount of ads the user has choosen. 

### AdSettings
| Field | Type |
| --- | --- |
| Id | INT |
| Username | NVARCHAR |
| AdCount | SMALLINT |

### Ads
| Field | Type |
| --- | --- |
| Id | INT | 
| Name | NVARCHAR |
| ImagePath | NVARCHAR |

Obviously in this case on our website we could query the AdSettings table to get the AdCount then perform a seperate query to get the ads but that wouldnt demonstrate the user to CROSS APPLY, so to make this more useful lets say we want  to cache all the Users and their specified ads in a table called UserAds

###UserAds
| Field | Type |
| --- | --- |
| Id | INT |
| UserName | NVARCHAR |
| AdId | INT |

We can then use CROSS APPLY to populate this table with the following query. If you want to follow along the script to create and populate this database is at the bottom of this post.



The following script will setup a database and some sample data you can use to follow through the above examples.

{% highlight sql %}
CREATE TABLE AdSettings
(
	Id INT IDENTITY PRIMARY KEY,
	Username NVARCHAR(30),
	AdCount SMALLINT DEFAULT(5)
)

CREATE TABLE Ads
(
	Id INT IDENTITY PRIMARY KEY,
	Name NVARCHAR(30),
	ImagePath NVARCHAR(100)
)

CREATE TABLE UserAds
(
	Id INT IDENTITY PRIMARY KEY,
	Username NVARCHAR(30),
	AdId INT FOREIGN KEY REFERENCES dbo.Ads(Id)
)
{% endhighlight %}