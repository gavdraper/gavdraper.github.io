---
layout: post
title: SQL Server Script To Check Your Backup RPO Status
date: '2019-02-13 08:31:22'
tags: backup-and-restore hadr
featured_image: '/images/hero-images/backup.jpg'
---
I recently wanted a script to tell me that for every database on a given server

1. What levels of backups I have
2. How many files would need to be restored to get to the most recent backup state. 
3. The size of all the files I'd need to restore
4. How up to date this process could get me

For example, if we have a database with the following backup schedule

* 23:00 - Full Backup
* 06:00, 12:00, 18:00 - Differential Backups
* Every 15 Minutes Log Backup

I wanted to know given the backups I have at the time I run this script how up to date I could restore to and how many files would be involved in the restore. The output of this new procedure looks like this...

![Backup Status Results]({{site.url}}/content/images/2019-BackupStatus/Intro.JPG)

This information will only show child items after the last parent item in the chain, for example

* Only differential backups created after the last full backup
* Only logs backups after the last differential or if there is no differential it will fall back to after the last full
  
Given this information we can see that to get up to date we need to

1. Restore 1 Full Backup
1. Restore 1 Differential Backup
1. Restore 1 Log Backup

Doing this will get us to within about 15 minutes of where the database currently is, we can see this by the fact our most recent log backup is 15 minutes old.

I'm going to walk through an example of creating some backups and restoring them with this logic. If you want to skip ahead and just get the backup status script then it's at the bottom of this post. 

I've created a database called RandomDB, It has no backup history and none are scheduled. The output of sp_BackupStatus now looks like this...

![No Backups]({{site.url}}/content/images/2019-BackupStatus/1-No-Backups.JPG)

If we then run a full backup then look at sp_BackupStatus again...

{% highlight sql %}
BACKUP DATABASE RandomDB TO DISK='C:\Temp\RandomDb.bak' WITH COMPRESSION
GO
sp_BackupStatus
{% endhighlight %}

![Full Backup]({{site.url}}/content/images/2019-BackupStatus/2-FullBackup.JPG)

Then let's run a few differential backups mixed in with a load of transaction log backups (Pauses between just to make the information returned from sp_BackupStatus a little clearer)...

{% highlight sql %}
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog1.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog2.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog3.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog4.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog5.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog6.bak' WITH COMPRESSION
WAITFOR DELAY '00:00:02'
BACKUP DATABASE RandomDb TO DISK='C:\Temp\RandomDb.bak' WITH DIFFERENTIAL, COMPRESSION
WAITFOR DELAY '00:00:02'
CREATE TABLE RandomDb.dbo.Test (Id INT) 
BACKUP LOG RandomDb TO DISK='C:\Temp\RandomDBLog7.bak' WITH COMPRESSION
GO
EXEC sp_BackupStatus
{% endhighlight %}

![Full Backup]({{site.url}}/content/images/2019-BackupStatus/3-Diffs.JPG)

Even though we took 7 log backs this script is letting us know that to get to the latest possible version we only need to 1 full, 1 differential and 1 log...

{% highlight sql %}
RESTORE DATABASE RandomDB2 FROM DISK = 'C:\Temp\RandomDb.bak' 
    WITH NORECOVERY, FILE = 1 --Full
RESTORE DATABASE RandomDB2 FROM DISK = 'C:\Temp\RandomDb.bak' 
    WITH NORECOVERY, FILE = 2 --Differential
RESTORE LOG RandomDB2 FROM DISK = 'C:\Temp\RandomDBLog7.bak'
{% endhighlight %}

To confirm this is up to date we can also check our Test table that we created right before the last log backup exists...

![Restored]({{site.url}}/content/images/2019-BackupStatus/restored.JPG)

One of the things I really like about this as at any point in the day I can run sp_BackupStatus and quickly see the total sizes of the backup files I'd need to restore to get to the latest possible point in time, This can also give a good indication as to how long this process would take.

And now for the sp_BackupStatus script...

