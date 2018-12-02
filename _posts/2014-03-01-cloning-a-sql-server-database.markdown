---
layout: post
title: Cloning a SQL Server Database
date: '2014-03-01 07:35:11'
---

The following script will backup and restore a database to the same server with a different name. Tested on SQL 2012/2008. You just need to set the 4 variables at the top of the script to specify source and destination along with directories to store the backup and restored database files.

```language-sql

USE master;
GO

/*USER INPUT*******************/
DECLARE @SourceDb varchar(200) = 'SourceDatabase'
DECLARE @DestinationDb VARCHAR(200) = 'DestinationDatabase'
DECLARE @BackupDirectory = 'C:\SQLBackups\'
DECLARE @UserDbDirectory = 'C:\UserDbs\'
/******************************/

DECLARE @LogicalFileName VARCHAR(200) = (SELECT name FROM sys.master_files WHERE database_id = DB_ID(@SourceDb)  AND type <> 1)
DECLARE @LogicalLogFileName VARCHAR(200) = (SELECT name FROM sys.master_files WHERE database_id = DB_ID(@SourceDb)  AND type = 1) 
DECLARE @BackupFile VARCHAR(200) = @BackupDirectory + @SourceDb + '.dat'            

DECLARE @Query NVARCHAR(1000)
SET @query = 'BACKUP DATABASE ' + @SourceDb + ' TO DISK = ' + QUOTENAME(@BackupFile,'''')
EXEC (@query)

SET @query = 'RESTORE DATABASE ' + @UserDbDirectory + ' FROM DISK = ' + QUOTENAME(@BackupFile,'''') 
SET @query = @query + ' WITH MOVE ' + QUOTENAME(@LogicalFileName,'''') + ' TO ' + QUOTENAME(@UserDbDirectory + @DestinationDb + '.mdf' ,'''')
SET @query = @query + ' , MOVE ' + QUOTENAME(@LogicalLogFileName,'''') + ' TO ' + QUOTENAME(@UserDbDirectory + @DestinationDb + '_log.ldf','''')
EXEC (@query)
```