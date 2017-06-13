---
layout: post
title: SQL Server 2017 Graph Data Features In Action
date: '2017-06-12 20:23:18'
---

The [Wikipedia Graph Database](https://en.wikipedia.org/wiki/Graph_database) page has the following definition...

> In computing, a graph database is a database that uses graph structures for semantic queries with nodes, edges and properties to represent and store data. A key concept of the system is the graph (or edge or relationship), which directly relates data items in the store. The relationships allow data in the store to be linked together directly, and in many cases retrieved with one operation.

Let's take a look at a sample use case for a Graph Database to learn wha this means and where it's useful...

Imagine a system like facebook where friends and friends of friends content can appear in your feed. Representing the friend of friend hierarchy is quite difficult in a relational database, especially when you consider you may then want to go down further levels to recommend friends of friends of friends.... Let's create a graph schema for this using the new graph features in SQL Server 2017

{% highlight sql %}
CREATE TABLE dbo.Person (
  Id INTEGER PRIMARY KEY, 
  FirstName NVARCHAR(100),
  LastName NVARCHAR(100)
) AS NODE;

CREATE TABLE dbo.Friend AS EDGE
{% endhighlight %}

Notice the AS NODE on the Person Table, that's the new syntax that marks this table as a node in our graph. Then note the AS EDGE on the FriendTable, Edge Tables are used to create links between Nodes.

Let's then create our list of people who at this point have no relationship between them...

{% highlight sql %}
INSERT INTO dbo.Person 
VALUES 
    (1,'Claire','Template'),
    (2, 'Luke','Cage'),
    (3,'Jessie','Jones'),
    (4,'Tony','Stark'),
    (5,'Matt','Murdock')
{% endhighlight %}

If we then want to define Friend links between our person entries  we insert node id's into the friend table. Let's imagine we want to link Claire Temple (Id : 1) and Luke Cage (Id : 2) as friends...

{% highlight sql %}
INSERT INTO dbo.Friend 
VALUES 
(
    (SELECT $node_id FROM Person WHERE id = 1), 
    (SELECT $node_id FROM Person WHERE id = 2)
);
{% endhighlight %}

To get the node id we have to look it up in the Person node table with the inner select. Let's then also make Claire a friend of Jessie and make Luke a friend of Matt.

{% highlight sql %}
INSERT INTO dbo.Friend 
VALUES 
((SELECT $node_id FROM Person WHERE id = 1), (SELECT $node_id FROM Person WHERE id = 3)),
((SELECT $node_id FROM Person WHERE id = 2), (SELECT $node_id FROM Person WHERE id = 5));
{% endhighlight %}

In this case we have direct friend links from Claire to Luke and Jessie, then through the link with Luke we have a friend of friend relationship to Matt.

![Friend Graph]({{site.url}}/content/images/2017-graph/graph.PNG)

In this image the circles represent our Node table and the lines the Edge table.

If we want to see all of Claire's friends we can do this...

{% highlight sql %}
SELECT 
    FriendOfPerson.FirstName + ' ' + FriendOfPerson.LastName Friend
FROM 
    Person Person, 
    Person FriendOfPerson, 
    Friend
WHERE 
    MATCH(Person-(Friend)->FriendOfPerson)
    AND person.FirstName='Claire'
{% endhighlight %}

Notice the match syntax here 

> Match(Person-(Friend)->FriendOfPerson)

We're saying for the Person Node follow al Friend edge links to person entires.

![Friends Results]({{site.url}}/content/images/2017-graph/friends.PNG)

We can get a list of Friends of Friends by then matching on friend again...

{% highlight sql %}
SELECT 
    FriendOfFriendOfPerson.FirstName + ' ' + FriendOfFriendOfPerson.LastName FriendOfFriend
FROM 
    Person Person, 
    Person FriendOfPerson, 
    Person FriendOfFriendOfPerson,
    Friend,
    Friend FriendOfFriend
WHERE
    MATCH(Person-(Friend)->FriendOfPerson-(FriendOfFriend)->FriendOfFriendOfPerson)
    AND person.FirstName='Claire'
{% endhighlight %}

![Friends Of Results]({{site.url}}/content/images/2017-graph/friend-of-friend.PNG)

AS you can see the new graph syntax make navigating through different levels of a graph a lot simpler than the many to many representation you'd have if you tried to implement this in a relational database.

Just to flex the graph database features a bit more let's now imagine that we want to list all the friends Claire and Luke have in common...

{% highlight sql %}
SELECT 
    Person.FirstName,
    Person2.FirstName,
    FriendOfPerson.FirstName
FROM 
    Person Person, 
    Person FriendOfPerson, 
    Friend,
    Person Person2, 
    Friend Friend2
WHERE 
    MATCH
    (
        Person-(Friend)->FriendOfPerson<-(Friend2)-Person2
    )
    AND person.FirstName='Claire'
    AND person2.FirstName = 'Luke'
{% endhighlight %}

Notice in this match statement we have the flow going both ways -> and <-. This is saying get me all of Claire's Friends and for each of them get me all their friends where their friends name is luke. The inserts we ran when we created our friend edge records above have no common friends between these two people, try adding a common friend using the insert syntax above and running this query again to see the matches.

In our case we didn't define any fields in our edge table and it exists purely as an edge. You can however add additional fields to these tables to give more information, for example we could store DateOfFriendship in the Friend Edge table to store the date the edge was created. That would allow us to find all Friend connections made in a specific period. Let's clear out our Friend table and reinitialize it with this new field and data...

{% highlight sql %}
TRUNCATE TABLE Friend

ALTER TABLE Friend ADD DateOfFriendShip DATETIME

INSERT INTO dbo.Friend 
VALUES 
    ((SELECT $node_id FROM Person WHERE id = 1), (SELECT $node_id FROM Person WHERE id = 2),'20160101'),
    ((SELECT $node_id FROM Person WHERE id = 1), (SELECT $node_id FROM Person WHERE id = 3),'20140705'),
    ((SELECT $node_id FROM Person WHERE id = 2), (SELECT $node_id FROM Person WHERE id = 5),'20100605');
{% endhighlight %}

So our graph now looks a bit like this...

![Graph with edge data]({{site.url}}/content/images/2017-graph/graph-edge-dates.PNG)

We can then query all Claire's friends made before 2016 by just adding a where filter on that date column like a normal SQL query..

{% highlight sql %}
SELECT 
    FriendOfPerson.FirstName + ' ' + FriendOfPerson.LastName Friend
FROM 
    Person Person, 
    Person FriendOfPerson, 
    Friend
WHERE 
    MATCH(Person-(Friend)->FriendOfPerson)
    AND person.FirstName='Claire'
    AND Friend.DateOfFriendship < '20160101'
{% endhighlight %}

As expected we get one result as Jessie is Claire's only friend made before 2016. 