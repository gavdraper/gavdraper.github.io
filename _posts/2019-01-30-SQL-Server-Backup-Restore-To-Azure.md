---
layout: post
title: Backing Up and Restoring Your On-Premise Databases To and From the Cloud
date: '2019-01-30 05:21:44'
tags: cloud azure backup-and-restore
featured_image: '/images/hero-images/typewriter.jpg'
---
I've been meaning to start a series of posts on "Dipping your toes into the cloud" for a while now, there are a number of things you can do to slowly take advantage of the cloud without having to re-architect your whole on-premise setup. This post will serve as part one of that series.

One of the easiest ways to start leveraging the "cloud" with minimal changes is to start moving your backups to your provider of choice. In this post I'm going to use Azure as SQL Server has built in support for it. 

First up we need to log in to the Azure Portal and create a new storage account, the portal UI changes frequently so I'll avoid too many screenshots. To add a storage account...

1.   Hit the "Create a Resource" button and search for storage account
2.   Give your account a name
3.   Choose a data centre location
4.   For account type choose one of the general purpose ones (Blob storage option will not work for what we're doing)
5.   Pick a replication strategy
6.   Pick an access tier, I'll normally use Cold Storage as they won't be frequently accessed

![New Storage Account Wizard]({{site.url}}/content/images/2019-Azure-SQL-Backup\new-storage.PNG)

Next up let's get the keys required to access this account...

1. Open the newly created storage account and navigate to the Access Keys menu item

   ![Access Keys]({{site.url}}/content/images/2019-Azure-SQL-Backup\access-keys.PNG)

2. Copy the value in Key 1 

We now need to create a credential in our on-premise SQL Server to access this...

{% highlight sql %}
CREATE CREDENTIAL [dbbackupstorescredential] 
   WITH IDENTITY = N'STORAGE_ACCOUNT_NAME_GOES_HERE', 
   SECRET = N'KEY_GOES_HERE'
{% endhighlight %}

We then need to configure our blob container

1.   Go back to your new storage account in the Azure Portal and click on Blobs in the menu

![Blobs Menu]({{site.url}}/content/images/2019-Azure-SQL-Backup\blobs-menu.PNG)

2.   Click the new container button, give it a name and click OK
3.   Click the newly created container
4.   Click properties in the menu
5.   Take a copy of the URL

We now have all we need to backup a database to the new blob storage container, The following TSQL will create a new backup in the blob storage container we just created...

{% highlight sql %}
BACKUP DATABASE MyDb 
   TO  URL = N'https://dbbackupstores.blob.core.windows.net/dbcontainer/MyBackup.bak' 
   WITH  CREDENTIAL = N'dbbackupstorescredential'
{% endhighlight %}

![Backup Complete]({{site.url}}/content/images/2019-Azure-SQL-Backup\backup-complete.PNG)

If you then go to back to the Azure Portal and go in to the "Storage Explorer" under the storage resource you can browse to Blob Containers and into your new container where you'll see the backup you just took...

![Storage Explorer]({{site.url}}/content/images/2019-Azure-SQL-Backup\storage-explorer.PNG)

Finally, let's restore our backup from Azure to a new on-premise database...

{% highlight sql %}
RESTORE DATABASE MyRestoredDatabase 
FROM  URL = N'https://dbbackupstores.blob.core.windows.net/dbcontainer/MyBackup.bak'
WITH  CREDENTIAL = N'dbbackupstorescredential'
   ,MOVE N'MyDb' TO N'C:\temp\MyRestoredDb.mdf'
   ,MOVE N'MyDb_Log' TO N'C:\temp\MyRestoredDb.ldf'
{% endhighlight %}

![Restore Complete]({{site.url}}/content/images/2019-Azure-SQL-Backup\restore-complete.PNG)