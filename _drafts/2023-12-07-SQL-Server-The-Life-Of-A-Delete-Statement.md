---
layout: post
title: Deleting Lots Of Data? Go Home SQL Server You're Doing It Wrong!
date: '2019-01-08 06:34:01'
---
After recently spending some time trying to optimize queries doing some fairly large scale deletes I discovered things don't work quite as optimally as I'd expected when deleting from multiple indexes. Before I go into too much detail a couple of things to understand first...

## Before We Start : Indexes In Indexes ##

* If a table has no clustered indexes then each record in the NonClustered indexes points back to the data in the table (HEAP) with its Row Identifier (RID) this is an identifier that points to the file, page and slot in page of the underlying row.
* If a table does have a Clustered index then the NonClustered indexes do not use the RID to point back to data in the clustered index, they instead use the fields in the clustered index. This is a little surprising to learn at first as for this to work it means that every NonClustered index you have also has the clustered index fields included in it in order to get back to the underlying rows in the table.

Lets demo the above points with a quick sample...

{% highlight sql %}
CREATE TABLE NonClusteredOnly
(
   Id INT IDENTITY,
   Name NVARCHAR(100),
   INDEX ndx_nonclusteredonly_name NONCLUSTERED(Name)
)

CREATE TABLE ClusteredOnly
(
   Id INT IDENTITY PRIMARY KEY,
   Name NVARCHAR(100)
)

CREATE TABLE ClusteredAndNonClustered
(
   Id INT IDENTITY PRIMARY KEY,
   Name NVARCHAR(100),
   INDEX ndx_clusteredandnonclustered_name NONCLUSTERED(Name)
)

INSERT INTO NonClusteredOnly (Name)
SELECT specific_name
FROM msdb.information_schema.routines 
WHERE routine_type = 'PROCEDURE'

INSERT INTO ClusteredOnly (Name)
SELECT specific_name
FROM msdb.information_schema.routines 
WHERE routine_type = 'PROCEDURE'

INSERT INTO ClusteredAndNonClustered (Name)
SELECT specific_name
FROM msdb.information_schema.routines 
WHERE routine_type = 'PROCEDURE'
{% endhighlight %}

After running that you'll have 3 tables and about 500 records in each depeneding on which version of SQL Server you're running.

Let's first look at what happens if we filter on the name column for select all columns from ClusteredOnly...

{% highlight sql %}
SELECT id,[Name] FROM ClusteredOnly
WHERE [Name] LIKE 'sp_verify_job%'
{% endhighlight %}

![Clustered Only Plan]({{site.url}}/content/images/2019-Theres-An-Index-In-My-Index\clustered-only-plan.PNG)

As you probably expected we have no index that can help with our predicate so it does a full scan on the clustered index.

Now let's run that same query on our table with only a single NonClustered index...

{% highlight sql %}
SELECT id,[Name] FROM NonClusteredOnly
WHERE [Name] LIKE 'sp_verify_job%'
{% endhighlight %}

![NonClustered Only Plan]({{site.url}}/content/images/2019-Theres-An-Index-In-My-Index\nonclustered-only-plan.PNG)

In this case we're using the NonClustered index to seek on our predicate but because that index doesnt have ID in it and our query wants the ID field it then uses the RID (Record Identifier) to go back to the heap and get that data in that field.

It gets more interesting when we run that same query on our table with a Clustered index on ID and NonClustered on Name, the obvious thing you'd expect to see is probably seek on the NonClustered then a lookup to the Clusted index to get the ID field but...

{% highlight sql %}
SELECT id,[Name] FROM ClusteredAndNonClustered
WHERE [Name] LIKE 'sp_verify_job%'
{% endhighlight %}

![NonClustered and Clustered Plan]({{site.url}}/content/images/2019-Theres-An-Index-In-My-Index\clustered-and-nonclustered-plan.PNG)

## Setting Up a Sandbox ##

### Delete No Index ###
### Delete Clustered Index ###
### Delete Non Clustered Index ###
### Delete Clustered and One Non Clustered Index ###
### Delete Clusterd and Multiple Non Clustered Indexes ###