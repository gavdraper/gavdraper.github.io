---
layout: post
title: Kicking Ass With Backup Jobs
date: '2018-09-03 09:34:01'
---
SQL Maintenance plans have been around for a long time and are commonly used for tasks like backups, CHECKDB, index rebuilds, statistics rebuilds and other cleanup work. The UI for building them is clunky and often slow, it's a bit of a pain to move them between servers and the tasks in them are not the most flexable. So what's the alternative?

## Ola Hallengren's DatabaseBackup ##
Ola Hallengren has a number of scripts one of which is DatabaseBackup (https://ola.hallengren.com/sql-server-backup.html). If you want to follow along you will need to install Ola's scripts from the previous link. Let's look at some examples of using this and see why it's more flexible...

Full backup of all user databases with checksum and remove backups older than 3 days...

{% highlight sql %}
EXECUTE dbo.DatabaseBackup
    @Databases = 'USER_DATABASES',
    @BackupType = 'FULL',
    @CheckSum = 'Y',
    @CleanupTime = 72,
    @Directory = 'D:\DBBackups',
{% endhighlight %}

The above script will create a directory structure inside the directory you give it that looks something like this...

    D:\DBBackups\ServerName\DatabaseName\Full\DBName+DateTime.bak

Another smart thing this script can do is if you are running in full recovery it can change backup type between log and full depending on weather a full backup already exists...

{% highlight sql %}
EXECUTE dbo.DatabaseBackup
    @Databases = 'USER_DATABASES',
    @Directory = 'D:\DBBackups',
    @BackupType = 'LOG',
    @ChangeBackupType = 'Y'
{% endhighlight %}

The above script uses @ChangeBackupType to switch between full and log. It will first check if a full backup exists, if it doesnt it will make a full backup, if it does it will make a log backup, these files will go to...

    D:\DBBackups\ServerName\DatabaseName\Log\DBName+DateTime.log
    D:\DBBackups\ServerName\DatabaseName\Full\DBName+DateTime.bak

Depending on RPO/RTO requirements I typically take all my full backups with Ola's Database backup script from an agent job.

You could quite easily schedule the above to also manage your log backups and that would work well however Brent Ozar's team have fairly recently release sp_AllNightLog.

## Brent Ozar Unlimited's sp_AllNightLog ##
sp_AllNightLog builds ontop of Ola's DatabaseBackup script to really help automate managing database log backups and even restoring them. On my primary I've been using this to setup jobs that will automatically backup my log files for any database I create that is in full recovery. As part of this process sp_AllNightLog created an agent job that polls for new databases and automatically adds them to it's list of databases to backup the logs for.

Like I said before behind the scenes sp_AllNightLog is making calls to Ola's DatabaseBackup, as part of this some of the options in DatabaseBackup are not available to be set from sp_AllNightLog, I made a small change to sp_AllNightLog that allows me to pass the CleanUpTime from sp_AllNightLog into DatabaseBackup essentially setting a retention policy on my log backups. 

The change looks like this...

TODO INSERT CHANGE HERE

One thing to note here is that at the time over writing sp_AllNightLog requires some changes to Ola's scripts for the ChangeBackupType switch to work so if you want to use that feature you need to update Ola's scripts on your server with the ones here https://github.com/BrentOzarULTD/sql-server-maintenance-solution

Setup Step By Step
If you want new databases to be uatomatically backed up then you need to enable xp_cmdshell on your SQL Server so sp_AllNightLog can monitor directories...

{% highlight sql %}
EXEC sp_configure 'show advanced options',1
GO
RECONFIGURE 
GO

EXEC sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO
{% endhighlight %}