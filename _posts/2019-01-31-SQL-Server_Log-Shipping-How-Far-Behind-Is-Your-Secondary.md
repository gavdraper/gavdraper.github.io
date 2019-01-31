---
layout: post
title: How To Check How Far Behind Your SQL Server Log Shipping Secondary Is
date: '2019-01-31 07:56:13'
tags: backup-and-restore hadr
---
Log shipping is one of the simplest and most bulletproof methods to get SQL Server to replicate data to a different server/location. For the most part, you set it up and don't need to touch it again, it just works. Out of the box the agent jobs SQL Server sets up for this generates alerts when a backup/restore hasn't run for a period of time notifying you that there is a problem. 

One thing you don't get however is any nice way to see how up to date each of your databases are on the secondary, sure you can look at the last restore time but if we're running behind it might have restored a log file 1 minute ago but that file could be 4 hours old. The easiest way I can find to tell how up to date the secondary is is to look at the datetime stamp in the filename of the last restored log file. The log backup filenames SQL Server uses for the out of the box log shipping jobs look like this...

>   MyDatabase_20190131080700.trn

With the datetime stamp in the filename being the time the backup was taken. We can see what files have been restored in the MSDB database in a table called backupmediafamily, with a few joins we also can get the restore time and the database name. To really make it simple to read we can also strip out the DATETIME from the filename and add some RPO thresholds to give us a quick status...

To make this process a bit easier I wrote the following script...

{% highlight sql %}
DECLARE @LowRPOWarning INT = 5
DECLARE @MediumRPOWarning INT = 10
DECLARE @HighRPOWarning INT = 15

;WITH LastRestores AS
(
SELECT
    [d].[name] [Database],
    bmf.physical_device_name [LastFileRestored],
    CONVERT(DATETIME,
        STUFF(STUFF(STUFF(SUBSTRING(physical_device_name,LEN([physical_device_name])-17,14),13,0,':'),11,0,':'),9,0,' ') 
    ) LastFileRestoredCreatedTime,
    r.restore_date [DateRestored],        
    RowNum = ROW_NUMBER() OVER (PARTITION BY d.Name ORDER BY r.[restore_date] DESC)
FROM master.sys.databases d
    INNER JOIN msdb.dbo.[restorehistory] r ON r.[destination_database_name] = d.Name
    INNER JOIN msdb..backupset bs ON [r].[backup_set_id] = [bs].[backup_set_id]
    INNER JOIN msdb..backupmediafamily bmf ON [bs].[media_set_id] = [bmf].[media_set_id] 
)
SELECT 
     CASE WHEN DATEDIFF(MINUTE,LastFileRestoredCreatedTime,GETDATE()) > @HighRPOWarning THEN 'RPO High Warning!'
        WHEN DATEDIFF(MINUTE,LastFileRestoredCreatedTime,GETDATE()) > @MediumRPOWarning THEN 'RPO Medium Warning!'
        WHEN DATEDIFF(MINUTE,LastFileRestoredCreatedTime,GETDATE()) > @LowRPOWarning THEN 'RPO Low Warning!'
        ELSE 'RPO Good'
     END [Status],
    [Database],
    [LastFileRestored],
    [LastFileRestoredCreatedTime],
    [DateRestored]
FROM [LastRestores]
WHERE [RowNum] = 1
{% endhighlight %}

It's a bit scrappy with the string manipulation and makes some assumptions about the filenames of your log backups being the same as the native built-in log ship solution (If these are different you may need to make some minor changes). At the top there are 3 defined RPO thresholds that if the last restored file time falls behind the status field will start to show warnings. From here you could easily setup custom alerts in SQL Server or your monitoring tool of choice to sound alarms when things fall behind.

On my demo server the results look like this...

![Log Ship Status Results]({{site.url}}/content/images/2019-Log-Ship-Status\results.PNG)

Do you have any other ways you use to check this information? I'd be interested to hear about alternatives.