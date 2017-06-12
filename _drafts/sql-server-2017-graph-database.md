---
layout: post
title: SQL Server 2017 Graph Database In Action
date: '2017-06-13 08:23:18'
---
Let's imagine a system like facebook where friends and frinds of friends content can appear in your feed. Representing the friend of friend hierachy is quite difficult in a relational database, esecially when you consider you may then want to go down further levels to recomened friends of friends of friends.... Let's create a graph schema for this using the new graph features in SQL Server 2017

{% highlight sql %}
CREATE TABLE dbo.Person (
  Id INTEGER PRIMARY KEY, 
  FirstName NVARCHAR(100),
  LastName NVARCHAR(100)
) AS NODE;

CREATE TABLE dbo.Friend AS EDGE
{% endhighlight %}

Notice the AS NODE on the Person Table, that's the new syntax that makes this table as a node in our graph. Then note the AS EDGE on the FriendTable, this Edge Table is used to create linkes between Nodes.
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

If we then want to define Friend links between our tables we insert node id's into the friend table. Let's imagine we want to link Claire Template (Id : 1) and Luke Cage (Id : 2) as friends...

{% highlight sql %}
INSERT INTO dbo.Friend 
VALUES 
    (
        (SELECT $node_id FROM Person WHERE id = 1), 
        (SELECT $node_id FROM Person WHERE id = 2)
    );
{% endhighlight %}

To get the node id we have to look it up in the Person edge table with the inner select. Let's then also make Claire a friend of Jessie and make Luke a friend of Matt.

{% highlight sql %}
INSERT INTO dbo.Friend VALUES ((SELECT $node_id FROM Person WHERE id = 1), 
       (SELECT $node_id FROM Person WHERE id = 3));

INSERT INTO dbo.Friend VALUES ((SELECT $node_id FROM Person WHERE id = 2), 
       (SELECT $node_id FROM Person WHERE id = 5));
{% endhighlight %}

In this case we have direct friend links from Claire to Luke and Jessie, then through the link with Luke we have a friend of friend relationship to Matt.

If we want to see all of Claires friends we can do this...

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

''' Match(Person-(Friend)->FriendOfPerson)

Here we're saying for the Person Node follow the Friend Edge to all other Person Nodes connected via an edge.

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

AS you can see the new graph syntax make navigating through different levels of a heirachy a lot simpler than the many to many representation you;d have if you tried to implement this in a relational database.
