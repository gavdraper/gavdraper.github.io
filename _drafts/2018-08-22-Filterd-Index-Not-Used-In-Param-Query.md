---
layout: post
title: Temporal Tables Causing High CPU
date: '2018-05-17 13:34:01'
---
I recently had an issue where queries against a temporal table's history table were causing the CPU to max out.When investigste the query was simple and had no reason to require so much CPU. Before I explain the solution let's recreate it....

First let's create a simple temporal table...

{% highlight sql %}
    CREATE TABLE SensorReading
    (
        Id INT IDENTITY PRIMARY KEY,
        SensorName NVARCHAR(30),
        SensorReading1 DECIMAL(24,10),
        SensorReading2 DECIMAL(24,10),
        SensorReading3 DECIMAL(24,10),
        SensorReading4 DECIMAL(24,10),
        SensorReading5 DECIMAL(24,10),
        SensorReading6 DECIMAL(24,10), [ValidFrom] datetime2 (2) GENERATED ALWAYS AS ROW START  
  , [ValidTo] datetime2 (2) GENERATED ALWAYS AS ROW END  
  , PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)  
 )    
 WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.EmployeeHistory));  
    )
{% endhighlight %}
