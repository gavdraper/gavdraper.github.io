---
layout: post
title: SQL Server Using Central Management Servers
date: '2018-05-14 22:12:24'
---

Central Management Servcers give us a way to manage a collection of SQL Servers as one, a query executed against a Central Management Server will run against every server in the group.

You can designate any SQL Server instance to be a Central Management Server by following this process in SQL Server Management Studio...

First under registered servers we create a new Central Management Server and specify and instance for it to run on.

![access registration wizard]({{site.url}}/content/images/2018-schema-locks/register.png)
![register new central server]({{site.url}}/content/images/2018-schema-locks/register-connection.PNG)

The optionally we create a group for our server registration to live under. I like to do this to allow for seperate groups later so we can run our queries on groups of servers rather than all of them that are managed on the Central Management server.

![access server group wizard]({{site.url}}/content/images/2018-schema-locks/server-group.png)
![register new server group]({{site.url}}/content/images/2018-schema-locks/sales-group.PNG)

Then we add our server registrations to our group...

![access server wizard]({{site.url}}/content/images/2018-schema-locks/server-registration.png)
![register new server]({{site.url}}/content/images/2018-schema-locks/sales-server.PNG)

One thing to note here is SQL Server will stop you from registering the Central Management Server instance as a server on the central management instance... 

![registration error]({{site.url}}/content/images/2018-schema-locks/failed-instance.PNG)

If you need to do this you can get round this by changing the servername so it doesnt match the one you set the central server on by either using it's IP address or fully qualifying it or using a different DNS entry, basically anything that will cause the built in checks string comparission to pass...

![registration success]({{site.url}}/content/images/2018-schema-locks/ip-address.PNG)

Once you've setup a few servers you can start to issue queries against them as a group...

![new group query]({{site.url}}/content/images/2018-schema-locks/group-query.png)

The results will come back in a single result like that have been UNION ALL'd accross servers with an additional field prefixed which is the name of the server registration the row has come from..

This can be really useful for management of multiple servers. Suppoose I have a script that lists all databases that have overdue backups, I can run that script once on server group rather than having to connect to each server individually. As an example run 

{% highlight sql %}
SELECT @@ServerName
{% endhighlight %}

![results]({{site.url}}/content/images/2018-schema-locks/results.PNG)