---
layout: post
title: SQL Server Find Unused Data Files
date: '2018-12-06 02:55:23'
---
You know that old SQL Server you've left running the last 5 years and had numerous databases dropped and restored to? Have any databases been detached/restores failed part way through and data files just been left behind unused?

Depending on how many databases you have it can be a bit of a pain to go through the files in each one and compare that to the files in your data/log directories. I recently wrote a bit of a hacky script to do just this and list any MDF/LDF files that are not attached to a database on a given instance. It is a bit of a hack and requires you have xp_cmdshell enabled which I'll leave you to read about separately and I would definitely not advise enabling it just for this purpose. But if you have it enabled the script may serve some use to you. If you have chosen the enable xp_cmdshell then the following code will turn it on...

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

To get a list of unused data files set the path in the below script to be the location of your data/log or a directory above them (It searches subdirectories) 

{% highlight sql %}
DROP TABLE #sqlfiles
DECLARE @Path NVARCHAR(200) = 'D:&#92;Databases&#92;'
DECLARE @CmdFileList VARCHAR(150) = 
    'cmd  /c ' + /* Run x_cmdshell */
    '"cd ' + @path + ' && ' +  /* change directory */
    'dir /b /s *.mdf, *.ldf ' /* List filenames only (/b) and subdirs (/s)  Lfd and Mdf*/

CREATE TABLE #SqlFiles ([filename] VARCHAR(1024))
INSERT INTO #SqlFiles 
   EXEC xp_cmdshell @CmdFileList

DELETE FROM #SqlFiles WHERE [Filename] IS NULL OR [Filename] = 'File Not Found'
SELECT [filename] FROM #SqlFiles
EXCEPT
SELECT physical_name FROM sys.master_files
{% endhighlight %}

Any filenames returned from the above are not attached to any databases on the instance you are connected to.