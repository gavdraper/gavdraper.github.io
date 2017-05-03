---
layout: post
title: Why CROSS APPLY is AWESOME
date: '2017-05-03 08:05:38'
---

CROSS APPLY was introducted as part of TSQL in SQL Server 2005. Origionally it was created as a way to join on table value functions. They are the perfect tool for TVF joins however what I've missed out on until recently is how they are also a great tool for table joins in certain situations. 

For example let's say we have a set that defines the second set it wants to join on. Imagine we have a setting for each user of our system that allows them to choose the amount of ads they want to see, we store this setting in a table called AdSettings, we then have a second table called Ads that we want to query to get the amount of ads each user has choosen. 

Obviously in this case our system could query the AdSettings table to get the AdCount then perform a seperate query to get the TOP(X) ads but that wouldnt demonstrate the power of CROSS APPLY, so to make this more useful lets say we want to cache all the Users and their specified ads in a table called UserAds. The final Schema looks like this...

| AdSettings | | | |  Ads | | | | UserAds | |
| --- | --- | ---| ---| ---| --- | --- | --- | 
| **Id** | INT | | | **Id** | INT | | | **Id** | INT | | 
| **Username** | NVARCHAR | | | **Name** | NVARCHAR | | |**UserName** | NVARCHAR |
| **AdCount** | SMALLINT | | | **ImagePath** | NVARCHAR | | | **AdId** | INT |

You can use this script to create the database and seed data if you want to follow along...

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
	AdName NVARCHAR(30),
	AdImagePAth NVARCHAR(100)
)

INSERT INTO dbo.AdSettings
        ( Username, AdCount )
SELECT N'Gavin',5
UNION ALL SELECT N'Joe',2 
UNION ALL SELECT  N'Sally',3 
UNION ALL SELECT N'Jane',5

INSERT INTO dbo.Ads
        ( Name, ImagePath )
SELECT 'Fizzy Drink', 'Fizzy.jpg'
UNION ALL SELECT 'Soft Drink', 'SoftDrink.jpg'
UNION ALL SELECT 'Flying Cars', 'FlyingCar.jpg'
UNION ALL SELECT 'Flying Cars 2', 'FlyingCar2.jpg'
UNION ALL SELECT 'Super Fast Laptop', 'UlraLaptop.jpg'
{% endhighlight %}

We can then use CROSS APPLY to populate the UserAds table with the following query.

{% highlight sql %}
INSERT INTO dbo.UserAds( Username, AdName,AdImagePAth )
SELECT  
	AdSettings.Username, 
	Ads.Name, 
	Ads.ImagePath 
FROM 
	AdSettings
	CROSS APPLY
        (
        	SELECT TOP (AdSettings.AdCount) *
        	FROM  Ads
        	ORDER BY id
        ) Ads
{% endhighlight %}

Given this data 

AdSettings

![SeedDataScreenshot]({{site.url}}/content/images/cross-apply/adsettings.JPG)

Ads

![SeedDataScreenshot]({{site.url}}/content/images/cross-apply/ads.JPG)

Our CROSS APPLY query will produce this...

![SeedDataScreenshot]({{site.url}}/content/images/cross-apply/userads.JPG)

You can see Gavin has 5 ads as the AdSettings for the Gavin user has an AdCount of 5 where as Sally has an AdCount of 3 and therefore only has 3 records in UserAds.
