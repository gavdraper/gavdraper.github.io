---
layout: post
title: SQL Server Save Time Managing Multiple Servers With Central Management Servers
date: '2018-05-15 06:12:24'
tags: central-management-server
---

Central Management Servers give us a way to manage a collection of SQL Servers as one, a query executed against a Central Management Server will run against every server in the group.

You can designate any SQL Server instance to be a Central Management Server by following this process in SQL Server Management Studio...

First under registered servers create a new Central Management Server and specify and instance for it to run on.

![access registration wizard]({{site.url}}/content/images/2018-Central-Management-Server/register.png)

![register new central server]({{site.url}}/content/images/2018-Central-Management-Server/register-connection.PNG)

Then optionally create a group for your server registrations to live under. I like to do this to keep things tidy.

![access server group wizard]({{site.url}}/content/images/2018-Central-Management-Server/server-group.png)

![register new server group]({{site.url}}/content/images/2018-Central-Management-Server/sales-group.PNG)

Then add your server registrations to the new group...

![access server wizard]({{site.url}}/content/images/2018-Central-Management-Server/server-registration.png)

![register new server]({{site.url}}/content/images/2018-Central-Management-Server/sales-server.PNG)

One thing to note here is SQL Server will stop you from registering the Central Management Server instance as a server on itself...

![registration error]({{site.url}}/content/images/2018-Central-Management-Server/failed-instance.PNG)

If you need to do this you can get round it by changing the server name so it doesn't match the one you set on the Central Server registration by something like using it's IP address or fully qualifying it or using a different DNS entry, basically anything that will cause the built in string comparison to pass...

![registration success]({{site.url}}/content/images/2018-Central-Management-Server/ip-address.PNG)

Once you've setup a few servers you can start to issue queries against them as a group...

![new group query]({{site.url}}/content/images/2018-Central-Management-Server/group-query.png)

The results will come back in a single resultset like they have been UNION ALL'd across servers with an additional field prefixed which is the name of the server registration the row has come from..

This can be really useful for management of multiple servers. Suppose you have a script that lists all databases that have overdue backups, You can run that script once on server group rather than having to connect to each server individually. As an example run...

{% highlight sql %}
SELECT @@ServerName
{% endhighlight %}

![results]({{site.url}}/content/images/2018-Central-Management-Server/results.PNG)

This can really save time as you get more and more instances to manage, for example people often have a set of morning check script they run on each instance when they come in to work, using this approach of running each script only once without having to connect to every instance manually can save a lot of time.