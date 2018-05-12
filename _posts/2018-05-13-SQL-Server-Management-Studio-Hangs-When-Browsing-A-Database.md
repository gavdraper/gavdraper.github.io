---
layout: post
title: SQL Server Management Studio Intermittently Hangs Browsing a Database
date: '2018-05-13 20:09:21'
---
If you're seeing SSMS hang/lock timeouts when expanding nodes in the object explorer for a database it's almost definitely caused by schema modification locks. Normally schema modification locks don't cause a problem as they are so short lived but if you're making schema changes inside long running transactions you can make the database un-browsable in the SSMS object explorer.

Below I'm going to go over some different examples that can cause this to happen, to run them first run the following script to setup a test database...

{% highlight sql %}
CREATE DATABASE MySchemaLocks
GO
USE MySchemaLocks
GO
CREATE TABLE Blah(Id INT)
CREATE NONCLUSTERED INDEX ndx_blah ON Blah(id)
{% endhighlight %}

## Dropping Indexes For Data Loading ##

The most common cause I've seen of this issue is dropping indexes at the start of a long running transaction and recreating them after...

{% highlight sql %}
BEGIN TRAN
    DROP INDEX ndx_blah ON Blah
{% endhighlight %}

At this point we have an open transaction that has dropped an index. Imagine it's now running a long query to load new data in keeping the schema modification lock open. If you now try to expand the tables node in SSMS...

![lock timeout error]({{site.url}}/content/images/2018-schema-locks/timeout.PNG)

Until this transaction completes the schema will be locked and SSMS will be unable to browse the database. If you run ROLLBACK then SSMS will be able to browse again. For this situation I'll always recommend that indexes are dropped and recreated outside of the transaction that's loading the data. Where possible always keep schema changes in the shortest-lived transaction possible.

## Creating Staging Tables ##
Creating/dropping staging tables in long transactions does the same as creating/dropping an index...

{% highlight sql %}
BEGIN TRAN
    CREATE TABLE tst (blah INT)
{% endhighlight %}

As before try expanding the tables node in SSMS...

![lock timeout error]({{site.url}}/content/images/2018-schema-locks/timeout.PNG)

Run ROLLBACK and this will work again.

Now for extra points what do you think this will do...

{% highlight sql %}
BEGIN TRAN
    CREATE TABLE #tst (blah INT)
{% endhighlight %}

![tables expanded]({{site.url}}/content/images/2018-schema-locks/tables.PNG)

Yep that works because the schema modifications were not made in our database, they were instead made in TempDB. Now try expanding TempDB's tables node in object explorer...

![tempdb lock timeout error]({{site.url}}/content/images/2018-schema-locks/tempdb-error.PNG)

So temp tables will place their schema modification locks on TempDB, I've never found any reason to be browsing TempDB in the SSMS object explorer so temporary objects in long running queries have never really caused me issues in SSMS.

## Other Causes ##
The above 2 scenarios are the most common causes I've seen as they are both commonly run in long running data load jobs however pretty much any schema modification can cause this e.g ALTER TABLE, ALTER INDEX, CREATE INDEX....

This post is centered around SSMS failing but schema modification locks cause all sorts of problems with user queries too. Most queries you'll run take schema stability locks which will block and wait on schema modification locks, this is another massive reason you want to keep your schema modification locks open for the shorted amount of time possible.