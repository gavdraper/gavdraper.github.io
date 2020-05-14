---
layout: post
title: How To Check How Far Behind Your SQL Server Log Shipping Secondary Is
date: '2019-01-31 07:56:13'
tags: backup-and-restore hadr
featured_image: '/images/hero-images/typewriter.jpg'
---
Log shipping is one of the simplest and most bulletproof methods to get SQL Server to replicate data to a different server/location. For the most part, you set it up and don't need to touch it again, it just works. Out of the box the agent jobs SQL Server sets up for this generates alerts when a backup/restore hasn't run for a period of time notifying you that there is a problem. 

One thing you don't get however is any nice way to see how up to date each of your databases are on the secondary. With a fairly simple query we can take the database name, last restored time and the backup time of the file we're restoring to give some useful information.

To make this even more interesting we can add some RPO thresholds to derive a status field...

{% highlight sql %}
DECLARE @LowRPOWarning INT = 5
DECLARE @MediumRPOWarning INT = 10
DECLARE @HighRPOWarning INT = 15

;WITH LastRestores AS
(
SELECT
    [d].[name] [Database],
    bmf.physical_device_name [LastFileRestored],
    bs.backup_start_date LastFileRestoredCreatedTime,
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

At the top there are 3 defined RPO thresholds that if the last restored file time falls behind the status field will start to show warnings. From here you could easily setup custom alerts in SQL Server or your monitoring tool of choice to sound alarms when things fall behind.

On my demo server the results look like this...

![Log Ship Status Results]({{site.url}}/content/images/2019-Log-Ship-Status\results.PNG)

Do you have any other ways you use to check this information? I'd be interested to hear about alternatives.

Edit : Thanks to LondonDBA in the comments for pointing out the backup_start_date field in the backupset table, which is a much cleaner option to the string manipulation on the filename that I was originally doing.