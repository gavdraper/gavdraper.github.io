---
layout: post
title: Work Out The Distance Between 2 Locations In SQL Server
date: '2011-02-16 09:28:25'
---

SQL Server 2008 introduced the Geography data type for working with location data. One of the things this new types makes very easy to do is get the distance between 2 locations.

Lets say you store your users longitude and latitude information as something like Decimal(9,6) and want to find out the distance between 2 users, this is trivial using the new data type.

Add a new field to the table called GeoLocation of type Geography. You will then need to populate this field with the data based on your existing longitude/latitude fields. You can generate the Geography datatype from the Longitude and Latitude using the STGeomFromText method which as the name implies converts a string to a Geography data type.
<pre class="brush: sql;toolbar:false">UPDATE
	Users
SET
	Geolocation = GEOGRAPHY ::STGeomFromText
		('POINT
			(' + 
			CONVERT(VARCHAR(30),Longitude) +' ' + 
			CONVERT(VARCHAR(30),Latitude) +
			')',
		 4326
		)</pre>
You then need to change your application so whenever a user is created or modified it updates the GeoLocation field to match the Longitude/Latitude fields. Now for the cool part let's say you want to create a stored procedure that for a given user will find all other users within a specified distance...
<pre class="brush:sql;toolbar: false;">CREATE PROCEDURE FindLocalUsers
(
   @UserId INT,
   @DistanceInMiles INT
)
AS
 
DECLARE @GeoLocation GEOGRAPHY
SELECT @GeoLocation = GeoLocation FROM Users WHERE UserId = @UserId
 
SELECT
    *
FROM
    Users
WHERE
    @DistanceInMiles = 0 OR
    Users.Geolocation.STDistance(@GeoLocation)&lt;=(@DistanceInMiles * 1609.344)</pre>
The STDistance function returns the metres between the specified users GeoLocation and the other users locations we then convert this to miles by multiplying it by 1609.344 and there you have it a stored procedure that can work out the distance between 2 locations and filter accordingly. If you no longer need the Longitude and Latitude fields you could remove them and work only with the Geography type.