{% highlight sql %}
/*
You may also want to create this index, I had some issues querying the 
below tables without it.
USE [msdb]
GO
CREATE NONCLUSTERED INDEX ndx_backupset_type_db_start
ON [dbo].[backupset] ([type],[database_name],[backup_start_date])
INCLUDE ([backup_size])
GO
*/
USE master
GO
DROP PROCEDURE IF EXISTS sp_BackupStatus
GO
CREATE PROCEDURE sp_BackupStatus
AS
SELECT
    DB_NAME(dbs.database_id) AS [Database],
    dbs.recovery_model_desc AS [RecoveryModel],
    ISNULL(CAST(DATEDIFF(MINUTE,LastFull.backup_start_date,GETDATE()) AS VARCHAR(20)) +' Minutes','NO FULL BACKUP!') [FullAge],
    ISNULL(CAST(LastFull.backup_size AS VARCHAR(20)) + 'MB','0MB') [FullSize],
    ISNULL(CAST(DATEDIFF(MINUTE,DifferentialSummary.backup_start_date,GETDATE()) AS VARCHAR(20)) + ' Minutes','Never') [LastDiffAge],
    ISNULL(CAST(DifferentialSummary.BackupSizeSinceFull AS VARCHAR(20)) + 'MB','0MB') [DiffRestoreSize],
    DifferentialSummary.FileCount [DiffFileCount],
    ISNULL(CAST(DATEDIFF(MINUTE,LogSummary.backup_start_date,GETDATE()) AS VARCHAR(20)) + ' Minutes','Never') [LastLogAge],
    ISNULL(CAST(LogSummary.BackupSizeSinceDiffOrFull AS VARCHAR(20)) + 'MB','0MB') [LogRestoreSize],
    LogSummary.FileCount [LogFileCount],
    ISNULL(CAST(ISNULL(DifferentialSummary.BackupSizeSinceFull,0) +
        ISNULL(LogSummary.BackupSizeSinceDiffOrFull,0) +
        LastFull.backup_size  AS VARCHAR(20)) + 'MB','0MB') [TotalSizeToRestore]
FROM 
    sys.databases dbs
    OUTER APPLY(
        SELECT TOP 1 
            backupset.backup_start_date, 
            CAST((backupset.backup_size/1024)/1024 AS INT) backup_size, 
            bmf.physical_device_name
        FROM 
            msdb.dbo.backupset 
            LEFT JOIN msdb.dbo.backupmediafamily bmf ON bmf.media_set_id = backupset.media_set_id
        WHERE 
            backupset.database_name = DB_NAME(dbs.database_id) AND 
            backupset.[Type] = 'D' 
        ORDER BY backup_start_date DESC
    ) LastFull
    OUTER APPLY(
        SELECT TOP 1 
            backupset.backup_start_date, 
            backupset.backup_size, 
            bmf.physical_device_name,
            DiffAggregations.BackupSizeSinceFull,
            DiffAggregations.FileCount
        FROM 
            msdb.dbo.backupset 
            LEFT JOIN msdb.dbo.backupmediafamily bmf ON bmf.media_set_id = backupset.media_set_id
            OUTER APPLY(
                SELECT
                    COUNT(*) AS FileCount,
                    CAST((SUM(aggr.backup_size)/1024)/1024 AS INT) BackupSizeSinceFull
                FROM
                    msdb.dbo.backupset aggr
                WHERE   
                    aggr.database_name = DB_NAME(dbs.database_id) AND
                    aggr.backup_start_date >= LastFull.backup_start_date AND
                    aggr.[Type] = 'I'  

            ) DiffAggregations            
        WHERE 
            backupset.database_name = DB_NAME(dbs.database_id) AND 
            backupset.[Type] = 'I'  AND
            backupset.backup_start_date >= LastFull.backup_start_date
        ORDER BY backup_start_date DESC
    ) DifferentialSummary    
    OUTER APPLY(
        SELECT TOP 1 
            backupset.backup_start_date, 
            backupset.backup_size, 
            bmf.physical_device_name,
            LogAggregations.BackupSizeSinceDiffOrFull,
            LogAggregations.FileCount
        FROM 
            msdb.dbo.backupset 
            LEFT JOIN msdb.dbo.backupmediafamily bmf ON bmf.media_set_id = backupset.media_set_id
            OUTER APPLY(
                SELECT
                    COUNT(*) FileCount,
                    CAST((SUM(aggr.backup_size)/1024)/1024 AS INT) BackupSizeSinceDiffOrFull
                FROM
                    msdb.dbo.backupset aggr
                WHERE   
                    aggr.database_name = DB_NAME(dbs.database_id) AND
                    aggr.backup_start_date >= ISNULL(DifferentialSummary.backup_start_date ,LastFull.backup_start_date) AND
                    aggr.[type] = 'L'
            ) LogAggregations
        WHERE 
            backupset.database_name = DB_NAME(dbs.database_id) AND 
            backupset.backup_start_date >= ISNULL(DifferentialSummary.backup_start_date ,LastFull.backup_start_date) AND
            backupset.[Type] = 'L'  
        ORDER BY backup_start_date DESC
    ) LogSummary        

WHERE   
    DB_NAME(dbs.database_id) NOT IN ('master','model','msdb','tempdb')
ORDER BY    
    [Database]
{% endhighlight %}

> Disclaimer : This was cobbled together in an evening and probably has all sorts of bugs. I've put a version of it in my scripts repository on GitHub (https://github.com/gavdraper/GavinScripts), feel free to submit issues and pull requests there.