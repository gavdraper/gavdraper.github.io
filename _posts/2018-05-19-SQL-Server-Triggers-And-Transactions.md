---
layout: post
title: SQL Server Triggers and Transactions
date: '2018-05-19 14:32:14'
tags: triggers transactions
---
In this post I'm going to detail with examples how triggers behave with transactions...

## How Do Triggers Behave With Transactions? ##
To demo this let's insert a record into a table that has an after insert trigger from a transaction and call rollback from the trigger. If you've read my [nested transactions post](https://gavindraper.com/2018/04/25/SQL-Server-Nested-Transactions/) you'll know that any ROLLBACK nested or not will end all transactions and that nested transactions are really a lie.

{% highlight sql %}
CREATE TABLE TriggerTest
(
   Blah NVARCHAR(20)
)
GO

CREATE TRIGGER TestInsertTrigger ON TriggerTest AFTER INSERT
AS
ROLLBACK
GO
{% endhighlight %}

If we then run an insert into the table from a transaction and then call commit on the transaction will it be saved or will the trigger roll it back?

{% highlight sql %}
BEGIN TRAN
INSERT INTO TriggerTest VALUES('test')
COMMIT
{% endhighlight %}

![SSMS Trigger Rollback Error]({{site.url}}/content/images/2018-triggers/error.PNG)

If we check our table you'll see there were no results inserted....

{% highlight sql %}
SELECT * FROM TriggerTest
{% endhighlight %} 

![Empty Results]({{site.url}}/content/images/2018-triggers/no-results.PNG)

So in short a trigger executes in the context of the calling transaction and  a rollback in a trigger will rollback the calling transaction.

## Do Errors in Triggers Prevent Data Being Saved? ##

What if instead of the ROLLBACK in our trigger we have a bug that causes an error...

{% highlight sql %}
ALTER TRIGGER TestInsertTrigger ON TriggerTest AFTER INSERT
AS
SELECT 1/0
GO
{% endhighlight %}

If we then run our insert again will it save?

{% highlight sql %}
BEGIN TRAN
INSERT INTO TriggerTest VALUES('test')
COMMIT
{% endhighlight %}

![SSMS Trigger Dive By Zero Error]({{site.url}}/content/images/2018-triggers/error2.PNG)

![Empty Results]({{site.url}}/content/images/2018-triggers/no-results.PNG)

Nope again we didn't make it to our commit and the transaction has been automatically rolled back.

How about if our insert wasn't in an explicit transaction?

{% highlight sql %}
INSERT INTO TriggerTest VALUES('test')
{% endhighlight %}

![Empty Results]({{site.url}}/content/images/2018-triggers/no-results.PNG)

Same error and still no records in TriggerTest table.

In short any rollback or run time error in a trigger will prevent the underlying data from being saved.

## Do Triggers Fire at The End Of Transactions Or As Changes Are Made? ##
Given a transaction with multiple statements does the trigger fire as each statement is called or only when the transaction ends? Let's check...

{% highlight sql %}
ALTER TRIGGER TestInsertTrigger ON TriggerTest AFTER INSERT
AS
PRINT 'IN TRIGGER'
GO
{% endhighlight %}

{% highlight sql %}
BEGIN TRAN
PRINT 'Before Inset 1'
INSERT INTO TriggerTest VALUES('One')
PRINT 'Before Insert 2'
INSERT INTO TriggerTest VALUES('Two')
COMMIT
{% endhighlight %}

![Trigger Order Output]({{site.url}}/content/images/2018-triggers/order.PNG)

So, triggers fire on each statement as they are run inside the transaction.