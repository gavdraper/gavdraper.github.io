---
layout: post
title: SQL Server Nested Transactions Probably Don't Work How You Think
date: '2018-04-25 21:03:23'
tags: transactions
---
SQL Server allows you to nest multiple transactions but the results of doing so are completely not obvious.

Take the following example...

{% highlight sql %}
BEGIN TRAN
    INSERT INTO MyTable(Blah) VALUES('Blah')
    BEGIN TRAN
        INSERT INTO MyTable(Blah) VALUES('Blah2')
    ROLLBACK
COMMIT
{% endhighlight %}

In this scenario what records do you think got inserted into MyTable? The syntax implies the second insert was rolled back and the first was committed, this however is not what happens. The rollback on the inner transaction rolls back everything all the way up to the top level transaction causing in this example no new records to be inserted. The above query will actually show an error because when it gets to the commit there is no active transaction as it's already been rolled back.

How about this query...

{% highlight sql %}
BEGIN TRAN
    INSERT INTO MyTable(Blah) VALUES('Blah')
    BEGIN TRAN
        INSERT INTO MyTable(Blah) VALUES('Blah2')
    COMMIT
ROLLBACK
{% endhighlight %}

Again at first glance you would probably assume the inner transaction gets committed and the outer one gets rolled back. However this also inserts zero new records, any inner commits are ignored when there are outer transactions and the final rollback will rollback everything to the top level.

What about named transactions?

{% highlight sql %}
BEGIN TRAN Tran1
    INSERT INTO MyTable(Blah) VALUES('Blah')
    BEGIN TRAN Tran2
        INSERT INTO MyTable(Blah) VALUES('Blah2')
    COMMIT TRAN Tran2
ROLLBACK TRAN Tran1
{% endhighlight %}

Again nothing gets inserted because even though you've named what transaction to commit the inner commit is ignored as there is an outer transaction.

With this in mind there is little point to ever nesting transactions and doing so can actually cause damage in making it easy to think something is happening that isn't.