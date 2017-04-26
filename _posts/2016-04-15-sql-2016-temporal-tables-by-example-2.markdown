---
layout: post
title: SQL 2016 Temporal Tables By Example
date: '2016-04-15 09:48:20'
---

## Brief Description
Temporal tables allow automated change tracking of a table directly in to a history table which can be queried separately or together using a new syntax. This has been part of the ANSI SQL standard since 2011 and is partially implemented in SQL Server 2016 RC2. Records get added to the history table after a Delete or Update operation however no history is created on Insert as the current record lives in the main table. 

The current table is called the temporal table and the table storing previous versions is referred to as the history table. They are 2 tables with identical schema linked by SQL Server. When creating them you can either create the 2 tables yourself then enable SYSTEM_VERSIONING on the current table to link the two or you can just enable it on the current table and have SQL Server create the history table for you.

## Simplified Example
The example below is for SQL Server 2016 RC2, assuming the syntax stays the same in future versions it should work in them too. For the most part you can use earlier versions of SQL Management Studio however if you use the 2016 version you will get some nicer UI for displaying history tables nested under the temporal table.

{% highlight sql %}
CREATE TABLE Users 
(
    Id INT IDENTITY PRIMARY KEY,
    Forename NVARCHAR(30) NOT NULL,
    Surname NVARCHAR(30) NOT NULL,
    Username NVARCHAR(30) NOT NULL,
    IsActive BIT NOT NULL,
    StartTime DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
    EndTime DATETIME2 GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (StartTime, EndTime)
) WITH(SYSTEM_VERSIONING = ON (HISTORY_TABLE=dbo.Users_History, DATA_CONSISTENCY_CHECK=ON));

INSERT INTO Users (Username,Forename,Surname,IsActive) VALUES('gdraper','Gav','Draper',1)
UPDATE Users SET Forename = 'Gavin' WHERE Forename = 'Gav' 

--Query history separately
SELECT * FROM Users
SELECT * FROM Users_History

--Query with fancy new syntax
DECLARE @CurrentDate DATETIME = GETDATE()
DECLARE @Tomorrow DATETIME = DATEADD(dd,1,@CurrentDate)

--Point in Time
SELECT * FROM Users FOR SYSTEM_TIME AS OF @CurrentDate 

--Period in Time
SELECT * FROM Users FOR SYSTEM_TIME FROM @CurrentDate TO @Tomorrow
{% endhighlight %}

Start and End columns can be marked as hidden if desired, this prevents them from showing up in SELECT * queries on the temporal table. In the example above SQL Server will create the User_History table for us as it doesn't exist but had it already existed SQL Server would have just linked the two.

## Entity Framework

The history and temporal table are schema bound together, this means changes you make to a schema require system versioning to be disabled whilst the change is made. For example to add a column you need to turn off system versioning, make the column change to both tables then turn it back on again, it's crucial that in this period no changes occur as they will not be tracked. This means any EF migrations made on a table with system versioning turned on will need to have either manual migrations or edits to the automatically generated migrations to handle this.

{% highlight sql %}
ALTER TABLE Users SET (SYSTEM_VERSIONING = OFF);
ALTER TABLE Users ADD COLUMN(IsAdmin BIT);
ALTER TABLE Users_History ADD COLUMN(IsAdmin BIT);
ALTER TABLE Users SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE=Users_History, DATA_CONSISTANCY_CHECK=ON)   
{% endhighlight %}    

  Once we've created our temporal tables, EF will know nothing of the history unless it's included in the model. For most cases it may be better to continue using EF to query current data and fall back to SQL or a micro ORM like dapper to query the history. 
     
## Performance
  Changes to indexes can be done without disabling the system versioning and the temporal/history table can both have different clustered/non clustered indexes. It may make sense for the history table’s clustered index to be ID, StartTime, and EndTime so you can easily get an ordered and filtered history for any object. 
  
Read and Insert performance on the current table will remain unaffected by enabling SYSTEM VERSIONING. Update/Delete performance will take a very minor hit as an insert into the history table will be triggered along with updating the period columns in the temporal table.