---
layout: post
title: SQL Server How to Check What Settings Are Set On Active Sessions
date: '2018-06-01 08:16:21'
tags: profiling
---
SQL Server has a number of settings set on a session that can influence the behaviour or queries.  When debugging issues, it's often useful to be able
to get a list of all user sessions and their current settings to check nothing specific to the session is causing odd behaviour. The sys.dm_exec_sessions
dynamic management view has a wealth of information on this. As well as containing all sorts of counters like CPU, Reads, Transaction counts etc is also lists
all the settings that are set on each session.

Different settings will also cause different query plans to be used so if 2 users are running the same query with the same parameters
where one is slow and one is fast it's often worth checking their session settings.

The below query will bring back all active sessions and the settings they have set on them.

{% highlight sql %}
SELECT
   login_name LoginName,
   host_name HostName,
   login_time LoginTime,
   DB_NAME(database_id) DatabaseName,
   program_name ProgramName,
   [status] Status,
   text_size,
   language,
   date_format,
   date_first,
   quoted_identifier,
   arithabort,
   ansi_null_dflt_on,
   ansi_defaults,
   ansi_warnings,
   ansi_padding,
   ansi_nulls,
   concat_null_yields_null
FROM
   sys.dm_exec_sessions
WHERE
   is_user_process = 1
{% endhighlight %}

From here we run queries that will highlight things like different settings used on a given Program/Databasename combination which could cause issues. For example if application A has 10 users connected with a date format of DMY and 1 user with MDY then if not handled correctly by the application you can get very mixed results. The following query can highlight some warnings around applications connecting with multiple date settings...

{% highlight sql %}
SELECT
   s1.program_name ProgramName,
   DB_NAME(s1.database_id) DatabaseName,
   s1.language,
   s1.date_format,
   s1.date_first,
   COUNT(*) Cnt
FROM
   sys.dm_exec_sessions s1
   CROSS APPLY(
      SELECT TOP 1 session_id FROM sys.dm_exec_sessions s2
      WHERE
         s1.program_name = s2.program_name
         AND s1.database_id = s2.database_id
         AND(
            s1.language <> s2.language OR
            s1.date_format <> s2.date_format OR
            s1.date_first <> s2.date_first
         )
   ) s2
WHERE
   s1.is_user_process = 1
   AND s2.session_id IS NOT NULL
   AND s1.program_name NOT LIKE 'Microsoft SQL Server Management Studio%' /*Ignore SSMS*/
GROUP BY
   s1.program_name,
   DB_NAME(s1.database_id),
   s1.language,
   s1.date_format,
   s1.date_first
ORDER BY ProgramName, DB_NAME(s1.database_id)
{% endhighlight %}

If you want to see this in action comment out the last where clause to stop it from ignoring SSMS. Open two tabs in SSMS connect them to the same database and in tab 1 run this...

{% highlight sql %}
SET DATEFIRST 1
{% endhighlight %}

Then in tab 2...

{% highlight sql %}
SET DATEFIRST 2
{% endhighlight%}

Now run the query to highlight the differences.

![Highlighting different session settings]({{site.url}}/content/images/2018-session-settings/differences.PNG)

If you're using a version of SQL Server before 2012 then dm_exec_sessions will not have the database_id. If you want to get it you'll have to go through dm_exec_requests...

{% highlight sql %}
SELECT
   dm_exec_sessions.program_name,
   DB_NAME(dm_exec_requests.database_id) [database]
FROM
   sys.dm_exec_sessions
   inner join sys.dm_exec_requests on dm_exec_requests.session_id = dm_exec_sessions.session_id
WHERE
   is_user_process = 1
{% endhighlight %}