---
layout: post
title: SQL Server Why Is My Transaction Log Getting So Large
date: '2018-05-07 23:09:11'
---
The Transaction log growing is completely normal however there are situations where a transaction log can get to a state where it wont stop growing which if left unmonitored can fill drives and bring down servers.

## Checkpoints ##
To understand how this works you first need to know what a checkpoint is. As changes are made in SQL Server they are written to the log file sequentially but the actual changes to the data pages are held in memory until a check point occurs. A checkpoint is the process that flushes all the changes in memory to disk then writes LSN of the last log record to the database boot page essentially marking everything up to that point as written to the data files on disk. If a server crashes before a checkpoint it is recovered back to a consistent state on startup by replaying the transactions in the transaction log that occurred after the last checkpoint.

## Causes Of Constant Growth ##
Normally when a transaction log grows out of control it's down to 1 of a couple of reasons...

1. You're in full recovery and haven't run a log backup lately.
1. There is a long running transaction thats either still running.
1. Someone has started a transaction in a tool like SSMS and not ended it. 
1. You have some kind of replication, mirroring or availability group in place and the secondary has not yet received the logs. For example if a read only secondary in an availability group goes offline the log file on the primary will not truncate until the secondary comes back online and catches up.

You can find out more about why you r transaction log isn't being reused by looking at the log_reuse_wait_desc in the following query...

{% highlight sql %}
SELECT 
   [name],
   log_reuse_wait, 
   log_reuse_wait_desc
FROM 
   sys.databases
{% endhighlight %}

The below result shows that we have a log file that's unable to truncate due to activate transactions...

![Log Waits]({{site.url}}/content/images/2018-log-growth\log-reuse-wait.PNG)

Or when the log file hasnt been backed since the last checkpoint...

![Log Backup Waits]({{site.url}}/content/images/2018-log-growth\log-backup-wait.PNG)

Growth is normal it's when SQL Server truncates the log that can vary. When SQL Server truncates the log as part of it's checkpoint process it wont actually shrink the log it instead marks the space as reusable and will write over that space again as it needs to. The process for when this happens depends on your recovery model.

## When Do Log Truncates Occur ##
Assuming none of the above conditions on why a truncate wont occur are true, under normal conditions they will happen when...

### Full/Bulk Recovery Model ###
The log is kept until it is backed up at which point if there has been a checkpoint since the last truncation it will run another truncate.

### Simple Recovery Model ###
The log is truncated after every checkpoint.

## Physically Shrinking The Log File ##
Under normal circumstances you should never need or want to physically shrink the log to free up disk space as it will inevitable just grow again as SQL Server needs it to and this can be a costly operation. The only situation where it can be ok to shrink the file is if it has grown due to one of the above issues and is now of a size that is a lot bigger than it needs to be for normal operating conditions.

Before you can physically shrink a log file you need there to be some space in it that can be taken back so shrinking a log file that is still growing out of control for one of the above reasons most likely wont reclaim any space. However once you have fixed the cause of your log growth you can shrink the file with the following command...

{% highlight sql %}
 DECLARE @LogFileName NVARCHAR(100)
 DECLARE @NewFileSizeInMB INT = 200
 SELECT 
   @LogFileName = [Name]
 FROM 
   sys.master_files 
 WHERE 
   type_desc = 'LOG' AND
   database_id = DB_ID()

DBCC SHRINKFILE(@LogFileName, @NewFileSizeInMb)
{% endhighlight %}