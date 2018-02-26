---
layout: post
title: How Many VLFs Is Too Many
date: '2018-02-26 07:45:18'
---
Whenever a transaction log file grows the space it is allocated is divided up into virtual log files (VLFs). Depending on the amount the log file is set to grow by the amount of VLFs that are added varies, on instances < SQL Server 2014

* A growth of 0mb to 64mb results in 4 new VLFs
* 64mb - 1gb = 8 VLFs
* Anything larger than 1gb gets 16 new VLFs

On instances 2014 and after

* If growth size < 1/8th of the current total size then add one new VLF

ELSE
* A growth of 0mb to 64mb results in 4 new VLFs
* 64mb - 1gb = 8 VLFs
* Anything larger than 1gb gets 16 new VLFs

Looking at the formula for 2014 you can see they have tried to cut down on the amount of VLFs allocated if your large transaction log is constantly growing in small increments. 

So for example on SQL Server 2008 if you have your log file set to grow at 1mb intervals and it grows 100mb that is going to assign 400 VLFs to that 100mb of new transaction log.

On SQL Server 2014+ if your log file is currently at 3gb with 1mb growth size set and 100mb of space needed it will allocate 1 VLF per growth so rather than the 400 VLFs created before 2014 you will instead get 100 VLFs (Still not ideal).

Let's create a new database and run a simple query to see how many VLFs it's created with...

{% highlight sql %}
CREATE DATABASE VlfDemo
GO
USE VlfDemo
GO

/*Get Count of VLFs*/
SELECT 
   COUNT(l.database_id) AS 'VLFs'
FROM 
   sys.databases s
   CROSS APPLY sys.dm_db_log_info(s.database_id) l
WHERE  
   [name] = 'VlfDemo'
GROUP BY 
   [name], s.database_id
{% endhighlight %}

Depending on SQL version and default settings the numbers may vary slightly but on my system this new database has 4 VLFs. Let's now change the transaction logs auto growth size to 1mb (This is not a good idea in the real world). On my system this  defaults at a 64mb growth size for new databases....

{% highlight sql %}
ALTER DATABASE VlfDemo MODIFY FILE ( NAME = N'VlfDemo_Log',  FILEGROWTH = 1000kb)
{% endhighlight %}

Let's now create a demo table with a char 6000 field, I'm using char to force space to be used in the transaction log...

{% highlight sql %}
CREATE TABLE LogFileGrower
(
   Id INT PRIMARY KEY IDENTITY,
   LongName CHAR(6000)
)
{% endhighlight %}

Now lets insert a load of data to get that transaction log working...

{% highlight sql %}
INSERT INTO LogFileGrower (LongName)
SELECT 
   'HelloWorld'
FROM 
   master.dbo.spt_values s1
   CROSS JOIN master.dbo.spt_values s2
WHERE 
   s2.number < 100
{% endhighlight %}

Firstly you will probably notice this query is quite slow, we're growing the transaction log in increments that are far too small causing constant file growths. On my laptop this took 3 minutes to complete. If we then run the above query to get the count of VLFs again, assuming you're on 2014+ you'll see we now have 7081 VLFs (Even more if you're on < SQL Server 2014).

For reference if you drop the database and recreate but this time set the growth size to 100mb on 2014+ you'll end up with 131 VLFs and on my laptop the same insert went from 3 minutes down to 1 minute (Mainly due to far less auto growths occurring than the amount of VLFs). So we can see from this already that high VLFs will slow down our writes that cause file growths. This can largely be mitigated by setting sensible file growths of even an initial file size for your environment and recovery model.

There are also some other fairly large impacts of large VLF counts. Namely slow backup/restore and recovery times for transaction logs. For example when a SQL Server comes online it  performs a recovery process on each databases that looks for incomplete transactions and rolls them back, this process can be massively slower if your VLF counts have gotten out of hand. 

In answer to how many VLFs is too many, it really depends. I try to aim for < 200 with alarm bells seriously ringing when I see systems in the thousands. At a previous company we had an unplanned power outage on one of our SQL Servers and when it came back up SQL Server took about half hour to bring a database back online that had somehow assigned itself 15,000 VLFs. 

## Lowering the VLF Count
If you have a database with a lot of VLFs firstly you need to correct the cause, this will normally involve changing the auto growth size as we did above. Then to lower the count we need to backup the transaction log if we're in full recovery and run shrinkfile on the log file. 

{% highlight sql %}
BACKUP DATABASE VlfDemo TO DISK='nul'
BACKUP LOG VlfDemo TO DISK='nul'
DBCC SHRINKFILE (N'VlfDemo_log' , 0, TRUNCATEONLY)
{% endhighlight %}

The 'nul' backup location just means don't save anything to disk, I'm just using it to force the database to think it's been backed up. NEVER use this on a realy system. After running this if you run the query to get the count of VLF's again you can see we're back to 4. You may also want to then set the initial filesize of the log back to the value it was before you ran the shrink as that'll save it having to auto grow several times to get back to where it needs to be. This can be done like this...

{% highlight sql %}
/*Set size to 1gb*/
ALTER DATABASE VlfDemo MODIFY FILE ( NAME = N'VlfDemo_Log',  Size = 1000000kb)
{% endhighlight %}

