---
layout: post
title: SQL Server Finding Unused Data Files
date: '2018-12-04 09:34:01'
---
You know that old SQL Server you've had running the last 5 years and had numerous databases dropped and restored to? Have any databases been detached/restores failed part way through and data files just been left behind unused?

Depending on how many databases you have it can be a bit of a pain to go through the files in each one and compare that to the files in your data/log directories. I recently wrote a bit of a hacky script to do just this and list any MDF/LDF files that are not attached to a database on a given instance. It is a bit of a hack and requires you have xp_cmdshell enabled which I'll leave you to read about separately and I would definitely not advise enabling it just for this purpose. But if you have it enabled the script may serve some use to you. If you have chosen the enable is then the following script will do that for you...

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

To use it just set the path to be above the path your data/log files are in and it will search all subdirectories for any files that are not part of a database...

{% highlight sql %}
DECLARE @Path NVARCHAR(200) = 'C:\Program Files\Microsoft SQL Server\MSSQL15.SQL2019'
DECLARE @CmdFileList VARCHAR(150) = 
	'cmd  /c ' + /* Run x_cmdshell */
	'"cd ' + @path + ' && ' +  /* change directory */
	'dir /b /s '+ /* List filenames only (/b) and subdirs (/s) */
	'| findstr /c:".mdf" /c:".ldf""' /* Match mdf/ldf in name */
CREATE TABLE #SqlFiles ([filename] VARCHAR(1024))
INSERT INTO #SqlFiles 
   EXEC xp_cmdshell @CmdFileList
DELETE FROM #SqlFiles WHERE [Filename] IS NULL
SELECT [filename] FROM #SqlFiles
EXCEPT
SELECT physical_name FROM sys.master_files
{% endhighlight %}