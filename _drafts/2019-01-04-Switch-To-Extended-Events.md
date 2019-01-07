---
layout: post
title: Profiler Is Dead, Long Live Extended Events
date: '2019-01-03 09:34:01'
tags: tsql maintenance
---
OK, so profiler isnt actually dead, it has however been deprecated since about 2012 and has had no new features since then, Extended Events on the other hand is getting new events and fields added all the time. And Yet... And yet for the most part people are still using profiler and ignoring Extended Events. In this post I'm going to go over a couple of common profiler use cases and show how it's actually easier to use in Extended Events once you've got your head round it. 

I'm not going to go into detail on extended events, if you want more then Jonathan Kehayias has some great posts on it here [https://www.sqlskills.com/blogs/jonathan/category/extended-events/](https://www.sqlskills.com/blogs/jonathan/category/extended-events/).

Lets take 3 common profiles that I used to use before switching to extended events

1) Show RPC Completed queries from user X
2) Show RPC Completed queries over x seconds
3) Show queries with memory grants > 1GB
4) Show deadlock graphs

Run the following TSQL snippets to create Extended Event sessions for the above 4 sessions.

Queries From User X
------
{% highlight sql %}
CREATE EVENT SESSION [RpcAndBatchCompletedForUserX] ON SERVER 
ADD EVENT sqlserver.rpc_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[sqlserver].[equal_i_sql_unicode_string]([sqlserver].[server_principal_name],N'domain\user') AND 
		[sqlserver].[not_equal_i_sql_unicode_string]([sqlserver].[database_name],N'master') AND 
		[sqlserver].[is_system]<>(1)
	)
),
ADD EVENT sqlserver.sp_statement_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[sqlserver].[equal_i_sql_unicode_string]([sqlserver].[server_principal_name],N'domain\user') AND 
		[sqlserver].[not_equal_i_sql_unicode_string]([sqlserver].[database_name],N'master') AND 
		[sqlserver].[is_system]<>(1)
	)
),
ADD EVENT sqlserver.sql_batch_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[sqlserver].[equal_i_sql_unicode_string]([sqlserver].[server_principal_name],N'domain\user') AND 
		[sqlserver].[not_equal_i_sql_unicode_string]([sqlserver].[database_name],N'master') AND 
		[sqlserver].[is_system]<>(1)
	)
)
WITH (MAX_MEMORY=4096 KB)
GO
{% endhighlight %}

### Queries Over X Seconds ###
{% highlight sql %}
CREATE EVENT SESSION [RpcAndBatchCompletedOverXSeconds] ON SERVER 
ADD EVENT sqlserver.rpc_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[package0].[not_equal_boolean]([sqlserver].[is_system],(1)) AND 
		[duration]>(10000000)
	)
),
ADD EVENT sqlserver.sp_statement_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[package0].[not_equal_boolean]([sqlserver].[is_system],(1)) AND 
		[duration]>(10000000)
	)
),
ADD EVENT sqlserver.sql_batch_completed
(
    ACTION
	(
		sqlserver.database_id,
		sqlserver.database_name,
		sqlserver.nt_username,
		sqlserver.server_principal_name,
		sqlserver.session_id
	)
    WHERE 
	(
		[package0].[not_equal_boolean]([sqlserver].[is_system],(1)) AND 
		[duration]>(10000000)
	)
)
WITH (MAX_MEMORY=4096 KB)
GO
{% endhighlight %}

### Queries With Memory Grants Over 1GB ###
{% highlight sql %}
CREATE EVENT SESSION MemoryUseageAbove10mb ON SERVER 
ADD EVENT sqlserver.query_memory_grant_usage --Ignores querys below 5mb 
(
    ACTION(sqlserver.sql_text)
    WHERE ([granted_memory_kb] > 1000000) --Above 1GB
)
WITH (MAX_MEMORY=4096 KB)
{% endhighlight %}

### Deadlocks With Graphs ###
{% highlight sql %}
CREATE EVENT SESSION DeadLocks ON SERVER 
ADD EVENT sqlserver.lock_deadlock
(
    ACTION(sqlserver.sql_text)
),
ADD EVENT sqlserver.lock_deadlock_chain
(
    ACTION(sqlserver.sql_text)
),
ADD EVENT sqlserver.xml_deadlock_report
WITH (MAX_MEMORY=4096 KB)
{% endhighlight %}


All of the above can be done through the extended event wizard in SSMS but I like to save these scripts and edit them when I need to create new Extended Event sessions to save time. If you're not sure what is available a good place to start is the wizard and from there you can click the script button to script the whole thing and tidy it up. Some of the above may look a bit lengthy at first glance but if you break it down it's really just a list of...

1) Events that you want to capture
1) Actions that you want to capture on those events (Basically fields of data available on those events)
1) Filters on the events e.g Duration > X
1) Targets (Not used in the above), targets define where to store the data you collect.
1) Settings for things like max memory, retention, max size, startup state etc...

Once you've run the above scripts you'll see the 4 Extended Events in SSMS...

![Extended Event List In SSMS]({{site.url}}/content/images/2019-Extended-Events\ssms-ee-list.PNG)

Notice they all have a stopped icon on them as none of them are currently running, to start one right click and select Start Session. You can then right click and "Watch Live Data" to see the events in real time.

![Extended Event Live Data]({{site.url}}/content/images/2019-Extended-Events\ssms-ee-live-data.PNG)

Once you're done, then right click the extended event session again and click stop. Whilst in theory Extended Events are lighter weight than Profiles were they are not free and depeneding on the Events and Actions you're capturing you can still effect the performance of the server.

The sessions I've created above are all currently scripted not to store the session data anywhere so at the minute all we have is the Live Data View. You can configure storage targets to persist this data but I'm not going to go into that here. If you want more information on that I recommend the Microsoft Docs here https://docs.microsoft.com/en-us/sql/database-engine/sql-server-extended-events-targets?view=sql-server-2014 

We've covered the TSQL approach above but Extended Events also has a nice Wizard built in to SSMS for setting up new sessions, this same wizard allows you to browse and search through all the available Events and Actions which can be a great way to find what you can capture. At the end of the Wizard you can either click to create the session or even better click the script button to get the TSQL for it so you can save it away for future. 

## Why Is this Better Than Profiler ##
# It's not deprecated - Profiler could go away with any next major version of SQL Server.
# One tool SSMS - No more having to drop in and out of SSMS to Profiler
# Easy to create and filter with TSQL - I find with Extended Events I'm must more likely to script my sessions or just leave them stopped on the server ready to run again. It always felt a bit clunky to say templates in profiler so I rarely did it.
# WAY WAY more events! - Profiler hasnt had any new events added to it since SQL Server 2008, Extended Events on the other hand now has massively more events than Profiler that you can watch for with more getting added with most releases.

