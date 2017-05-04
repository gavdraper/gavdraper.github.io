---
layout: post
title: SQL Server Indexed Views Explained
date: '2012-05-02 08:06:16'
---

Indexed views are a really powerful and possibly underused feature of SQL Server. It’s basically a way of writing a view then have SQL materialize it to a physical table and maintain it updating the view as its underlying data changes. As with all things like this there are a few limitations where they cant be used, there is a comprehensive list of these limitations here <a href="http://sqlheaven.blogspot.co.uk/2011/06/indexed-view-limitations.html">here</a>.

For this post I’m going to be using the following schema for my examples…

{% highlight sql %}
CREATE TABLE [dbo].[GroupTypes](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Name] [nvarchar](50) NOT NULL
) ON [PRIMARY]
CREATE CLUSTERED INDEX pk_GroupTypes ON GroupTypes(Id)

CREATE TABLE [dbo].[Groups](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Name] [nvarchar](50) NOT NULL,
	[TypeId] [int] NOT NULL
) ON [PRIMARY]
CREATE CLUSTERED INDEX pk_Groups ON Groups(Id)

CREATE TABLE [dbo].[Users](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Username] [nvarchar](80) NOT NULL,
	[Firstname] [nvarchar](80) NOT NULL,
	[Surname] [nvarchar](80) NOT NULL
) ON [PRIMARY]
CREATE CLUSTERED INDEX pk_Users ON Users(Id)

CREATE TABLE [dbo].[UserGroups](
	[UserId] [int] NOT NULL,
	[GroupId] [int] NOT NULL
) ON [PRIMARY]
CREATE CLUSTERED INDEX pk_UserGroups ON UserGroups(UserId,GroupId)
CREATE INDEX ix_UsersGroups_Group ON UserGroups(GroupId) INCLUDE(UserId)
{% endhighlight %}

If you want to follow along then you will want to use a tool like <a href="http://www.red-gate.com/">RedGates</a> SQL Data Generator to populate the tables with 100,000 or so records.

For the examples sake lets say we want query that returns a record for each group that each user is in, the query might look something like this…

{% highlight sql %}
SELECT 
	Users.Id AS UserId,
	Users.UserName,
	GroupTypes.Id AS GroupTypeId,
	GroupTypes.Name AS GroupType,
	Groups.Id AS GroupId,
	Groups.Name AS [Group]
FROM
	dbo.Users
	INNER JOIN dbo.UserGroups ON UserGroups.UserID = Users.Id
	INNER JOIN dbo.Groups ON UserGroups.GroupId = Groups.Id
	INNER JOIN dbo.GroupTypes ON Groups.TypeId = GroupTypes.Id  
{% endhighlight %}

If I run that on my database where I've generated 150,000 users and 40,000 groups this query takes 120ms. Let's say that's unacceptable and we've done everything we can to index the tables and still can't get the speed we need. The query above can be turned in to an indexed view by doing the following...

{% highlight sql %}
CREATE VIEW UsersAndGroups WITH SCHEMABINDING
AS
SELECT
	Users.Id AS UserId,
	Users.UserName,
	GroupTypes.Id AS GroupTypeId,
	GroupTypes.Name AS GroupType,
	Groups.Id AS GroupId,
	Groups.Name AS [Group]
FROM
	dbo.Users
	INNER JOIN dbo.UserGroups ON UserGroups.UserID = Users.Id
	INNER JOIN dbo.Groups ON UserGroups.GroupId = Groups.Id
	INNER JOIN dbo.GroupTypes ON Groups.TypeId = GroupTypes.Id  
GO
CREATE UNIQUE CLUSTERED INDEX ixUsersAndGroups ON UsersAndGroups(UserId,GroupTypeId,GroupId)
{% endhighlight %}

You can see it's pretty much an ordinary view with 2 exceptions.

1. The view name has “WITH SCEMABINDING” after it. This locks the underlying tables preventing schema changes being made to them that would affect the view. For example no fields in the underlying tables could be renamed or dropped if the view is referencing them. 
1. After the view is created a clustered index is then added to it. This is the line that makes it an index view and causes SQL to materialize the view to disk and maintain it from then on in.

If you now run

{% highlight sql %}
SELECT * FROM UsersAndGroups
{% endhighlight %}

No change? If you look at the query plan you will see why....<a href="/content/images/WPImport/2012/05/qplan.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="qplan" border="0" alt="qplan" src="/content/images/WPImport/2012/05/qplan_thumb.png" width="685" height="274"></a>

It’s still querying the underlying tables and the query plan is the same as it was when we ran the query outside of the indexed view. To get he query to use the indexed view table and not touch the underlying tables we need to add the NOEXPAND hint like this…

{% highlight sql %}
SELECT * FROM UsersAndGroups WITH (NOEXPAND)
{% endhighlight %}

On my machine the query then goes from about 120ms to 25ms, the query plan then looks like this ....

<a href="/content/images/WPImport/2012/05/dsf.png"><img style="background-image: none; border-right-width: 0px; padding-left: 0px; padding-right: 0px; display: inline; border-top-width: 0px; border-bottom-width: 0px; border-left-width: 0px; padding-top: 0px" title="dsf" border="0" alt="dsf" src="/content/images/WPImport/2012/05/dsf_thumb.png" width="376" height="101"></a> 

You can see we are now pulling that data straight out of the indexed view and not touching any other tables giving a massive speed increase.

You could quite easily get carried away creating these everywhere and speeding up your read queries, however remember that there is an overhead to maintaining them. Each time you change data in a table that is used for an index view SQL Server has to also update the indexed view table, so make sure you give it a bit of thought before creating hundreds of them.