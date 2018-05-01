---
layout: post
title: SQL Server Dedicated Admin Connection and Why You NEED It!
date: '2018-05-02 22:13:42'
---
DAC or Dedicated Admin Connection is one of those things you don't know you need until it's too late.

## What is It? ##
The dedicated admin connection allows a single user to connect and run queries on it's own singe reserved thread. By default the DAC only allows local connections e.g connections made from the server the SQL instance is running on. In order to allow remote connections to the DAC run the following SQL...

{% highlight sql %}
sp_configure 'remote admin connections', 1;  
GO  
RECONFIGURE;  
GO  
{% endhighlight %}

If you don't have access to get onto the actual servers your instance run on either locally or via RDP then you should make sure you enable remote admin connections or you wont be able to use them.

## Why You Need It ##
At some point you're going to experience a SQL Server that wont allow any more connections or even respond to existing connections, this is usually caused by the server getting backed up under heavy load for example a backlog of RESOURCE SEMAPHORE waits (memory waits) will eventually cause a server to not respond.

At this point phones are going to be going crazy with people demanding you get that server running again. You fire up SSMS and can't connect or perform and kind of diagnosis at this point with the DAC enabled your only real option is to restart the instance with no real chance to troubleshoot.

## Using the DAC ##
So your server isn't responding and you're unable to connect to it normally, assuming you've enabled the DAC you can connect to it via SSMS by prefixing the instance name with ":admin". Remember only a single sys admin user can connect to the DAC at a time so this is not a connection you want to be using to run your day to day scripts under. Also this connection is single threaded so there will be no parallel plans at play here and any commands that require parallel operations will fail DBCC CHECKDB, BACKUP etc....

Once you're in you can then run your diagnosis queries like sp_who2, whoisactive etc.. and any other scripts you have to find out what's bringing the server to it's knees. Hopefully at this point you can fix the issue and bring the server back without needing to restart the instance, maybe it just requires killing Bob's offending session he's left a transaction open in and went home for the night. Even if you can't fix it then and need to do a restart hopefully you at least had time to gather some information on what was happening to prevent it happening again.
